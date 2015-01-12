# Important Classes

## ClientTable

```c++
class ClientTable : public AbstractClientTable {
public:
  // Instantiate AbstractRow, TableOpLog, and ProcessStorage using config.
  ClientTable(int32_t table_id, const ClientTableConfig& config);

  ~ClientTable();

  void RegisterThread();

  void GetAsync(int32_t row_id);
  void WaitPendingAsyncGet();
  void ThreadGet(int32_t row_id, ThreadRowAccessor *row_accessor);
  void ThreadInc(int32_t row_id, int32_t column_id, const void *update);
  void ThreadBatchInc(int32_t row_id, const int32_t* column_ids,
                      const void* updates,
                      int32_t num_updates);
  void FlushThreadCache();

  void Get(int32_t row_id, RowAccessor *row_accessor);
  void Inc(int32_t row_id, int32_t column_id, const void *update);
  void BatchInc(int32_t row_id, const int32_t* column_ids, const void* updates,
    int32_t num_updates);

  void Clock();
  cuckoohash_map<int32_t, bool> *GetAndResetOpLogIndex(int32_t client_table);

  ProcessStorage& get_process_storage () {
    return process_storage_;
  }

  TableOpLog& get_oplog () {
    return oplog_;
  }

  const AbstractRow* get_sample_row () const {
    return sample_row_;
  }

  int32_t get_row_type () const {
    return row_type_;
  }

private:
  int32_t table_id_;
  int32_t row_type_;
  // 指向每一个row的指针
  const AbstractRow* const sample_row_;
  // Table操作日志
  TableOpLog oplog_;
  // 进程的Table
  ProcessStorage process_storage_;
  // Table的一致性controller
  AbstractConsistencyController *consistency_controller_;

  // ThreadTable指针
  boost::thread_specific_ptr<ThreadTable> thread_cache_;
  // Table操作日志的index
  TableOpLogIndex oplog_index_;
};

}  // namespace petuum
```

## ThreadTable

```c++
class ThreadTable : boost::noncopyable {
public:
  explicit ThreadTable(const AbstractRow *sample_row);
  ~ThreadTable();
  void IndexUpdate(int32_t row_id);
  void FlushOpLogIndex(TableOpLogIndex &oplog_index);

  AbstractRow *GetRow(int32_t row_id);
  void InsertRow(int32_t row_id, const AbstractRow *to_insert);
  void Inc(int32_t row_id, int32_t column_id, const void *delta);
  void BatchInc(int32_t row_id, const int32_t *column_ids,
    const void *deltas, int32_t num_updates);

  void FlushCache(ProcessStorage &process_storage, TableOpLog &table_oplog,
		  const AbstractRow *sample_row);

private:
  // Vector[set, set, set, ..., set]
  std::vector<std::unordered_set<int32_t> > oplog_index_;
  // HashMap<rowid, Row>
  boost::unordered_map<int32_t, AbstractRow* > row_storage_;
  // HashMap<rowid RowOpLog>
  boost::unordered_map<int32_t, RowOpLog* > oplog_map_;
  // Row指针
  const AbstractRow *sample_row_;
};
```

## TableGroup
```c++
class TableGroup : public AbstractTableGroup {
public:
  TableGroup(const TableGroupConfig &table_group_config,
             bool table_access, int32_t *init_thread_id);

  ~TableGroup();

  bool CreateTable(int32_t table_id,
      const ClientTableConfig& table_config);

  void CreateTableDone();

  void WaitThreadRegister();

  AbstractClientTable *GetTableOrDie(int32_t table_id) {
    auto iter = tables_.find(table_id);
    CHECK(iter != tables_.end()) << "Table " << table_id << " does not exist";
    return static_cast<AbstractClientTable*>(iter->second);
  }

  int32_t RegisterThread();

  void DeregisterThread();

  void Clock();

  void GlobalBarrier();

private:
  typedef void (TableGroup::*ClockFunc) ();
  ClockFunc ClockInternal;

  void ClockAggressive();
  void ClockConservative();

  // TreeMap<rowid, ClientTable>
  std::map<int32_t, ClientTable* > tables_;
  // barrier
  pthread_barrier_t register_barrier_;
  // 注册的app thread（也就是worker thread）数目
  std::atomic<int> num_app_threads_registered_;

  // Max staleness among all tables.
  int32_t max_table_staleness_;
  // Table处于第几个clock里面
  VectorClockMT vector_clock_;
};
```

## SSPClientRow

```c++
// ClientRow is a wrapper on user-defined ROW data structure (e.g., vector,
// map) with additional features:
//
// 1. Reference Counting: number of references used by application. Note the
// copy in storage itself does not contribute to the count
// 2. Row Metadata
//
// ClientRow does not provide thread-safety in itself. The locks are
// maintained in the storage and in (user-defined) ROW.
class SSPClientRow : public ClientRow {
public:
  // ClientRow takes ownership of row_data.
  SSPClientRow(int32_t clock, AbstractRow* row_data):
      ClientRow(clock, row_data),
      clock_(clock){ }

  void SetClock(int32_t clock) {
    std::unique_lock<std::mutex> ulock(clock_mtx_);
    clock_ = clock;
  }

  int32_t GetClock() const {
    std::unique_lock<std::mutex> ulock(clock_mtx_);
    return clock_;
  }

  // Take row_data_pptr_ from other and destroy other. Existing ROW will not
  // be accessible any more, but will stay alive until all RowAccessors
  // referencing the ROW are destroyed. Accesses to SwapAndDestroy() and
  // GetRowDataPtr() must be mutually exclusive as they the former modifies
  // row_data_pptr_.
  void SwapAndDestroy(ClientRow* other) {
    clock_ = dynamic_cast<SSPClientRow*>(other)->clock_;
    ClientRow::SwapAndDestroy(other);
  }

private:  // private members
  mutable std::mutex clock_mtx_;
  int32_t clock_;
};
```


## SerializedRowReader

```c++
// Provide sequential access to a byte string that's serialized rows.
// Used to facilicate server reading row data.

// st_separator : serialized_table_separator
// st_end : serialized_table_end

// Tables are serialized as the following memory layout
// 1. int32_t : table id, could be st_separator or st_end
// 2. int32_t : row id, could be st_separator or st_end
// 3. size_t : serialized row size
// 4. row data
// repeat 1, 2, 3, 4
// st_separator can not happen right after st_separator
// st_end can not happen right after st_separator

// Rules for serialization:
// The serialized row data is guaranteed to end when seeing a st_end or with
// finish reading the entire memory buffer.
// When seeing a st_separator, there could be another table or no table
// following. The latter happens only when the buffer reaches its memory
// boundary.

class SerializedRowReader : boost::noncopyable {
public:
  // does not take ownership
  SerializedRowReader(const void *mem, size_t mem_size):
      mem_(reinterpret_cast<const uint8_t*>(mem)),
      mem_size_(mem_size) {
    VLOG(0) << "mem_size_ = " << mem_size_;
  }
  ~SerializedRowReader() { }

  bool Restart() {
    offset_ = 0;
    current_table_id_ = *(reinterpret_cast<const int32_t*>(mem_ + offset_));
    offset_ += sizeof(int32_t);

    if (current_table_id_ == GlobalContext::get_serialized_table_end())
      return false;
    return true;
  }

  const void *Next(int32_t *table_id, int32_t *row_id, size_t *row_size) {
    // When starting, there are 4 possiblilities:
    // 1. finished reading the mem buffer
    // 2. encounter the end of an table but there are other tables following
    // (st_separator)
    // 3. encounter the end of an table but there is no other table following
    // (st_end)
    // 4. normal row data

    if (offset_ + sizeof (int32_t) > mem_size_)
      return NULL;
    *row_id = *(reinterpret_cast<const int32_t*>(mem_ + offset_));
    offset_ += sizeof(int32_t);

    do {
      if (*row_id == GlobalContext::get_serialized_table_separator()) {
        if (offset_ + sizeof (int32_t) > mem_size_)
          return NULL;

        current_table_id_ = *(reinterpret_cast<const int32_t*>(mem_ + offset_));
        offset_ += sizeof(int32_t);

        if (offset_ + sizeof (int32_t) > mem_size_)
          return NULL;

        *row_id = *(reinterpret_cast<const int32_t*>(mem_ + offset_));
        offset_ += sizeof(int32_t);
        // row_id could be
        // 1) st_separator: if the table is empty and there there are other
        // tables following;
        // 2) st_end: if the table is empty and there are no more table
        // following
        continue;
      } else if (*row_id == GlobalContext::get_serialized_table_end()) {
        return NULL;
      } else {
        *table_id = current_table_id_;
        *row_size = *(reinterpret_cast<const size_t*>(mem_ + offset_));
        offset_ += sizeof(size_t);
        const void *data_mem = mem_ + offset_;
        offset_ += *row_size;
        //VLOG(0) << "mem read offset = " << offset_;
        return data_mem;
      }
    }while(1);
  }

private:
  const uint8_t *mem_;
  size_t mem_size_;
  size_t offset_; // bytes to be read next
  int32_t current_table_id_;
};
```

## ProcessStorage

```c++
// ProcessStorage is shared by all threads.
//
// TODO(wdai): Include thread storage in ProcessStorage.
class ProcessStorage {
public:
  // capacity is the upper bound of the number of rows this ProcessStorage
  // can store.
  explicit ProcessStorage(int32_t capacity, size_t lock_pool_size);

  ~ProcessStorage();

  // Find row row_id; row_accessor is a read-only smart pointer. Return true
  // if found, false otherwise. Note that the # of active row_accessor
  // cannot be close to capacity, or Insert() will have undefined behavior
  // as we may not be able to evict any row that's not being referenced by
  // row_accessor.
  bool Find(int32_t row_id, RowAccessor* row_accessor);

  // Check if a row exists, does not count as one access
  bool Find(int32_t row_id);

  // Insert a row, and take ownership of client_row. Return true if row_id
  // does not already exist (possibly evicting another row), false if row
  // row_id already exists and is updated. If hitting capacity, then evict a
  // row using ClockLRU.  Return read reference and evicted row id if
  // row_accessor and evicted_row_id is supplied. We assume
  // row_id is always non-negative, and use *evicted_row_id = -1 if no row
  // is evicted. The evicted row is guaranteed to have 0 reference count
  // (i.e., no application is using).
  //
  // Note: To stay below the capacity, we first check num_rows_. If
  // num_rows_ >= capacity_, we subtract (num_rows_ - capacity_) from
  // num_rows_ and then evict (num_rows_ - capacity_ + 1) rows using
  // ClockLRU before inserting. This could result in over-eviction when two
  // threads simultaneously do this eviction, but this is fine.
  //
  // TODO(wdai): Watch out when over-eviction clears the inactive list.
  bool Insert(int32_t row_id, ClientRow* client_row);
  bool Insert(int32_t row_id, ClientRow* client_row,
      RowAccessor* row_accessor, int32_t* evicted_row_id = 0);

  bool Insert(int32_t row_id, ClientRow* client_row,
              RowAccessor *row_accessor, int32_t *evicted_row_id,
              ClientRow** evicted_row);

private:    // private functions
  // Evict one inactive row using CLOCK replacement algorithm.
  void EvictOneInactiveRow();

  // Find row_id in storage_map_, assuming there is lock on row_id. If
  // found, update it with client_row, reference LRU, and set row_accessor
  // accordingly, and return true. Return false if row_id is not found.
  bool FindAndUpdate(int32_t row_id, ClientRow* client_row);
  bool FindAndUpdate(int32_t row_id, ClientRow* client_row,
      RowAccessor* row_accessor);

private:    // private members
  // Number of rows allowed in this storage.
  int32_t capacity_;

  // Number of rows in the storage. We choose not to use Cuckoo's size()
  // which is more expensive.
  std::atomic<int32_t> num_rows_;

  // Shared map with ClockLRU. The key type is row_id (int32_t), and the
  // value type consists of a ClientRow* pointer (void*) and a slot #
  // (int32_t).
  // HashMap<rowId, ClientRow>
  cuckoohash_map<int32_t, std::pair<void*, int32_t> > storage_map_;

  // Depends on storage_map_, thus need to be initialized after it.
  ClockLRU clock_lru_;

  // Lock pool.
  StripedLock<int32_t> locks_;
};
```

## RowOpLog
```c++
class RowOpLog : boost::noncopyable {
public:
  RowOpLog(uint32_t update_size, InitUpdateFunc InitUpdate):
    update_size_(update_size),
    InitUpdate_(InitUpdate) { }

  ~RowOpLog() {
    auto iter = oplogs_.begin();
    for (; iter != oplogs_.end(); iter++) {
      delete reinterpret_cast<uint8_t*>(iter->second);
    }
  }

  void* Find(int32_t col_id) {
    auto iter = oplogs_.find(col_id);
    if (iter == oplogs_.end()) {
      return 0;
    }
    return iter->second;
  }

  const void* FindConst(int32_t col_id) const {
    auto iter = oplogs_.find(col_id);
    if (iter == oplogs_.end()) {
      return 0;
    }
    return iter->second;
  }

  void* FindCreate(int32_t col_id) {
    auto iter = oplogs_.find(col_id);
    if (iter == oplogs_.end()) {
      void* update = reinterpret_cast<void*>(new uint8_t[update_size_]);
      InitUpdate_(col_id, update);
      oplogs_[col_id] = update;
      return update;
    }
    return iter->second;
  }

  // Guaranteed ordered traversal
  void* BeginIterate(int32_t *column_id) {
    iter_ = oplogs_.begin();
    if (iter_ == oplogs_.end()) {
      return 0;
    }
    *column_id = iter_->first;
    return iter_->second;
  }

  void* Next(int32_t *column_id) {
    iter_++;
    if (iter_ == oplogs_.end()) {
      return 0;
    }
    *column_id = iter_->first;
    return iter_->second;
  }

  // Guaranteed ordered traversal, in ascending order of column_id
  const void* BeginIterateConst(int32_t *column_id) const {
    const_iter_ = oplogs_.cbegin();
    if (const_iter_ == oplogs_.cend()) {
      return 0;
    }
    *column_id = const_iter_->first;
    return const_iter_->second;
  }

  const void* NextConst(int32_t *column_id) const {
    const_iter_++;
    if (const_iter_ == oplogs_.cend()) {
      return 0;
    }
    *column_id = const_iter_->first;
    return const_iter_->second;
  }

  int32_t GetSize() const {
    return oplogs_.size();
  }

private:
  // 
  const uint32_t update_size_;
  // TreeMap<rowId, updateFunc>
  std::map<int32_t, void*> oplogs_;
  // 最初的update函数
  InitUpdateFunc InitUpdate_;

  std::map<int32_t, void*>::iterator iter_;
  mutable std::map<int32_t, void*>::const_iterator const_iter_;
};
```