[TOC]

来源：https://blog.csdn.net/lovehuangjiaju/article/details/49474521

在上一节中，我们提到Task提交通过makeOffers提交到Executor上
 
```scala
// Make fake resource offers on just one executor
  private def makeOffers(executorId: String) {
   // Filter out executors under killing
   if (!executorsPendingToRemove.contains(executorId)) {
    val executorData = executorDataMap(executorId)
    val workOffers = Seq(
     new WorkerOffer(executorId, executorData.executorHost, executorData.freeCores))
    launchTasks(scheduler.resourceOffers(workOffers))
    }
   }
```

上面的代码依赖于两个重要方法，它们分别是TaskSchedulerImpl resourceOffers方法及CoarseGrainedSchedulerBackend的launchTasks方法


```scala
//TaskSchedulerImpl resourceOffers方法
 /**
   * Called by cluster manager to offer resources on slaves. We respond by asking our active task
   * sets for tasks in order of priority. We fill each node with tasks in a round-robin manner so
   * that tasks are balanced across the cluster.
   */
  def resourceOffers(offers: Seq[WorkerOffer]): Seq[Seq[TaskDescription]] = synchronized {
    // Mark each slave as alive and remember its hostname
    // Also track if new executor is added
    // 处理新的executor加入
    var newExecAvail = false
    for (o <- offers) {
      executorIdToHost(o.executorId) = o.host
      activeExecutorIds += o.executorId
      if (!executorsByHost.contains(o.host)) {
        executorsByHost(o.host) = new HashSet[String]()
        executorAdded(o.executorId, o.host)
        newExecAvail = true
      }
      for (rack <- getRackForHost(o.host)) {
        hostsByRack.getOrElseUpdate(rack, new HashSet[String]()) += o.host
      }
    }

    // Randomly shuffle offers to avoid always placing tasks on the same set of workers.
    //随机打散，使Task均匀分配各Worker节点上
    val shuffledOffers = Random.shuffle(offers)
    // Build a list of tasks to assign to each worker.
    val tasks = shuffledOffers.map(o => new ArrayBuffer[TaskDescription](o.cores))
    val availableCpus = shuffledOffers.map(o => o.cores).toArray

    //根据调度策略获取ArrayBuffer[TaskSetManager]
    val sortedTaskSets = rootPool.getSortedTaskSetQueue
    for (taskSet <- sortedTaskSets) {
      logDebug("parentName: %s, name: %s, runningTasks: %s".format(
        taskSet.parent.name, taskSet.name, taskSet.runningTasks))
      if (newExecAvail) {
        taskSet.executorAdded()
      }
    }

    // Take each TaskSet in our scheduling order, and then offer it each node in increasing order
    // of locality levels so that it gets a chance to launch local tasks on all of them.
    // NOTE: the preferredLocality order: PROCESS_LOCAL, NODE_LOCAL, NO_PREF, RACK_LOCAL, ANY
    //按就近原则进行Task调度
    var launchedTask = false
    for (taskSet <- sortedTaskSets; maxLocality <- taskSet.myLocalityLevels) {
      do {
        launchedTask = resourceOfferSingleTaskSet(
            taskSet, maxLocality, shuffledOffers, availableCpus, tasks)
      } while (launchedTask)
    }

    if (tasks.size > 0) {
      hasLaunchedTask = true
    }
    return tasks
  }
```

调用完resourceOffers方法后，再调用launchTasks方法，最终在Worker节点上启动任务的运行


```scala
//CoarseGrainedSchedulerBackend中的launchTasks方法
 // Launch tasks returned by a set of resource offers
    private def launchTasks(tasks: Seq[Seq[TaskDescription]]) {
      for (task <- tasks.flatten) {
        val serializedTask = ser.serialize(task)
        //序列化后的任何不能超过设定的大小
        if (serializedTask.limit >= akkaFrameSize - AkkaUtils.reservedSizeBytes) {
          scheduler.taskIdToTaskSetManager.get(task.taskId).foreach { taskSetMgr =>
            try {
              var msg = "Serialized task %s:%d was %d bytes, which exceeds max allowed: " +
                "spark.akka.frameSize (%d bytes) - reserved (%d bytes). Consider increasing " +
                "spark.akka.frameSize or using broadcast variables for large values."
              msg = msg.format(task.taskId, task.index, serializedTask.limit, akkaFrameSize,
                AkkaUtils.reservedSizeBytes)
              taskSetMgr.abort(msg)
            } catch {
              case e: Exception => logError("Exception in error callback", e)
            }
          }
        }
        else {
          val executorData = executorDataMap(task.executorId)
          executorData.freeCores -= scheduler.CPUS_PER_TASK
          //Worker节点上的CoarseGrainedExecutorBackend对象将接受LaunchTask消息，在Worker节点的Executor上启动Task的执行
          executorData.executorEndpoint.send(LaunchTask(new SerializableBuffer(serializedTask)))
        }
      }
    }
```

