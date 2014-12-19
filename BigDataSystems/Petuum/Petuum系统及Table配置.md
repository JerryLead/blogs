# Petuum系统及Table配置

## 系统配置

系统相关的配置都会被存放到`petuum::TableGroupConfig`对象中。

### 1. 建立一个distributed app

每一个worker process需要在`host_map`中注册（以一个entry方式存在）。为了简化配置，用户可以定义**optional** server file，交给PS处理。下面的API会用`server_file`中的内容初始化`host_map`。

```c++
void petuum::GetHostInfos(std::string server_file, std::map<int32_t, HostInfo> *host_map);
```

一个Sever file的例子，三列分别是`processID, IP addresss, port`：
```c0 192.168.1.1 100001 192.168.1.2 10000
```
另外，需要告诉PS总共要run多少个process以及每个process的ID，如下表

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
|num\_total\_clients| 1 | Number of processes to run|
| client\_id | 0 | This process's ID |

### 2. 让每个node上run多个worker threads

设置app threads的个数，包含init thread。

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
| num\_total\_app\_threads | 2 | Number of local application threads, including the init threads |

### 3. 建立多个Table

基于以下原因用户可能由建立多个Tables的需求：
- 有多个row types，不同row type对应的row个数（row capacity）也不一样
- 不同的 staleness constraints
- 一个table里row个数太多，放不下

要设置多个table，需要更改下面的默认值：

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
| num\_tables | 1 | 系统包含的table个数|

### 4. Taking SnapShots and Resume From SnapShots

SnapShot可以暂存中间计算结果，对于迭代型的ML算法来说，既可以暂停程序，也可以进行错误恢复，类似与Spark的checkpoint。相关参数见下表：

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
| snapshot\_clock | -1 | take snapshots every `x` iterations |
| snapshot\_dir | "" | 存放snapshot的目录 |
| resume_clock | -1 | if specified, resume from iteration `x` |
| resume\_dir |  ""|从存放snapshot的目录中恢复 |

### 5. Runtime Statistics

PS可以记录runtime statistics，但需要在defns.mk中注释掉下面这一行

```c++
PETUUM_CXXFLAGS += -DPETUUM_STATS
```
## Table配置

### 1. 选择Client cahce types

Client端的cache type由两种：
  
- BoundedDense

	BoundedDense是一个连续的memory chunk，适用于模型能够全部装载到client的memory里面的情况。如果`C`代表cache capacity ，那么此时可以访问到row IDs就是\[0, C-1\]。
- BoundedSparse

	BoundedSparse支持换出操作，因此适合于memory不够的情况。
	
	
### 2. Staleness threshold

Petuum里面最重要的概念就是staleness，也就是允许worker thread最多读取多少轮前的parameters。默认是0，通过下表设置

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
| table\_info.table\_staleness | 0 | SSP staleness threshold |

### 3. Row capacity

| 名称 | 默认值 | 解释 |
|:-------|:----------|:-------|
| table\_info.row\_capacity | 0 | Row capacity |

一些（比如dense）row types需要设置这个参数。1代表sparse，0代表dense。如果updates是dense的，最好设置成dense。