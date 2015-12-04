## Spark scheduler

### Master 分配 Executor 方法
    
```scala
  private def schedule(): Unit = {
    // 先打乱 workers
    val shuffledWorkers = Random.shuffle(workers) 
    // 对通过 Spark-submit 提交（也就是 AppClient 类提交）的 app 来说下面这个 for 语句没用
    for (worker <- shuffledWorkers if worker.state == WorkerState.ALIVE) {
      for (driver <- waitingDrivers) {
        if (worker.memoryFree >= driver.desc.mem && worker.coresFree >= driver.desc.cores) {
          launchDriver(worker, driver)
          waitingDrivers -= driver
        }
      }
    }
    // 开始在 workers 上分配 executors
    startExecutorsOnWorkers()
  }

```
#### `startExecutorsOnWorkers()` 逻辑

```scala
private def startExecutorsOnWorkers(): Unit = {
    // FIFO 调度策略
    for (app <- waitingApps if app.coresLeft > 0) {
      // 得到每个 executor 需要的 cores 数目
      val coresPerExecutor: Option[Int] = app.desc.coresPerExecutor
      // 挑选出可用的 workers，将可用 workers 的资源（空闲 CPU core 个数）按照降序排列
      val usableWorkers = workers.toArray.filter(_.state == WorkerState.ALIVE)
        .filter(worker => worker.memoryFree >= app.desc.memoryPerExecutorMB &&
          worker.coresFree >= coresPerExecutor.getOrElse(1))
        .sortBy(_.coresFree).reverse
      // 资源分配算法，assignedCores 是一个数组，第 i 个元素表示应该往第 i个 usableWorkers 上分配多少个 core
      val assignedCores = scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)

      // 已经得到应该往每个 worker 上分配多少个 core，开始分配
      for (pos <- 0 until usableWorkers.length if assignedCores(pos) > 0) {
        allocateWorkerResourceToExecutors(
          app, assignedCores(pos), coresPerExecutor, usableWorkers(pos))
      }
    }
  }
```

#### `scheduleExecutorsOnWorkers(app, usableWorkers, spreadOutApps)`逻辑

```scala
  private def scheduleExecutorsOnWorkers(
      app: ApplicationInfo,
      usableWorkers: Array[WorkerInfo],
      spreadOutApps: Boolean): Array[Int] = {
    // 首先进行一系列初始化
    // 每个 executor 需要多少个 core
    val coresPerExecutor = app.desc.coresPerExecutor
    // 每个 executor 最少需要多少个 core，默认是 1
    val minCoresPerExecutor = coresPerExecutor.getOrElse(1)
    // 如果用户没有设置 coresPerExecutor，那么 oneExecutorPerWorker 为 true
    val oneExecutorPerWorker = coresPerExecutor.isEmpty
    // 每个 executor 需要的 memory 用量
    val memoryPerExecutor = app.desc.memoryPerExecutorMB
    // 可用的 workers 个数
    val numUsable = usableWorkers.length
    // 每个 worker 要提供的 cores 个数
    val assignedCores = new Array[Int](numUsable) 
    // 在每个 worker 上分配的 executor 个数
    val assignedExecutors = new Array[Int](numUsable) 
    // 要分配的 core 个数 = min(app 需求的 cores，workers 剩余 cores 之和)
    var coresToAssign = math.min(app.coresLeft, usableWorkers.map(_.coresFree).sum)


    // Keep launching executors until no more workers can accommodate any
    // more executors, or if we have reached this application's limits

    // 从所有 workers 中筛选出可用的 workers，筛选算法见 canLauchExecutor
    var freeWorkers = (0 until numUsable).filter(canLaunchExecutor)
    
    while (freeWorkers.nonEmpty) {
      freeWorkers.foreach { pos =>
        var keepScheduling = true
        // 如果该 worker 上可以启动 executor
        while (keepScheduling && canLaunchExecutor(pos)) {
          // 需要分配的 cores 的数目减去每个 executor 需要的 core 个数
          coresToAssign -= minCoresPerExecutor
          // 将分配好的 core 信息保存到 assignedCores 里面
          assignedCores(pos) += minCoresPerExecutor

          // If we are launching one executor per worker, then every iteration assigns 1 core
          // to the executor. Otherwise, every iteration assigns cores to a new executor.
          if (oneExecutorPerWorker) {
            assignedExecutors(pos) = 1
          } else {
            assignedExecutors(pos) += 1
          }

          // Spreading out an application means spreading out its executors across as
          // many workers as possible. If we are not spreading out, then we should keep
          // scheduling executors on this worker until we use all of its resources.
          // Otherwise, just move on to the next worker.
          // 如果选择 spreadOut 模式，那么在一个 worker 上分配一个 executor 的 cores 后，就更换
          // worker 再分配
          if (spreadOutApps) {
            keepScheduling = false
          }
        }
      }
      // 从 freeWorkers 中再挑选出可以启动 executor 的 workers
      freeWorkers = freeWorkers.filter(canLaunchExecutor)
    }
    返回在 workers 上分配的 CPU core 资源信息
    assignedCores
```

#### `scheduleExecutorsOnWorkers().canLaunchExecutor`逻辑

```scala
   /** Return whether the specified worker can launch an executor for this app. */
    def canLaunchExecutor(pos: Int): Boolean = {
      // 如果 app 里要分配的 core 个数大于每个 executor 需要的个数，仍然继续调度
      val keepScheduling = coresToAssign >= minCoresPerExecutor
      // 当前 worker 是否有足够的 core 来分配
      val enoughCores = usableWorkers(pos).coresFree - assignedCores(pos) >= minCoresPerExecutor

      // If we allow multiple executors per worker, then we can always launch new executors.
      // Otherwise, if there is already an executor on this worker, just give it more cores.
      val launchingNewExecutor = !oneExecutorPerWorker || assignedExecutors(pos) == 0
      // 如果每个 worker 可以分配多个 executor，或者这个 worker 上还没分配到 executor
      if (launchingNewExecutor) {
        // 计算要在该 worker 上分配了多少 memory
        val assignedMemory = assignedExecutors(pos) * memoryPerExecutor
        // 计算该 worker 上是否有足够的 memory 来分配 executor
        val enoughMemory = usableWorkers(pos).memoryFree - assignedMemory >= memoryPerExecutor
        // 检测是否超过 app executor 数目上限，用于动态调度
        val underLimit = assignedExecutors.sum + app.executors.size < app.executorLimit
        keepScheduling && enoughCores && enoughMemory && underLimit
      } else {
        // We're adding cores to an existing executor, so no need
        // to check memory and executor limits
        // 如果每个 worker 只运行一个 executor，那么直接在该 executor 上增加 core 个数
        keepScheduling && enoughCores
      }
    }
```

#### `allocateWorkerResourceToExecutors()`逻辑

```scala
  /**
   * Allocate a worker's resources to one or more executors.
   * @param app the info of the application which the executors belong to
   * @param assignedCores number of cores on this worker for this application
   * @param coresPerExecutor number of cores per executor
   * @param worker the worker info
   */
  private def allocateWorkerResourceToExecutors(
      app: ApplicationInfo,
      assignedCores: Int,
      coresPerExecutor: Option[Int],
      worker: WorkerInfo): Unit = {
    // If the number of cores per executor is specified, we divide the cores assigned
    // to this worker evenly among the executors with no remainder.
    // Otherwise, we launch a single executor that grabs all the assignedCores on this worker.

    // 计算需要在该 worker 上启动多少个 executors (assignedCores / coresPerExecutor)
    val numExecutors = coresPerExecutor.map { assignedCores / _ }.getOrElse(1)
    // 每个 executor 需要多少个 core
    val coresToAssign = coresPerExecutor.getOrElse(assignedCores)
    // 将 executor 信息加到 app 上，在 worker 上启动相应的 executor
    for (i <- 1 to numExecutors) {
      val exec = app.addExecutor(worker, coresToAssign)
      launchExecutor(worker, exec)
      app.state = ApplicationState.RUNNING
    }
  }
```