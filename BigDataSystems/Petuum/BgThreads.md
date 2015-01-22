# BgWorkers

BgWorkers的角色与ServerThreads的角色类似，都是管理本进程里的bg/server threads。BgWorker通过BgContext来管理，ServerThreads通过ServerContext来管理。

BgContext里面存放了以下数据结构：

```c++
int version;   // version of the data, increment when a set of OpLogs
               // are sent out; may wrap around
               // More specifically, version denotes the version of the
               // OpLogs that haven't been sent out.
               // version表示client端的最新opLog还没有发送给server
RowRequestOpLogMgr *row_request_oplog_mgr;

// initialized by BgThreadMain(), used in CreateSendOpLogs()
// For server x, table y, the size of serialized OpLog is ...
map<serverId, map<int32_t, size_t> > server_table_oplog_size_map;
// The OpLog msg to each server
map<serverId,, ClientSendOpLogMsg* > server_oplog_msg_map;
// map server id to oplog msg size
map<serverId,, size_t> server_oplog_msg_size_map;
// size of oplog per table, reused across multiple tables
map<int32_t, size_t> table_server_oplog_size_map;

/* Data members needed for server push */
VectorClock server_vector_clock;
```

## Bg thread初始化

1. 在bg thread初始化时会先打印出来“Bg Worker starts here, my id = 100/1100”。
2. InitBgContext()。设置一下`bg_context->row_request_oplog_mgr = new SSPPushRowRequestOpLogMgr`。然后对PS中的每一个`serverId`，将其放入下列数据结构，`server_table_oplog_size_map.insert(serverId, map<int, int>())`，`server_oplog_msg_map.insert(serverId, 0)`，`server_oplog_msg_size_map.insert(serverId, 0)`，`table_server_oplog_size_map.insert(serverId, 0)`，`server_vector_clock.AddClock(serverId)`。AddClock会将`serverId, clock=0`放入到`server_vector_clock`中。
3. BgServerHanshake()。

	```
	1. 通过ConnectToNameNodeOrServer(name_node_id)连接Namenode。
	      首先打印出"ConnectToNameNodeOrServer server_id"。
	      然后将自己的client_id填入到ClientConnectMsg中。
	      最后将msg发送给server_id对应的local/remote server thread（这里是Namenode thread）。
	2. 等待Namenode返回的ConnectServerMsg (kConnectServer)消息。
	3. 连接PS里面的每个server thread，仍然是通过ConnectToNameNodeOrServer(server_id)。
	4. 等待，直到收到所有server thread返回的kClientStart信息，每收到一条信息就会打印"get kClientStart from server_id"。
	5. 收到namenode和所有server返回的信息后，退出。
	```
4. 解除`pthread_barrier_wait`。
5. 去接受本进程内的AppInitThread的连接。使用`RecvAppInitThreadConnection()`去接受连接，连接消息类型是kAppConnect。
6. 如果本bg thread是head bg thread（第一个bg thread）就要承担CreateClientTable的任务，先打印"head bg handles CreateTable"，然后调用HandleCreateTables()，然后wait直到Table创建完成。
7. 最后便进入了无限等待循环，等待接受msg，处理msg。

### HandleCreateTables()

> the app thread shall not submit another create table request before the current one returns as it is blocked waiting

1. 假设要create 3 tables，那么会去`comm_bus`索取这每个table的BgCreateTableMsg (kBgCreateTable)，然后从msg中提取`staleness, row_type, row_capacity, process_cache_capacity, thread_cache_capacity, oplog_capacity`。
2. 将`table_id, staleness, row_type, row_capacity`包装成`CreateTableMsg`，然后将该msg发送到Namenode。
3. 等待接收Namenode的反馈信息CreateTableReplyMsg (kCreateTableReply)，收到就说明namenode已经知道head bg thread要创建ClientTable。
4. 然后可以创建`client_table = new ClientTable(table_id, client_table_config)`。
5. 将`client_table`放进`map<table_id, ClientTable> tables`里。
6. 打印"Reply app thread"，回复app init thread表示ClientTable已经创建。

### `ClientTable(table_id, client_table_config)`

与ServerTable直接存储parameter rows不同，ClientTable是一个逻辑概念，它相当于一个ServerTable的buffer/cache，app thread将最新的参数先写入到这个buffer，然后push到Server上。从Server端pull parameter rows的时候也一样，先pull到ClientTable里面然后读到app thread里面。

![](figures/ClientTableUpdate.png)

1. 提取`table_id, row_type`。
2. 创建一个`Row sample_row`，创建这个row只是用来使用Row中的函数，而不是ClientTable中实际存储value的row，实际的row存放在`process_storage`中。
3. 初始化一下oplog，oplog用于存储parameter的本地更新，也就是实际的updated value。有几个bg thread，就有几个oplog.opLogPartition。
4. 初始化`process_storage(config.process_cache_capacity)`。`process_storage`被所有thread共享，里面存储了ClientTable的实际rows，但由于`process_storage`有大小限制（row的个数），可能存储ClientTable的一部分，完整的Table存放在Server端。
5. 初始化`oplog_index`，目前还不知道这个东西是干嘛的？
6. 设置Table的一致性控制器，如果是SSP协议就使用SSPConsistencyController，如果是SSPPush协议，使用SSPPushConsistencyController。

## 当bg thread收到kAppConnect消息

1. `++num_connected_app_threads`

## 当bg thread收到kRowRequest消息

1. 接收到`row_request_msg`，类型是RowRequestMsg。
2. 调用`CheckForwardRowRequestToServer(sender_id, row_request_msg)`来处理rowRequest消息，`sender_id`就是app thread id。

### `CheckForwardRowRequestToServer(app_thread_id, row_request_msg)`

1. 从msg中提取出`table_id, row_id, clock`。
2. 从tables中找到`table_id`对应的ClientTable table。
3. 提取出table对应的ProcessStorage，并去该storage中查找`row_id`对应的row。
4. 如果找到了对应的row，且row的clock满足要求（row.clock >= request.clock），那么只是发一个空RowRequestReplyMsg消息给app thread，然后return。如果没找到对应的row，那就要去server端取，会执行下面的步骤：
5. 构造一个RowRequestInfo，初始化它的`app_thread_id, clock = row_request_msg.clock, version = bgThread.version - 1`。Version in request denotes the update version that the row on server can see. Which should be 1 less than the current version number。
6. 将这个RowRequestInfo加入到RowRequestOpLogMgr中，使用`bgThread.row_request_oplog_mgr->AddRowRequest(row_request, table_id, row_id)`。
7. 如果必须send这个RowRequestInfo（本地最新更新也没有）到server，就会先根据`row_id`计算存储该`row_id`的`server_id`（通过GetRowPartitionServerID(table_id, row_id)，只是简单地`server_ids[row_id % num_server]`），然后发`row_request_msg`请求给server。

### `SSPRowRequestOpLogMgr.AddRowRequest(row_request, table_id, row_id)`

1. 提取出request的version (也就是bgThread.version - 1)。
2. request.sent = true。
3. 去`map<(tableId, rowId), list<RowRequestInfo> > bgThread.row_request_oplog_mgr.pending_row_requests`里取出`(request.table_id, request.row_id)`对应的list<RowRequestInfo>，然后从后往前查看，将request插入到合适的位置，使得prev.clock < request.clock < next.clock。如果插入成功，那么会打印"I'm requesting clock is request.clock. There's a previous request requesting clock is prev.clock."。然后将request.sent设置为false（意思是不用send request到server端，先暂时保存），`request_added`设置为true。
4. `++version_request_cnt_map[version]`。


> 可见在client和server端之间不仅要cache push/pull的parameters，还要cache push/pull的requests。