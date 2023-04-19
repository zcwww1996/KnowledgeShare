[TOC]
来源： https://blog.csdn.net/lovehuangjiaju/article/details/49427837

在上一节中的 Stage提交中我们提到，最终stage被封装成TaskSet，使用taskScheduler.submitTasks提交，具体代码如下：

```java
taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, stage.firstJobId, properties))
```

Stage由一系列的tasks组成，这些task被封装成TaskSet，TaskSet类定义如下：

```java
/**
 * A set of tasks submitted together to the low-level TaskScheduler, usually representing
 * missing partitions of a particular stage.
 */
private[spark] class TaskSet(
    val tasks: Array[Task[_]],
    val stageId: Int,
    val stageAttemptId: Int,
    val priority: Int,
    val properties: Properties) {
    val id: String = stageId + "." + stageAttemptId

  override def toString: String = "TaskSet " + id
}
```

submitTasks方法定义在TaskScheduler Trait当中，目前TaskScheduler 只有一个子类TaskSchedulerImpl，其submitTasks方法源码如下：

```java
//TaskSchedulerImpl类中的submitTasks方法
override def submitTasks(taskSet: TaskSet) {
    val tasks = taskSet.tasks
    logInfo("Adding task set " + taskSet.id + " with " + tasks.length + " tasks")
    this.synchronized {
      //创建TaskSetManager，TaskSetManager用于对TaskSet中的Task进行调度，包括跟踪Task的运行、Task失败重试等
      val manager = createTaskSetManager(taskSet, maxTaskFailures)
      val stage = taskSet.stageId
      val stageTaskSets =
        taskSetsByStageIdAndAttempt.getOrElseUpdate(stage, new HashMap[Int, TaskSetManager])
      stageTaskSets(taskSet.stageAttemptId) = manager
      val conflictingTaskSet = stageTaskSets.exists { case (_, ts) =>
        ts.taskSet != taskSet && !ts.isZombie
      }
      if (conflictingTaskSet) {
        throw new IllegalStateException(s"more than one active taskSet for stage $stage:" +
          s" ${stageTaskSets.toSeq.map{_._2.taskSet.id}.mkString(",")}")
      }
      //schedulableBuilder中添加TaskSetManager，用于完成所有TaskSet的调度，即整个Spark程序生成的DAG图对应Stage的TaskSet调度
      schedulableBuilder.addTaskSetManager(manager, manager.taskSet.properties)

      if (!isLocal && !hasReceivedTask) {
        starvationTimer.scheduleAtFixedRate(new TimerTask() {
          override def run() {
            if (!hasLaunchedTask) {
              logWarning("Initial job has not accepted any resources; " +
                "check your cluster UI to ensure that workers are registered " +
                "and have sufficient resources")
            } else {
              this.cancel()
            }
          }
        }, STARVATION_TIMEOUT_MS, STARVATION_TIMEOUT_MS)
      }
      hasReceivedTask = true
    }
    //为Task分配运行资源
    backend.reviveOffers()
  }
```

SchedulerBackend有多种实现，如下图所示：

[![SchedulerBackend](http://p3.so.qhimgs1.com/t0283797dff07de2127.jpg)](https://shop.io.mi-img.com/app/shop/img?id=shop_b9d7d1a3ed2b9e94f87f6cdcdd3b552f.png)

我们以SparkDeploySchedulerBackend为例进行说明，SparkDeploySchedulerBackend继承自CoarseGrainedSchedulerBackend中的reviveOffers方法，具有代码如下：

```java
//CoarseGrainedSchedulerBackend中定义的reviveOffers方法
  override def reviveOffers() {
    //driverEndpoint发送ReviveOffers消息，由DriverEndPoint接受处理
    driverEndpoint.send(ReviveOffers)
  }
driverEndpoint的类型是RpcEndpointRef
//CoarseGrainedSchedulerBackend中的成员变量driverEndpoint
var driverEndpoint: RpcEndpointRef = null
```

它具有如下定义形式：

```java
//RpcEndpointRef是远程RpcEndpoint的引用，它是一个抽象类，有一个子类AkkaRpcEndpointRef
/**
 * A reference for a remote [[RpcEndpoint]]. [[RpcEndpointRef]] is thread-safe.
 */
private[spark] abstract class RpcEndpointRef(@transient conf: SparkConf)
  extends Serializable with Logging 

//在底层采用的是Akka进行实现
private[akka] class AkkaRpcEndpointRef(
    @transient defaultAddress: RpcAddress,
    @transient _actorRef: => ActorRef,
    @transient conf: SparkConf,
    @transient initInConstructor: Boolean = true)
  extends RpcEndpointRef(conf) with Logging {

  lazy val actorRef = _actorRef

  override lazy val address: RpcAddress = {
    val akkaAddress = actorRef.path.address
    RpcAddress(akkaAddress.host.getOrElse(defaultAddress.host),
      akkaAddress.port.getOrElse(defaultAddress.port))
  }

  override lazy val name: String = actorRef.path.name

  private[akka] def init(): Unit = {
    // Initialize the lazy vals
    actorRef
    address
    name
  }

  if (initInConstructor) {
    init()
  }

  override def send(message: Any): Unit = {
    actorRef ! AkkaMessage(message, false)
  }
//其它代码省略
```

DriverEndpoint中的receive方法接收driverEndpoint.send(ReviveOffers)发来的消息，DriverEndpoint继承了ThreadSafeRpcEndpoint trait，具体如下：

```bash
class DriverEndpoint(override val rpcEnv: RpcEnv, sparkProperties: Seq[(String, String)])
    extends ThreadSafeRpcEndpoint with Logging
```

ThreadSafeRpcEndpoint 继承 RpcEndpoint trait，RpcEndpoint对receive方法进行了描述，具体如下：

```java
/**
   * Process messages from [[RpcEndpointRef.send]] or [[RpcCallContext.reply)]]. If receiving a
   * unmatched message, [[SparkException]] will be thrown and sent to `onError`.
   */
  def receive: PartialFunction[Any, Unit] = {
    case _ => throw new SparkException(self + " does not implement 'receive'")
  }
```

DriverEndpoint 中的对其receive方法进行了重写，具体实现如下：

```java
override def receive: PartialFunction[Any, Unit] = {
      case StatusUpdate(executorId, taskId, state, data) =>
        scheduler.statusUpdate(taskId, state, data.value)
        if (TaskState.isFinished(state)) {
          executorDataMap.get(executorId) match {
            case Some(executorInfo) =>
              executorInfo.freeCores += scheduler.CPUS_PER_TASK
              makeOffers(executorId)
            case None =>
              // Ignoring the update since we don't know about the executor.
              logWarning(s"Ignored task status update ($taskId state $state) " +
                s"from unknown executor with ID $executorId")
          }
        }
      //重要！处理发送来的ReviveOffers消息
      case ReviveOffers =>
        makeOffers()

      case KillTask(taskId, executorId, interruptThread) =>
        executorDataMap.get(executorId) match {
          case Some(executorInfo) =>
            executorInfo.executorEndpoint.send(KillTask(taskId, executorId, interruptThread))
          case None =>
            // Ignoring the task kill since the executor is not registered.
            logWarning(s"Attempted to kill task $taskId for unknown executor $executorId.")
        }

    }
```

从上面的代码可以看到，处理ReviveOffers消息时，调用的是makeOffers方法
 
```java
// Make fake resource offers on all executors
    private def makeOffers() {
      // Filter out executors under killing
      //所有可用的Executor
      val activeExecutors = executorDataMap.filterKeys(!executorsPendingToRemove.contains(_))
      //WorkOffer表示Executor上可用的资源，
      val workOffers = activeExecutors.map { case (id, executorData) =>
        new WorkerOffer(id, executorData.executorHost, executorData.freeCores)
      }.toSeq
      //先调用TaskSchedulerImpl的resourceOffers方法，为Task的运行分配资源
      //再调用CoarseGrainedSchedulerBackend中的launchTasks方法启动Task的运行，最终Task被提交到Worker节点上的Executor上运行
      launchTasks(scheduler.resourceOffers(workOffers))
    }
```

上面的代码逻辑全部是在Driver端进行的，调用完launchTasks方法后，Task的执行便在Worker节点上运行了，至此完成Task的提交。