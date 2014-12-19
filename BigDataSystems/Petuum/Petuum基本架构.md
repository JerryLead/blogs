# Petuum 基本架构


## Parameter Server (PS) 概念

PS is a key-value store that allows different processes to share access to a set of variables.

对于Distirbuted ML来说，process指的是learning process，variables指的是parameters。PS的特点是

1.  Data partition
	
	每个节点存放一部分data
2. Shared model

	多个learning process共享model（模型里面包含参数）
	
## 基本系统架构
1. 一个Server

	maintains the master copy of the **parameters** and propagates the workers’ **writes (updates)** to other workers.
2. 多个Worker

	每个worker通过client library去server那里获取parameters。Client library还会cache之前从server那里获取到的parameters，这样worker就不必每次都去Server那里获取最新的parameters。这个cache成为process storage。每次对parameter所做的write(update)操作都会被insert到一个update table（代码里对应**Oplog**）。
	
##  数据模型
 
在PS中，parameters被表示成key-value paris，并存放在多个Table中。每一个table包含多个Rows，每个Row有一个RowID。Row中的每一个cell都包含一个Column ID，每个cell一般存放一个parameter。这样存放到Parameter Server中的每个parameter可以表示成<RowID, ColumnID, Parameter>。

Table-Row既是数据模型也是存储格式。PS允许app选择适合自己的数据结构来组织每个row中的parameters，甚至允许app自定义Rows。

每一个table都有自己的update table，update table也有自己的rows，不过是用来存放log的，这里称之为row  oplog。
	
##  创建一个PS的app

创建一个简单的app，该app包含一个single-threaded的client和一个Table。

### 1. 引入头文件
```c++
#include <petuum_ps_common/include/petuum_ps.hpp>
```

所有app只需包含这个头文件，该文件包含了PS的所有APIs。第一步是去初始化PS的环境，相当于Spark里面的SparkContext，负责初始化的线程被称作init thread。

为了简化例子，我们只run一个worker process。如果要run多个worker process的话，所有的worker process要执行同样的初始化流程，并创建多个tables。

### 2. 注册row types
Row type可以是多种类型，但需要在PS启动计算前注册。下面的API可以创建一个row ID到row type的映射，其中row ID是32位的integer。之后，app可以在创建table时候使用row ID来获取相应的row type。

下面的例子会创建一个row type，类型为vector<T>，这个类型由PS的API提供。Row里面的T就是parameter的类型。更具体地，这里我们注册`petuum::DenseRow<int>`到PS中，并将所有参数初始化为0，如下：

```c++
// register row type petuum::DenseRow<int> with ID 0.petuum::PSTableGroup::RegisterRow<petuum::DenseRow<int> >(0);
```
	
### 3. 初始化PS环境
就像在SparkConf中要设置master，port，app name等，在Petuum中，需要设置`host_map`。我们需要将每一个worker process的信息加入到该map中，形成一个entry。每个entry有一个ID（从0开始计数的整数），一个IP地址，还有一个当前未用的port（比如10000）。具体代码如下：

```c++
petuum::TableGroupConfig table_group_config;table_group_config.host_map.insert(std::make_pair(0, HostInfo(0,    "127.0.0.1", "10000")));
petuum::PSTableGroup::Init(table_group_config, false);
```

将worker process加入到`host_map`中后，就可以使用`petuum::PSTableGroup::Init()`来初始化PS的环境，Init()还包含一个boolean flag，如果设置为true，就表示init thread可以访问table的所有APIs，这些APIs在`petuum:PSTableGroup::GetTableOrDie()`中定义。一般将flag置为false。

### 4. 创建Tables

先show代码

```c++
petuum::ClientTableConfig table_config;table_config.table_info.row_type = 0;table_config.table_info.row_capacity = 100;
table_config.process_cache_capacity = 1000;table_config.oplog_capacity = 1000;
// here 0 is the table ID, which will be used later to get table.bool suc = petuum::PSTableGroup::CreateTable(0, table_config);
```
对于一个app来说，上面的配置参数都需要设置。配置参数的具体含义见下表：

| 名称 | 默认值 | 解释|
|:-----|:------|:-------|
| table\_info.row\_type| N/A | row type|
| table\_info.row\_capacity| 0 | 对于 DenseRow，需要设置它的大小|
| process\_cache\_capacity| 0 | table最多可以包含多少row|
| process\_oplog\_capacity | 0 | update table里面最多可以写入多少个row|

调用`CreateTable()`后，就会去创建tables，创建好后需要调用下面的API来完成table创建过程。

```c++
petuum::PSTableGroup::CreateTableDone();
```
### 5. 创建并运行Worker threads

接下来我们将会创建一个worker thread，该thread可以通过Table接口来访问到parameters。

首先定义一个概念，可以访问table APIs的worker thread被称为**table thread**。

在成为table thread之前，该worker thread需要通过下面的API来注册自己

```c++
int thread_id = petuum::PSTableGroup::RegisterThread();
```
然后就可以通过Table ID来得到table实例：

```c++
petuum::Table<int> table = petuum::PSTableGroup::GetTableOrDie(0);
```
可以通过这个`petuum:Table`类型来访问table里面的parameters，之后可以进行计算。

当worker thread完成计算之后，需要通过下面的API注销自己

```c++
petuum::PSTableGroup::DeregisterThread();
```

如果想让init thread也能访问到table的API，需要将`petuum::PSTableGroup::Init(table_group_config, false);`中的false改为true。init thread不需要注册和注销自己，但它需要通过下面的API等待所有其他thread完成注册。

```c++
petuum::PSTableGroup::WaitThreadRegister();
```

### 6. Stop PS
当所有的worker threads都完成计算退出，我们可以通过下面的API shutdown PS。

```c++
petuum::PSTableGroup::ShutDown();
```

## Table API

### 1. 访问Table
在read或者update table之前，需要先get table
```c++
// Gain access to table.template<typename UPDATE>petuum::Table<UPDATE> petuum::PSTableGroup::GetTableOrDie(int table_id);
```
### 2. Read parameters

先new一个`RowAccessor`对象，给定`row_id`后，下面的API会将row信息写入到`row_accessor`指向的`RowAccessor`对象。

```c++
void petuum::Table::Get(int32_t row_id, RowAccessor *row_accessor);
```

### 3. Update parameters
Petuum提供了两种更新参数的方式：
- 只更新一个parameter
	通过`row_id`和`column_id`定位到parameter，然后更新
	
	```c++
	void petuum::Table<UPDATE>::Inc(int32_t row_id, int32_t column_id, UPDATE update);
	```
- 更新一组参数
	通过`row_id`定位到row，然后更新
	```c++
	void petuum::Table<UPDATE>::BatchInc(int32_t row_id, const UpdateBatch<UPDATE>& update_batch);
	```

### 4. Completion of A Clock Tick

```c++
static void petuum::PSTableGroup::Clock();
```

## 编译

### 1. 编译PS
进入root文件夹，执行：
```c++make third_party_coremake ps_lib -j8
```
PS library依赖很多第三方库，第一条command就是去编译这些库的。

### 2. 编译app
在自己的app目录下建立Makefile，并将`defns.mk`里面的内容加入到app的Makefile中。





