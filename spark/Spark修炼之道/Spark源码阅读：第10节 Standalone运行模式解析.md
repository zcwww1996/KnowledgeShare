[TOC]

来源： https://blog.csdn.net/lovehuangjiaju/article/details/49747437

Spark Standalone采用的是Master/Slave架构，主要涉及到的类包括：


```scala
类：org.apache.spark.deploy.master.Master
说明：负责整个集群的资源调度及Application的管理。
消息类型：
接收Worker发送的消息
1. RegisterWorker
2. ExecutorStateChanged
3. WorkerSchedulerStateResponse
4. Heartbeat

向Worker发送的消息
1. RegisteredWorker
2. RegisterWorkerFailed
3. ReconnectWorker
4. KillExecutor
5.LaunchExecutor
6.LaunchDriver
7.KillDriver
8.ApplicationFinished

向AppClient发送的消息
1. RegisteredApplication
2. ExecutorAdded
3. ExecutorUpdated
4. ApplicationRemoved
接收AppClient发送的消息
1. RegisterApplication
2. UnregisterApplication
3. MasterChangeAcknowledged
4. RequestExecutors
5. KillExecutors

向Driver Client发送的消息
1.SubmitDriverResponse
2.KillDriverResponse
3.DriverStatusResponse

接收Driver Client发送的消息
1.RequestSubmitDriver 
2.RequestKillDriver         
3.RequestDriverStatus
类org.apache.spark.deploy.worker.Worker
说明：向Master注册自己并启动CoarseGrainedExecutorBackend，在运行时启动Executor运行Task任务
消息类型：
向Master发送的消息
1. RegisterWorker
2. ExecutorStateChanged
3. WorkerSchedulerStateResponse
4. Heartbeat
接收Master发送的消息
1. RegisteredWorker
2. RegisterWorkerFailed
3. ReconnectWorker
4. KillExecutor
5.LaunchExecutor
6.LaunchDriver
7.KillDriver
8.ApplicationFinished
类org.apache.spark.deploy.client.AppClient.ClientEndpoint
说明：向Master注册并监控Application，请求或杀死Executors等
消息类型：
向Master发送的消息
1. RegisterApplication
2. UnregisterApplication
3. MasterChangeAcknowledged
4. RequestExecutors
5. KillExecutors
接收Master发送的消息
1. RegisteredApplication
2. ExecutorAdded
3. ExecutorUpdated
4. ApplicationRemoved
类：org.apache.spark.scheduler.cluster.DriverEndpoint
说明：运行时注册Executor并启动Task的运行并处理Executor发送来的状态更新等
消息类型：
向Executor发送的消息
1.LaunchTask
2.KillTask
3.RegisteredExecutor
4.RegisterExecutorFailed
接收Executor发送的消息
1.RegisterExecutor
2.StatusUpdate
类：org.apache.spark.deploy.ClientEndpoint 
说明：管理Driver包括提交Driver、Kill掉Driver及获取Driver状态信息
向Master发送的消息
1.RequestSubmitDriver 
2.RequestKillDriver         
3.RequestDriverStatus

接收Master 发送的消息
1.SubmitDriverResponse
2.KillDriverResponse
3.DriverStatusResponse
```

上面所有的类都继承自org.apache.spark.rpc.ThreadSafeRpcEndpoint，其底层实现目前都是通过AKKA来实现的，具体如下图所示：

[![2.png](https://www.helloimg.com/images/2020/08/20/2dda38583b39107a8.png)](https://ae03.alicdn.com/kf/U667f4f9ba5cb447aaa90e5b737eda164U.jpg)

<br>

各类之间的交互关系如下图所示：

[![1.png](https://www.helloimg.com/images/2020/08/20/14a555939da8ad383.png)](https://ae03.alicdn.com/kf/Udbdf2a58935243a2950aca1cd7ccc6780.jpg)

# 1. AppClient与Master间的交互
SparkContext在创建时，会调用，createTaskScheduler方法创建相应的TaskScheduler及SchedulerBackend


```scala
 // Create and start the scheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master)
    _schedulerBackend = sched
    _taskScheduler = ts
    _dagScheduler = new DAGScheduler(this)
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler‘s
    // constructor
    _taskScheduler.start()
Standalone运行模式创建的TaskScheduler及SchedulerBackend具体源码如下：

/**
   * Create a task scheduler based on a given master URL.
   * Return a 2-tuple of the scheduler backend and the task scheduler.
   */
  private def createTaskScheduler(
      sc: SparkContext,
      master: String): (SchedulerBackend, TaskScheduler) = {

   //省略其它非关键代码
   case SPARK_REGEX(sparkUrl) =>
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)
    //省略其它非关键代码

}
```

创建完成TaskScheduler及SchedulerBackend后，调用TaskScheduler的start方法，启动SchedulerBackend(Standalone模式对应SparkDeploySchedulerBackend）

```scala
//TaskSchedulerImpl中的start方法 
override def start() {
    //调用SchedulerBackend的start方法 
    backend.start()
    //省略其它非关键代码
  }
```

对应SparkDeploySchedulerBackend中的start方法源码如下：

```
override def start() {
    super.start()

    //省略其它非关键代码
    //Application相关信息（包括应用程序名、executor运行内存等）
    val appDesc = new ApplicationDescription(sc.appName, maxCores, sc.executorMemory,
      command, appUIAddress, sc.eventLogDir, sc.eventLogCodec, coresPerExecutor)
    //创建AppClient，传入相应启动参数
    client = new AppClient(sc.env.rpcEnv, masters, appDesc, this, conf)
    client.start()
    waitForRegistration()
  }
```

AppClient类中的start方法原码如下：

```scala
//AppClient Start方法
  def start() {
    // Just launch an rpcEndpoint; it will call back into the listener.
    //ClientEndpoint，该ClientEndpoint为AppClient的内部类
    //它是AppClient的rpcEndpoint
    endpoint = rpcEnv.setupEndpoint("AppClient", new ClientEndpoint(rpcEnv))
  }
ClientEndpoint在启动时，会向Master注册Application

override def onStart(): Unit = {
      try {
        registerWithMaster(1)
      } catch {
        case e: Exception =>
          logWarning("Failed to connect to master", e)
          markDisconnected()
          stop()
      }
    }
```

registerWithMaster方法便是向Master注册Application，其源码如下：

```scala
/**
     * Register with all masters asynchronously. It will call `registerWithMaster` every
     * REGISTRATION_TIMEOUT_SECONDS seconds until exceeding REGISTRATION_RETRIES times.
     * Once we connect to a master successfully, all scheduling work and Futures will be cancelled.
     *
     * nthRetry means this is the nth attempt to register with master.
     */
    private def registerWithMaster(nthRetry: Int) {
      registerMasterFutures = tryRegisterAllMasters()
      //注册失败时重试
      registrationRetryTimer = registrationRetryThread.scheduleAtFixedRate(new Runnable {
        override def run(): Unit = {
          Utils.tryOrExit {
            if (registered) {
              registerMasterFutures.foreach(_.cancel(true))
              registerMasterThreadPool.shutdownNow()
            } else if (nthRetry >= REGISTRATION_RETRIES) {
              markDead("All masters are unresponsive! Giving up.")
            } else {
              registerMasterFutures.foreach(_.cancel(true))
              registerWithMaster(nthRetry + 1)
            }
          }
        }
      }, REGISTRATION_TIMEOUT_SECONDS, REGISTRATION_TIMEOUT_SECONDS, TimeUnit.SECONDS)
    }
```

向所有Masters注册，因为Master可能实现了高可靠（HA)，例如ZooKeeper的HA方式，所以存在多个Master，但最终只有Active Master响应，具体源码如下：


```scala
 /**
     *  Register with all masters asynchronously and returns an array `Future`s for cancellation.
     */
    private def tryRegisterAllMasters(): Array[JFuture[_]] = {
      for (masterAddress <- masterRpcAddresses) yield {
        registerMasterThreadPool.submit(new Runnable {
          override def run(): Unit = try {
            if (registered) {
              return
            }
            logInfo("Connecting to master " + masterAddress.toSparkURL + "...")
            //获取Master rpcEndpoint
            val masterRef =
              rpcEnv.setupEndpointRef(Master.SYSTEM_NAME, masterAddress, Master.ENDPOINT_NAME)
            //向Master发送RegisterApplication信息            masterRef.send(RegisterApplication(appDescription, self))
          } catch {
            case ie: InterruptedException => // Cancelled
            case NonFatal(e) => logWarning(s"Failed to connect to master $masterAddress", e)
          }
        })
      }
    }
```

Master会接收来自AppClient的RegisterApplication消息，具体源码如下：

```scala
//org.apache.spark.deploy.master.Master.receive方法接受AppClient发送来的RegisterApplication消息
override def receive: PartialFunction[Any, Unit] = {

    case RegisterApplication(description, driver) => {
      // TODO Prevent repeated registrations from some driver
      if (state == RecoveryState.STANDBY) {
        // ignore, don‘t send response
      } else {
        logInfo("Registering app " + description.name)
        //创建ApplicationInfo
        val app = createApplication(description, driver)
        //注册Application
        registerApplication(app)
        logInfo("Registered app " + description.name + " with ID " + app.id)
        persistenceEngine.addApplication(app)
        //向AppClient发送RegisteredApplication消息
        driver.send(RegisteredApplication(app.id, self))
        schedule()
      }
    }
```

AppClient内部类ClientEndpoint接收Master发来的RegisteredApplication消息

```scala
 override def receive: PartialFunction[Any, Unit] = {
      case RegisteredApplication(appId_, masterRef) =>
        // FIXME How to handle the following cases?
        // 1. A master receives multiple registrations and sends back multiple
        // RegisteredApplications due to an unstable network.
        // 2. Receive multiple RegisteredApplication from different masters because the master is
        // changing.
        appId = appId_
        registered = true
        master = Some(masterRef)
        listener.connected(appId)
       //省略其它非关键代码
}
```

通过上述过程便完成Application的注册。其它交互信息如下

```scala
//------------------AppClient 向 Master  发送的消息------------------//

  //AppClient向Master注册Application
  case class RegisterApplication(appDescription: ApplicationDescription, driver: RpcEndpointRef)
    extends DeployMessage

 //AppClient向Master注销Application
  case class UnregisterApplication(appId: String)

 //Master从故障中恢复后，发送MasterChange消息给AppClient，AppClient接收到该消息后，更改保存的Master信息，然后发送MasterChangeAcknowledged给Master
  case class MasterChangeAcknowledged(appId: String)

//为Application的运行申请数量为requestedTotal的Executor
  case class RequestExecutors(appId: String, requestedTotal: Int)

//杀死Application对应的Executors
  case class KillExecutors(appId: String, executorIds: Seq[String])
//------------------Master  向 AppClient 发送的消息------------------//

  //向AppClient发送Application注册成功的消息
  case class RegisteredApplication(appId: String, master: RpcEndpointRef) extends DeployMessage

  // TODO(matei): replace hostPort with host
  //Worker启动了Executor后，发送该消息通知AppClient
  case class ExecutorAdded(id: Int, workerId: String, hostPort: String, cores: Int, memory: Int) {
    Utils.checkHostPort(hostPort, "Required hostport")
  }
   //Executor状态更新后，发送该消息通知AppClient
  case class ExecutorUpdated(id: Int, state: ExecutorState, message: Option[String],
    exitStatus: Option[Int])

  //Application成功运行或失败时，Master发送该消息给AppClient
  //AppClient接收该消息后，停止Application的运行
  case class ApplicationRemoved(message: String)

   // Master 发生变化时，会利用MasterChanged消息通知Worker及AppClient
  case class MasterChanged(master: RpcEndpointRef, masterWebUiUrl: String)
```

# 2. Master与Worker间的交互
这里只给出其基本的消息交互，后面有时间再来具体分析。

```scala
//------------------Worker向Master发送的消息------------------//

  //向Master注册Worker，Master在完成Worker注册后，向Worker发送RegisteredWorker消息，此后便可接收来自Master的调度
  case class RegisterWorker(
      id: String,
      host: String,
      port: Int,
      worker: RpcEndpointRef,
      cores: Int,
      memory: Int,
      webUiPort: Int,
      publicAddress: String)
    extends DeployMessage {
    Utils.checkHost(host, "Required hostname")
    assert (port > 0)
  }

  //向Master汇报Executor的状态变化
  case class ExecutorStateChanged(
      appId: String,
      execId: Int,
      state: ExecutorState,
      message: Option[String],
      exitStatus: Option[Int])
    extends DeployMessage

  //向Master汇报Driver状态变化
  case class DriverStateChanged(
      driverId: String,
      state: DriverState,
      exception: Option[Exception])
    extends DeployMessage

  //Worker向Master汇报其运行的Executor及Driver信息
  case class WorkerSchedulerStateResponse(id: String, executors: List[ExecutorDescription],
     driverIds: Seq[String])

  //Worker向Master发送的心跳信息，主要向Master报活
  case class Heartbeat(workerId: String, worker: RpcEndpointRef) extends DeployMessage
```

 
```scala
 //------------------Master向Worker发送的消息------------------//

  //Worker发送RegisterWorker消息注册Worker，注册成功后Master回复RegisteredWorker消息给Worker
  case class RegisteredWorker(master: RpcEndpointRef, masterWebUiUrl: String) extends DeployMessage

  //Worker发送RegisterWorker消息注册Worker，注册失败后Master回复RegisterWorkerFailed消息给Worker
  case class RegisterWorkerFailed(message: String) extends DeployMessage

   //Worker心跳超时后，Master向Worker发送ReconnectWorker消息，通知Worker节点需要重新注册
  case class ReconnectWorker(masterUrl: String) extends DeployMessage

  //application运行完毕后，Master向Worker发送KillExecutor消息，Worker接收到消息后，删除对应execId的Executor
  case class KillExecutor(masterUrl: String, appId: String, execId: Int) extends DeployMessage
   //向Worker节点发送启动Executor消息
  case class LaunchExecutor(
      masterUrl: String,
      appId: String,
      execId: Int,
      appDesc: ApplicationDescription,
      cores: Int,
      memory: Int)
    extends DeployMessage

  //向Worker节点发送启动Driver消息
  case class LaunchDriver(driverId: String, driverDesc: DriverDescription) extends DeployMessage

  //杀死对应Driver
  case class KillDriver(driverId: String) extends DeployMessage

  case class ApplicationFinished(id: String)
```

# 3. Driver Client与 Master间的消息交互
Driver Client主要是管理Driver，包括向Master提交Driver、请求杀死Driver等，其源码位于org.apache.spark.deploy.client.scala源码文件当中，类名为：org.apache.spark.deploy.ClientEndpoint。要注意其与org.apache.spark.deploy.client.AppClient.ClientEndpoint类的本质不同。

```scala
 //------------------Driver Client间Master信息的交互------------------//

  //Driver Client向Master请求提交Driver
  case class RequestSubmitDriver(driverDescription: DriverDescription) extends DeployMessage
  //Master向Driver Client返回注册是否成功的消息
  case class SubmitDriverResponse(
      master: RpcEndpointRef, success: Boolean, driverId: Option[String], message: String)
    extends DeployMessage


  //Driver Client向Master请求Kill Driver
  case class RequestKillDriver(driverId: String) extends DeployMessage
  // Master回复Kill Driver是否成功
  case class KillDriverResponse(
      master: RpcEndpointRef, driverId: String, success: Boolean, message: String)
    extends DeployMessage


  //Driver Client向Master请求Driver状态
  case class RequestDriverStatus(driverId: String) extends DeployMessage
  //Master向Driver Client返回状态请求信息
  case class DriverStatusResponse(found: Boolean, state: Option[DriverState],
    workerId: Option[String], workerHostPort: Option[String], exception: Option[Exception])
```

4. Driver与Executor间的消息交互


```scala
//------------------Driver向Executor发送的消息------------------//
  //启动Task
  case class LaunchTask(data: SerializableBuffer) extends CoarseGrainedClusterMessage
  //杀死Task
  case class KillTask(taskId: Long, executor: String, interruptThread: Boolean)
    extends CoarseGrainedClusterMessage
 //Executor注册成功
case object RegisteredExecutor extends CoarseGrainedClusterMessage
 //Executor注册失败
case class RegisterExecutorFailed(message: String) extends CoarseGrainedClusterMessage



//------------------Executor向Driver发送的消息------------------//
 //向Driver注册Executor
 case class RegisterExecutor(
      executorId: String,
      executorRef: RpcEndpointRef,
      hostPort: String,
      cores: Int,
      logUrls: Map[String, String])
    extends CoarseGrainedClusterMessage {
    Utils.checkHostPort(hostPort, "Expected host port")
  }

 //向Driver汇报状态变化
 case class StatusUpdate(executorId: String, taskId: Long, state: TaskState,
    data: SerializableBuffer) extends CoarseGrainedClusterMessage

  object StatusUpdate {
    /** Alternate factory method that takes a ByteBuffer directly for the data field */
    def apply(executorId: String, taskId: Long, state: TaskState, data: ByteBuffer)
      : StatusUpdate = {
      StatusUpdate(executorId, taskId, state, new SerializableBuffer(data))
    }
  }
```

