[TOC]
博文推荐：http://blog.csdn.net/anzhsoft/article/details/39268963<br>
由大神张安站写的Spark架构原理，使用Spark版本为1.2，本文以Spark 1.5.0为蓝本，介绍Spark应用程序的执行流程。 
本文及后面的源码分析都以下列代码为样板

来源： https://blog.csdn.net/lovehuangjiaju/article/details/49391307


```java
import org.apache.spark.{SparkConf, SparkContext}

object SparkWordCount{
  def main(args: Array[String]) {
    if (args.length == 0) {
      System.err.println("Usage: SparkWordCount <inputfile> <outputfile>")
      System.exit(1)
    }

    val conf = new SparkConf().setAppName("SparkWordCount")
    val sc = new SparkContext(conf)

    val file=sc.textFile("file:///hadoopLearning/spark-1.5.1-bin-hadoop2.4/README.md")
    val counts=file.flatMap(line=>line.split(" "))
                   .map(word=>(word,1))
                   .reduceByKey(_+_)
    counts.saveAsTextFile("file:///hadoopLearning/spark-1.5.1-bin-hadoop2.4/countReslut.txt")

  }
}
```

代码中的SparkContext在Spark应用程序的执行过程中起着主导作用，它负责与程序个Spark集群进行交互，包括申请集群资源、创建RDD、accumulators 及广播变量等。SparkContext与集群资源管理器、Worker结节点交互图如下图所示。 
![image](https://files.catbox.moe/p0sxra.png)

官网对图下面几点说明： 
1) 不同的Spark应用程序对应该不同的Executor，这些Executor在整个应用程序执行期间都存在并且Executor中可以采用多线程的方式执行Task。这样做的好处是，各个Spark应用程序的执行是相互隔离的。除Spark应用程序向外部存储系统写数据进行数据交互这种方式外，各Spark应用程序间无法进行数据共享。 
2) Spark对于其使用的集群资源管理器没有感知能力，只要它能对Executor进行申请并通信即可。这意味着不管使用哪种资源管理器，其执行流程都是不变的。这样Spark可以不同的资源管理器进行交互。 
3) Spark应用程序在整个执行过程中要与Executors进行来回通信。 
4) Driver端负责Spark应用程序任务的调度，因此最好Driver应该靠近Worker节点。

Spark目前支持的集群管理器包括：<br>
- Standalone 
- Apache Mesos 
- Hadoop YARN

在提交Spark应用程序时，Spark支持下列几种Master URL

![image](https://files.catbox.moe/hpld6x.png)

有了前面的知识铺垫后，现在我们来说明一下Spark的创建过程，SparkContext创建部分核心源码如下：

```java
// We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
    // retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
    _heartbeatReceiver = env.rpcEnv.setupEndpoint(
      HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))

    // Create and start the scheduler
    //根据master及SparkContext对象创建TaskScheduler，返回SchedulerBackend及TaskScheduler
    val (sched, ts) = SparkContext.createTaskScheduler(this, master)
    _schedulerBackend = sched
    _taskScheduler = ts
    //根据SparkContext对象创建DAGScheduler
    _dagScheduler = new DAGScheduler(this)
    _heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)

    // start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler‘s
    // constructor
    _taskScheduler.start()

    _applicationId = _taskScheduler.applicationId()
    _applicationAttemptId = taskScheduler.applicationAttemptId()
    _conf.set("spark.app.id", _applicationId)
    _env.blockManager.initialize(_applicationId)
```

跳到createTaskScheduler方法，可以看到如下源码：

```java
/**
   * Create a task scheduler based on a given master URL.
   * Return a 2-tuple of the scheduler backend and the task scheduler.
   */
  private def createTaskScheduler(
      sc: SparkContext,
      master: String): (SchedulerBackend, TaskScheduler) = {
    // 正则表达式，用于匹配local[N] 和 local[*]
    val LOCAL_N_REGEX = """local\[([0-9]+|\*)\]""".r
    // 正则表达式，用于匹配local[N, maxRetries], maxRetries表示失败后的最大重复次数
    val LOCAL_N_FAILURES_REGEX = """local\[([0-9]+|\*)\s*,\s*([0-9]+)\]""".r
    //正则表达式，用于匹配local-cluster[N, cores, memory]，它是一种伪分布式模式
    val LOCAL_CLUSTER_REGEX = """local-cluster\[\s*([0-9]+)\s*,\s*([0-9]+)\s*,\s*([0-9]+)\s*]""".r
    // 正则表达式用于匹配 Spark Standalone集群运行模式
    val SPARK_REGEX = """spark://(.*)""".r
    // 正则表达式用于匹配 Mesos集群资源管理器运行模式匹配 mesos:// 或 zk:// url
    val MESOS_REGEX = """(mesos|zk)://.*""".r
    // 正则表达式和于匹配Spark in MapReduce v1，用于兼容老版本的Hadoop集群
    val SIMR_REGEX = """simr://(.*)""".r

    // When running locally, don‘t try to re-execute tasks on failure.
    val MAX_LOCAL_TASK_FAILURES = 1

    master match {
      //本地单线程运行
      case "local" =>
        //TaskShceduler采用TaskSchedulerImpl
        //资源调度采用LocalBackend
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, 1)
        scheduler.initialize(backend)
        (backend, scheduler)
      //匹配本地多线程运行模式,匹配local[N]和Local[*]
      case LOCAL_N_REGEX(threads) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*] estimates the number of cores on the machine; local[N] uses exactly N threads.
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        if (threadCount <= 0) {
          throw new SparkException(s"Asked to run locally with $threadCount threads")
        }
        //TaskShceduler采用TaskSchedulerImpl
        //资源调度采用LocalBackend
        val scheduler = new TaskSchedulerImpl(sc, MAX_LOCAL_TASK_FAILURES, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)
      //匹配local[*, M]和local[N, M] 
      case LOCAL_N_FAILURES_REGEX(threads, maxFailures) =>
        def localCpuCount: Int = Runtime.getRuntime.availableProcessors()
        // local[*, M] means the number of cores on the computer with M failures
        // local[N, M] means exactly N threads with M failures
        val threadCount = if (threads == "*") localCpuCount else threads.toInt
        //TaskShceduler采用TaskSchedulerImpl
        //资源调度采用LocalBackend
        val scheduler = new TaskSchedulerImpl(sc, maxFailures.toInt, isLocal = true)
        val backend = new LocalBackend(sc.getConf, scheduler, threadCount)
        scheduler.initialize(backend)
        (backend, scheduler)
      //匹配Spark Standalone运行模式
      case SPARK_REGEX(sparkUrl) =>
        //TaskShceduler采用TaskSchedulerImpl
        val scheduler = new TaskSchedulerImpl(sc)
        val masterUrls = sparkUrl.split(",").map("spark://" + _)
        //资源调度采用SparkDeploySchedulerBackend
        val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        (backend, scheduler)
      //匹配local-cluster运行模式
      case LOCAL_CLUSTER_REGEX(numSlaves, coresPerSlave, memoryPerSlave) =>
        // Check to make sure memory requested <= memoryPerSlave. Otherwise Spark will just hang.
        val memoryPerSlaveInt = memoryPerSlave.toInt
        if (sc.executorMemory > memoryPerSlaveInt) {
          throw new SparkException(
            "Asked to launch cluster with %d MB RAM / worker but requested %d MB/worker".format(
              memoryPerSlaveInt, sc.executorMemory))
        }
        //TaskShceduler采用TaskSchedulerImpl
        val scheduler = new TaskSchedulerImpl(sc)

        val localCluster = new LocalSparkCluster(
          numSlaves.toInt, coresPerSlave.toInt, memoryPerSlaveInt, sc.conf)
        val masterUrls = localCluster.start()
          //资源调度采用SparkDeploySchedulerBackend
        val backend = new SparkDeploySchedulerBackend(scheduler, sc, masterUrls)
        scheduler.initialize(backend)
        backend.shutdownCallback = (backend: SparkDeploySchedulerBackend) => {
          localCluster.stop()
        }
        (backend, scheduler)
      //"yarn-standalone"或"yarn-cluster"运行模式
      case "yarn-standalone" | "yarn-cluster" =>
        if (master == "yarn-standalone") {
          logWarning(
            "\"yarn-standalone\" is deprecated as of Spark 1.0. Use \"yarn-cluster\" instead.")
        }
        val scheduler = try {
        //TaskShceduler采用YarnClusterScheduler
          val clazz = Utils.classForName("org.apache.spark.scheduler.cluster.YarnClusterScheduler")
          val cons = clazz.getConstructor(classOf[SparkContext])
          cons.newInstance(sc).asInstanceOf[TaskSchedulerImpl]
        } catch {
          // TODO: Enumerate the exact reasons why it can fail
          // But irrespective of it, it means we cannot proceed !
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }
        val backend = try {
        //资源调度采用YarnClusterSchedulerBackend
          val clazz =
            Utils.classForName("org.apache.spark.scheduler.cluster.YarnClusterSchedulerBackend")
          val cons = clazz.getConstructor(classOf[TaskSchedulerImpl], classOf[SparkContext])
          cons.newInstance(scheduler, sc).asInstanceOf[CoarseGrainedSchedulerBackend]
        } catch {
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }
        scheduler.initialize(backend)
        (backend, scheduler)
      //yarn-client运行模式
      case "yarn-client" =>
         //TaskShceduler采用YarnScheduler，YarnScheduler为TaskSchedulerImpl的子类
        org.apache.spark.scheduler.cluster.YarnScheduler
        val scheduler = try {
          val clazz = Utils.classForName("org.apache.spark.scheduler.cluster.YarnScheduler")
          val cons = clazz.getConstructor(classOf[SparkContext])
          cons.newInstance(sc).asInstanceOf[TaskSchedulerImpl]

        } catch {
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }
        //资源采用org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend
        val backend = try {
          val clazz =
            Utils.classForName("org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend")
          val cons = clazz.getConstructor(classOf[TaskSchedulerImpl], classOf[SparkContext])
          cons.newInstance(scheduler, sc).asInstanceOf[CoarseGrainedSchedulerBackend]
        } catch {
          case e: Exception => {
            throw new SparkException("YARN mode not available ?", e)
          }
        }

        scheduler.initialize(backend)
        (backend, scheduler)
      //Mesos运行模式
      case mesosUrl @ MESOS_REGEX(_) =>
        MesosNativeLibrary.load()
        //TaskScheduler采用TaskSchedulerImpl
        val scheduler = new TaskSchedulerImpl(sc)
        val coarseGrained = sc.conf.getBoolean("spark.mesos.coarse", false)
        val url = mesosUrl.stripPrefix("mesos://") // strip scheme from raw Mesos URLs
        //根据coarseGrained选择粗粒度还是细粒度
        val backend = if (coarseGrained) {
          //精粒度资源调度CoarseMesosSchedulerBackend
          new CoarseMesosSchedulerBackend(scheduler, sc, url, sc.env.securityManager)
        } else {
          //细粒度资源调度MesosSchedulerBackend
          new MesosSchedulerBackend(scheduler, sc, url)
        }
        scheduler.initialize(backend)
        (backend, scheduler)

      //Spark IN MapReduce V1运行模式
      case SIMR_REGEX(simrUrl) =>
        //TaskScheduler采用TaskSchedulerImpl
        val scheduler = new TaskSchedulerImpl(sc)
         //资源调度采用SimrSchedulerBackend
        val backend = new SimrSchedulerBackend(scheduler, sc, simrUrl)
        scheduler.initialize(backend)
        (backend, scheduler)

      case _ =>
        throw new SparkException("Could not parse Master URL: ‘" + master + "‘")
    }
  }
}
```

资源调度SchedulerBackend类及相关子类如下图<br>
![image](https://files.catbox.moe/8f4je3.png)


任务调度器，TaskScheduler类及其子数如下图：<br>
![image](https://files.catbox.moe/kqdc1q.png)