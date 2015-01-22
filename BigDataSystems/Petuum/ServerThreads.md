# ServerThreads

## 基本结构
1. 每个client上的app进程持有一个ServerThreads object，这个object管理该client上的所有server threads。这些server threads的启动过程：`app.main() => PSTableGroup::Init() => ServerThreads::Init() => ServerThreadMain(threadId) for each server thread`。
2. 每个server thread实际上是一个Server object。ServerThreads对象通过`vector<pthread_t> threads`和`vector<int> threads_ids`来引用server threads，通过其ServerContex指针用来访问每个server thread对应的Server object（`server_context_ptr->server_obj`）。
3. 对于每一个server thread，都持有一个ServerContext，其初始化时`server_context.bg_threads_ids`存储PS中所有bg threads的`bg_thread_id`，`server_context.server_obj`存储该server thread对应的Server object。
4. 每个Server object里面存放了三个数据结构：`client_bg_map<client_id, bg_id>`存放PS中有那些client，每个client上有那些bg threads；`client_ids`存放PS中有那些client；`client_clocks`是VectorClock，存放来自client的clock，初始化时clock为0。每个Server thread在初始化时会去connect PS中所有的bg thread，然后将`(client_id, 0)`添加到server thread对应的Server object中的`client_clocks`中。如果某个client上有多个bg thread，那么`(client_id, 0)`会被重复添加到`client_clocks: VectorClock`中，会做替换。注意`client_clocks: VectorClock`的长度为PS中client的总个数，也就是每一个client对应一个clock，而不是每个bg thread对应一个clock。Server object还有一个`client_vector_clock_map<int, VectorClock>`的数据结构，key为`client_id`，value为该client上所有bg thread的VectorClock。也就是说每个server thread不仅存放了每个client的clock，也存放了该client上每个bg thread的clock。
5. Server object还有一个`bg_version_map<bg_thread_id, int>`的数据结构，该结构用于存放server thread收到的bg thread的最新oplog版本。

## CreateTable

Server thread启动后，会不断循环等待消息，当收到Namenode发来的`create_table_msg`时，会调用`HandleCreateTable(create_table_msg)`来createTable，会经历以下步骤：

1. 从msg中提取出tableId。
2. 回复消息给Namenode说准备创建table。
3. 初始化TableInfo消息，包括table的`staleness, row_type, columnNum (row_capacity)`。
4. 然后调用server thread对应的Server object创建table，使用`Server.CreateTable(table_id, table_info)`。
5. Server object里面有个`map<table_id, ServerTable> tables`数据结构，`CreateTable(table_id)`就是new出一个ServerTable，然后将其加入这个map。
6. ServerTable object会存放`table_info`，并且有一个`map<row_id, ServerRow> storage`，这个map用来存放ServerTable中的rows。另外还有一个`tmp_row_buff[row_length]`的buffer。new ServerTable时，只是初始化一些这些数据结构。

## HandleClientSendOpLogMsg

当某个server thread收到client里bg thread发来的`client_send_oplog_msg`时，会调用ServerThreads的`HandleOpLogMsg(client_send_oplog_msg)`，该函数会执行如下步骤：

1. 从msg中抽取出`client_id`，判断该msg是否是clock信息，并提取出oplog的version。
2. 调用server thread对应的`ServerObj.ApplyOpLog(client_send_oplog_msg)`。该函数会将oplog中的updates requests都更新到本server thread维护的ServerTable。
3. 如果msg中没有携带clock信息，那么执行结束，否则继续下面的步骤：
4. 调用`ServerObj.Clock(client_id, bg_id)`，并返回`bool clock_changed`。该函数会更新client的VectorClock（也就是每个bg thread的clock），如果client的VectorClock中唯一最小的clock被更新，那么client本身的clock也需要更新，这种情况下`clock_changed`为true。
5. 如果`clock_changed == false`，那么结束，否则，进行下面的步骤：
6. `vector<ServerRowRequest> requests = serverObject.GetFulfilledRowRequests()`。
7. 对每一个request，提取其`table_id, row_id, bg_id`，然后算出bg thread的`version = serverObj.GetBgVersion(bg_id)`。
8. 根据提取的`row_id`去Server object的ServerTable中提取对应的row，使用方法`ServerRow server_row = ServerObj.FindCreateRow(table_id, row_id)`。
9. 调用`RowSubscribe(server_row, bg_id_to_client_id)`。如果consistency model是SSP，那么RowSubscribe就是SSPRowSubscribe；如果是SSP push，那么RowSubscribe就是SSPPushRowSubscribe。NMF使用是后者，因此这一步就是`SSPPushRowSubscribe(server_row, bg_id_to_client_id)`。该方法的意思是将`client_id`注册到该`server_row`，这样将该`server_row`在调用`AppendRowToBuffs`可以使用`callback_subs.AppendRowToBuffs()`。
10. 查看Server object中VectorClock中的最小clock，使用方法`server_clock = ServerObj.GetMinClock()`。
11. `ReplyRowRequest(bg_id, server_row, table_id, row_id, sersver_clock)`。
12. 最后调用`ServerPushRow()`。

### `Server.ApplyOpLog(oplog, bg_thread_id, version)`

1. check一下，确保自己`bg_version_map`中该bg thread对应的version比这个新来的version小1。
2. 更新`bg_version_map[bg_thread_id] = version`。
3. oplog里面可以存在多个update request，对于每一个update request，执行以下步骤：
4. 读取oplog中的`table_id, row_id, column_ids, num_updates, started_new_table`到updates。
5. 根据`table_id`从`ServerObj.tables`中找出对应的ServerTable。
6. 执行ServerTable的`ApplyRowOpLog(row_id, column_ids, updates, num_updates)`。该方法会找出ServerTable对应的row，并对row进行`BatchInc(column_ids, updates)`。如果ServerTable不存在该row，就先`CreateRow(row_id)`，然后`BatchInc()`。
7. 打出"Read and Apply Update Done"的日志。

### `ServerObj.Clock(client_id, bg_id)`

1. 执行`ServerObj.client_vector_clock_map[client_id].Tick(bg_id)`，该函数将client对应的VectorClock中`bg_id`对应的clock加1。
2. 如果`bg_id`对应的原始clock是VectorClock中最小值，且是唯一的最小值，那么clock+1后，需要更新client对应的clock，也就是对`client_clocks.Tick(client_id)`。
3. 然后看是否达到了snapshot的clock，达到就进行checkpoint。

## HandleRowRequestMsg

当某个server thread收到client里bg thread发来的`row_request_msg`时，会调用ServerThreads的`HandleRowRequest(bg_id, row_request_msg)`，该函数会执行如下步骤：

1. 从msg中提取出`table_id, row_id, clock`。
2. 查看ServerObj中的所有client的最小clock。使用`server_clock = ServerObj.GetMinClock()`。
3. 如果msg请求信息中的clock > `server_clock`，也就是说目前有些clients在clock时的更新信息还没有收到，那么先将这个msg的request存起来，等到ServerTable更新到clock时，再reply。具体会执行`ServerObj.AddRowRequest(sender_id, table_id, row_id, clock)`。
4. 如果msg请求信息中的clock  <= `server_clock`，也就是说ServerTable中存在满足clock要求的rows，那么会执行如下步骤：
5. 得到`bg_id`的version，使用`version = ServerObj.GetBgVersion(sender_id)`，`sender_id`就是发送`row_request_msg`请求的client上面的bg thread。
6. 将ServerTable中被request的row取出来到`server_row`。
7. 调用`RowSubscribe(server_row, sender_id_to_thread_id)`。
8. 将`server_row`reply给bg thread，具体使用`ReplyRowRequest(sender_id, server_row, table_id, row_id, server_clock, version)`。



### `ServerObj.AddRowRequest(sender_id, table_id, row_id, clock)`

当来自client的request当前无法被处理的时候（server的row太old），server会调用这个函数将请求先放到队列里。具体执行如下步骤：

1. 先new一个ServerRowRequest的结构体，将`bg_id, table_id, row_id, clock`放到这个结构体中。
2. 将ServerRowRequest放进`map<clock, vector<ServerRowRequest>> clock_bg_row_requests`中，该数据结构的key是clock，vector中的index是`bg_id`，value是ServerRowRequest。

### `ReplyRowRequest(sender_id, server_row, table_id, row_id, server_clock, version)`

1. 先构造一个`ServerRowRequestReplyMsg`，然后将`table_id, row_id, server_clock, version`填入这个msg中。
2. 然后将msg序列化后发回给`bg_id`对应的bg thread。

