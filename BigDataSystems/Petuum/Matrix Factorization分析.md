# Matrix Factorization分析

## 1. 初始化
### Configure Petuum PS
```c++
// Configure PS row types
petuum::PSTableGroup::RegisterRow<petuum::DenseRow<float> >(0);  // Register dense 
```
注册Row类型，实际动作是将`Class DenseRow<float>`放到了一个`map<int32_t, CreateFunc> creator_map_`里面，map的key就是`RegisterRow(id)`中的id，这里是0。

### Start PS

```c++
// Start PS
// IMPORTANT: This command starts up the name node service on client 0.
//            We therefore do it ASAP, before other lengthy actions like
//            loading data.
petuum::PSTableGroup::Init(table_group_config, false);  // Initializing thread does not need table access
```
实际动作是new出来一个TableGroup。

Server is different from NameNode. NameNode is not considered as server.

Thread id范围
- 0~100: Server和NameNode thread使用
- 200~1000: app thread使用

Server thread需要设置consistency_model，具体如下：

```c++
  //ServerThreadMainFunc ServerThreadMain;
  ConsistencyModel consistency_model = GlobalContext::get_consistency_model();
  switch(consistency_model) {
    case SSP:
      ServerPushRow = SSPServerPushRow;
      RowSubscribe = SSPRowSubscribe;
      break;
    case SSPPush:
      ServerPushRow = SSPPushServerPushRow;
      RowSubscribe = SSPPushRowSubscribe;
      VLOG(0) << "RowSubscribe = SSPPushRowSubscribe";
      break;
    default:
      LOG(FATAL) << "Unrecognized consistency model " << consistency_model;
  }
  ```
  background tread is used for storing opLog，同样backgroud thread也需要设置consistency_model如下：
  ```c++
  BgThreadMainFunc BgThreadMain;
  ConsistencyModel consistency_model = GlobalContext::get_consistency_model();
  switch(consistency_model) {
    case SSP:
      {
        BgThreadMain = SSPBgThreadMain;
        MyCreateClientRow = CreateSSPClientRow;
        GetRowOpLog = SSPGetRowOpLog;
      }
      break;
    case SSPPush:
      {
        BgThreadMain = SSPBgThreadMain;
        MyCreateClientRow = CreateClientRow;
        system_clock_ = 0;
        GetRowOpLog = SSPGetRowOpLog;
      }
      break;
    default:
      LOG(FATAL) << "Unrecognized consistency model " << consistency_model;
  }
  ```
  
  bg_workers也会添加vector clock.
  
  init thread也会添加vector clock
  
  TableGroupConfig里面还有一个aggressive_clock属性：
  ```c++
  // If set to true, oplog send is triggered on every Clock() call.
  // If set to false, oplog is only sent if the process clock (representing all
  // app threads) has advanced.
  // Aggressive clock may reduce memory footprint and improve the per-clock
  // convergence rate in the cost of performance.
  // Default is false (suggested).
  bool aggressive_clock;
  ```
  如果是true的，每一个commit（也就是clock()）都要send oplog。
  
## Server thread执行逻辑

 ConnectToNameNode()
 
 Server thread可以connect所有client里面的bg threads。Server thread的功能：
 - 接收到kCreateTable消息后，会HandleCreateTable()
 - 接收到kRowRequest消息后，会HandleRowRequest()
 - 接收到kClientSendOpLog消息后，会HandleOpLogMsg()
 - 接收到kClientShutDown消息后，会HandleShutDownMsg()

 
## StandardMatrixLoader

`num_workers_`是整个集群中的worker thread个数。每一个worker thread有一个访问Matrix的index，这个index被存在`worker_next_el_pos_`中。

Client的main thread会利用StandardMatrixLoader将整个Matrix load到内存，然后让每个worker thread顺序访问。

## matrixfact.CreateTable()
CreateTable() 先设置table的`max_table_staleness`属性，然后调用`Bgworkers::CreateTable(table_id, table_config)`，该函数会将要create的Table信息通过`SendInProc(id_st_=100, msg, msg_size)`发送给Bg thread。

Bg thread initialization logic:
- Establish connections with all server threads (app threads cannot send message to bg threads until this is done);
- Wait on a "Start" message from each server thread;
- Receive connections from all app threads. Server message (currently none for pull model) may come in at the same time.

在初始化`petuum::PSTableGroup::Init(table_group_config, false);`里面就调用Bg thread的`SSPBgThreadMain()`的方法，然后调用`BgWorkers::HandleCreateTables()`方法。由于在Init()的时候，还没有createTable的需求，因此`BgWorkers::HandleCreateTables()`会快速返回。当main()中调用createTable时，比如，` petuum::PSTableGroup::CreateTable(0,table_config);
` 会向bg thread发送createTable的消息（类型是kBgCreateTable），然后标号是100的bg thread会调用HandleCreateTable()，bg thread的HandleCreateTable()会向NamNode发送创建Table的信息，收到NameNode反馈的信息后，会使用下面的语句来真正地创建表，也就是说Table存在于bg thread中：

```c++
client_table  = new ClientTable(table_id, client_table_config);
```

在创建一个Table时，会同时创建其Consistency model，目前只有两种：
- SSP：对应创建 SSPConsistencyController
- SSPPush：对应创建 SSPPushConsistencyController

在MF中，L，R和Loss table的Consistency model都是SSPPush。

ConsistencyController负责控制对Table的访问，提供了GetAsync(row_id)，Get(row_id, row_accessor), ThreadGet(row_id, row_accessor)等方法。其中最重要的方法是Get(row_id)，该方法会check freshness，如果row_id不存在或者stale is too old。

bg thread创建完表以后，会将创建完的信息发送给main() thread。