# CreateTable过程

## 基本流程

1. 每个App main Thread（比如每个节点上的matrixfact.main()）调用`petuum::PSTableGroup::CreateTable(tableId, table_config)`来创建Table。
2. 该方法会调用同在一个Process里的bg head thread向NameNode thread发送创建Table的请求。
3. NameNode收到CreateTable请求，如果该Table还未创建，就在自己的线程里创建一个ServerTable。之后会忽略其他要创建同一Table的请求。
4. NameNode将CreateTable请求发送到cluster中的每个Server thread。
5. Server thread收到CreateTable请求后，先reply msg to NameNode thread，表示自己已经知道要创建Table，然后直接在线程里创建一个ServerTable。
6. 当NameNode thread收到cluster中所有Server thread返回的reply消息后，就开始reply bg head thread说“Table已被ServerThreads创建”。
7. 当App main()里定义的所有的Table都被创建完毕（比如matrixfact里要创建三个Table），NameNode thread会向cluster中所有head bg thread发送“所有的Tables都被创建了”的消息。

代码结构图

![CreateTable](figures/CreateTable.png)