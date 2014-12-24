# STRADS


STRADS意思是STRucture-Aware Dynamic Scheduler，一个动态调度框架，用于计算Big Model类型的app。STRADS调度的是模型参数，而不是data，调度参数会让参数更新和收敛的速度更快，但要求在参数间没有依赖。目前包含两个app：Lasso和Logistic Regression。

## STRADS的四个组件
四个组件组成了scatter/gather的拓扑结构。A Coordinator, multiple workers and an aggregator make a scatter/gather style topology.

### 1. Scheduler

Scheduler maintain weight information of model parameter. In each iteration, scheduler selects a set of promising model parameters to dispatch based on the weight information so that updating the scheduled parameters is likely to increase convergence speed than updating randomly selected parameter as common in stochastic method. Scheduler update weight information on receiving weight change from the coordinator when the dispatched is completed in the coordinator side. In addition to weight based sampling, STRADS scheduler runs user defined model dependency checking routine for a give set of model parameters. If any pair of parameters has too strong interference, one of them will be removed from the set.

Scheduler维护参数间的权重信息$w$。在每次迭代时，scheduler先按照权重选择一部分promising的参数集合$S$分发给worker，这样的选择方式比random的选择方式能更快地收敛。当参数集合$S$计算并更新完毕后（这个过程由coordinator负责），coordinator会将$w$的更新信息发给scheduler。另外，scheduler可以运行user defined model 依赖检测程序来检测是否参数间有强依赖关系，如果有，那么去掉一个参数。


### 2. 一个Coordinator

Coordinator is in charge of keeping model parameters, scattering a dispatch of parameter over the worker machines, sending back weight change information to the scheduler. In i-th iteration, the coordinator receive a dispatch set from the scheduler and scatter the dispatch together to all over the worker machines. On receiving updated model parameter values from the aggregator, it will udpate model parameters and send weight change information to the scheduler.

Coordinator负责持有参数，分发参数给worker，更新参数，并将更新后的$w$信息发给scheduler。当迭代到第i轮时，coordinator会先收到scheduler发来的一个参数集合（dispatch set），然后将参数集合发到所有的worker节点。Worker计算参数更新$\Delta\theta$后，将$\Delta\theta$发给aggregator，aggregator负责更新参数，并将更新结果发给Coordinator，Coordinator存储新的$\theta$后将权重$w$更新信息发给scheduler。

### 3. 多个Worker

On receiving a dispatch, worker executes user function to make a partial result with a partition of input data that is assigned to the worker machine. Each worker sends back its partial results to the aggregator.

当收到参数集合$S$后，worker在data partition上执行user function，并将计算结果$\Delta\theta$发给aggregator。

### 4. Aggregator

On collecting all partial results of one dispatch, aggregator runs user defined aggregation function to get new value of model parameters. New model parameter values are sent to the coordinator to be kept.

当收到所有的所有worker发来的partial results后，aggregator运行user defined aggregation function来计算出新的参数$theta$，然后将新的参数发送给coordinator存放。


## STRADS提供的其他low level primitives

- Scheduling
- Global Barrier
- Data Partitioning
- Message abstraction

## STRADS编程接口

整个编程范型是 scatter/gather。

Basically, STRADS allows users to define functions to run on scheduler, coordinator, workers and aggregator vertexes. In addition to the functions, use can define message types as C++ template for communicating across different vertexes. STRADS programming interfaces are implemented in the form of two classes. 

STRADS允许在scheduler，coordinator，workers和aggregator vertexes上自定义函数。除了自定义函数外，用户也可以定义不同vertex之间消息传递的message type。具体地， STRADS的编程接口由下面两个类实现：

### Handler Class

Handler class is a template class where user can define user functions as class method here. T1 ~ T4 are template of user defined messages and used as parameter and return type of class methods(user functions). STRADS requires 4 major user functions for scheduling/updating parameters and three minor function for checking progress such as calculating objective function value.

用于调度和参数更新的4个Major User Functions，T1 ~ T4 是用户定义的消息类型
```c++
    T1 &dispatch_scheduling(SYSMSG, T3) 
    void do_work(T1, T2) 
    void do_msgcombiner(T2, stradsctx)
    void do_aggregator(T3, stradsctx)
    int check_dependency(list parameters)
    set_initi_priority(list weight, model_cnt)
```
Minor User Functions for progress checking，比如要查看目标函数的value。
```c++
    void do_obj_calc(T4, stradsctx)
    void do_msgcombiner_obj(T4, stradsctx)
    void do_object_aggregation(T4, stradsctx)
```
### Message Class

Message class is a template class that allows user to define a type of messages that contains arbitrary number of elements. Logically, user can define any type for the element. STRADS provides several template classes that can make a message with one, two or three different kinds of element types. Again, you can put arbitrary number of elements with different types on a message. If you define your message type only with POD type, you can simply finish message class definition with defining elements type.

    element class
    message class

消息可以包含任意多个elements，每个element可以是任意类型。



