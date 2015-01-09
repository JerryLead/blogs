# TableUpdate

```c++
void SSPConsistencyController::BatchInc(int32_t row_id,
  const int32_t* column_ids, const void* updates, int32_t num_updates) {

  // thread_cache_是ThreadTable的指针，ThreadTable就是ClientTable或者ServerTable
  // IndexUpadte(row_id)会
  thread_cache_->IndexUpdate(row_id);

  OpLogAccessor oplog_accessor;
  oplog_.FindInsertOpLog(row_id, &oplog_accessor);

  const uint8_t* deltas_uint8 = reinterpret_cast<const uint8_t*>(updates);

  for (int i = 0; i < num_updates; ++i) {
    void *oplog_delta = oplog_accessor.FindCreate(column_ids[i]);
    sample_row_->AddUpdates(column_ids[i], oplog_delta, deltas_uint8
			    + sample_row_->get_update_size()*i);
  }

  RowAccessor row_accessor;
  bool found = process_storage_.Find(row_id, &row_accessor);
  if (found) {
    row_accessor.GetRowData()->ApplyBatchInc(column_ids, updates,
                                             num_updates);
  }
}
```


## ClientTable属性解释

```c++
Class ClientTable {
	private:
	  // table Id
	  int32_t table_id_;
	  // Table里面row的类型，比如DenseRow<float>
	  int32_t row_type_;
	  // Row的游标（指针）
	  const AbstractRow* const sample_row_;
	  // Table的更新日志
	  TableOpLog oplog_;
	  // 进程里cache的Table
	  ProcessStorage process_storage_;
	  // Table的一致性控制协议
	  AbstractConsistencyController *consistency_controller_;

	  // ThreadTable就是ClientTable或者ServerTable
	  // thread_cahce就是Threads维护的ClientTable的全局对象
	  boost::thread_specific_ptr<ThreadTable> thread_cache_;
	  // 操作日志，每个bg thread对应一个index value
	  TableOpLogIndex oplog_index_;
}
```