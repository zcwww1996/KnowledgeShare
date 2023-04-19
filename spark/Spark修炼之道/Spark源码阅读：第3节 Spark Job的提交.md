[TOC]
前一我们分析了SparkContext的创建，这一节，我们介绍在RDD执行的时候，如何提交job进行分析，同样是下面的源码：

来源： https://blog.csdn.net/lovehuangjiaju/article/details/49256603


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

上面的程序代码counts.saveAsTextFile(“file:///hadoopLearning/spark-1.5.1-bin-hadoop2.4/countReslut.txt”)会触发action操作，Spark会生成一个Job来执行相关计算


```java
//将RDD保存为Hadoop支持的文件系统，包括本地文件、HDFS等，使用的是Hadoop的OutputFormat类
/**
   * Output the RDD to any Hadoop-supported file system, using a Hadoop `OutputFormat` class
   * supporting the key and value types K and V in this RDD.
   */
  def saveAsHadoopFile(
      path: String,
      keyClass: Class[_],
      valueClass: Class[_],
      outputFormatClass: Class[_ <: OutputFormat[_, _]],
      conf: JobConf = new JobConf(self.context.hadoopConfiguration),
      codec: Option[Class[_ <: CompressionCodec]] = None): Unit = self.withScope {
    // Rename this as hadoopConf internally to avoid shadowing (see SPARK-2038).
    // hadoop配置信息
    val hadoopConf = conf
    hadoopConf.setOutputKeyClass(keyClass)
    hadoopConf.setOutputValueClass(valueClass)
    // Doesn't work in Scala 2.9 due to what may be a generics bug
    // TODO: Should we uncomment this for Scala 2.10?
    // conf.setOutputFormat(outputFormatClass)
    hadoopConf.set("mapred.output.format.class", outputFormatClass.getName)
    for (c <- codec) {
      hadoopConf.setCompressMapOutput(true)
      hadoopConf.set("mapred.output.compress", "true")
      hadoopConf.setMapOutputCompressorClass(c)
      hadoopConf.set("mapred.output.compression.codec", c.getCanonicalName)
      hadoopConf.set("mapred.output.compression.type", CompressionType.BLOCK.toString)
    }

    // Use configured output committer if already set
    if (conf.getOutputCommitter == null) {
      hadoopConf.setOutputCommitter(classOf[FileOutputCommitter])
    }

    FileOutputFormat.setOutputPath(hadoopConf,
      SparkHadoopWriter.createPathFromString(path, hadoopConf))
    //调用saveAsHadoopDataset方法进行RDD保存
    saveAsHadoopDataset(hadoopConf)
  }
```

跳转到saveAsHadoopDataset，并调用其self.context.runJob即SparkContext中的runJob方法

```java
/**
   * Output the RDD to any Hadoop-supported storage system, using a Hadoop JobConf object for
   * that storage system. The JobConf should set an OutputFormat and any output paths required
   * (e.g. a table name to write to) in the same way as it would be configured for a Hadoop
   * MapReduce job.
   */
  def saveAsHadoopDataset(conf: JobConf): Unit = self.withScope {
    // Rename this as hadoopConf internally to avoid shadowing (see SPARK-2038).
    val hadoopConf = conf
    val wrappedConf = new SerializableConfiguration(hadoopConf)
    val outputFormatInstance = hadoopConf.getOutputFormat
    val keyClass = hadoopConf.getOutputKeyClass
    val valueClass = hadoopConf.getOutputValueClass
    if (outputFormatInstance == null) {
      throw new SparkException("Output format class not set")
    }
    if (keyClass == null) {
      throw new SparkException("Output key class not set")
    }
    if (valueClass == null) {
      throw new SparkException("Output value class not set")
    }
    SparkHadoopUtil.get.addCredentials(hadoopConf)

    logDebug("Saving as hadoop file of type (" + keyClass.getSimpleName + ", " +
      valueClass.getSimpleName + ")")

    if (isOutputSpecValidationEnabled) {
      // FileOutputFormat ignores the filesystem parameter
      val ignoredFs = FileSystem.get(hadoopConf)
      hadoopConf.getOutputFormat.checkOutputSpecs(ignoredFs, hadoopConf)
    }

    val writer = new SparkHadoopWriter(hadoopConf)
    writer.preSetup()

    val writeToFile = (context: TaskContext, iter: Iterator[(K, V)]) => {
      val config = wrappedConf.value
      // Hadoop wants a 32-bit task attempt ID, so if ours is bigger than Int.MaxValue, roll it
      // around by taking a mod. We expect that no task will be attempted 2 billion times.
      val taskAttemptId = (context.taskAttemptId % Int.MaxValue).toInt

      val (outputMetrics, bytesWrittenCallback) = initHadoopOutputMetrics(context)

      writer.setup(context.stageId, context.partitionId, taskAttemptId)
      writer.open()
      var recordsWritten = 0L

      Utils.tryWithSafeFinally {
        while (iter.hasNext) {
          val record = iter.next()
          writer.write(record._1.asInstanceOf[AnyRef], record._2.asInstanceOf[AnyRef])

          // Update bytes written metric every few records
          maybeUpdateOutputMetrics(bytesWrittenCallback, outputMetrics, recordsWritten)
          recordsWritten += 1
        }
      } {
        writer.close()
      }
      writer.commit()
      bytesWrittenCallback.foreach { fn => outputMetrics.setBytesWritten(fn()) }
      outputMetrics.setRecordsWritten(recordsWritten)
    }
    //调用runJob方法执行RDD计算
    self.context.runJob(self, writeToFile)
    writer.commitJob()
  }
```

SparkContext中的runJob方法，该方法中再调用DAGScheduler中的runJob方法，具体源码如下：

```java
//SparkContext中的runJob方法
/**
   * Run a function on a given set of partitions in an RDD and pass the results to the given
   * handler function. This is the main entry point for all actions in Spark.
   */
  def runJob[T, U: ClassTag](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
      throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
      logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    //调用dagScheduler的runJob方法
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
  }
```

DAGScheduler中的runJob方法，该方法中通过submitJob方法进行任务的提交，具体源码如下：

```java
//DAGScheduler中的runJob方法
def runJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): Unit = {
    val start = System.nanoTime
    //调用DAGScheduler中的submitJob方法，返回JobWaiter对象，该对象等待job完成，完成后调用resultHandler函数进行后续处理
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    waiter.awaitResult() match {
      //处理成功
      case JobSucceeded =>
        logInfo("Job %d finished: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
      //处理失败
      case JobFailed(exception: Exception) =>
        logInfo("Job %d failed: %s, took %f s".format
          (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
  }
```

submitJob方法的具体源码如下：


```java
//DAGScheduler中的submitJob方法
  /**
   * Submit a job to the job scheduler and get a JobWaiter object back. The JobWaiter object
   * can be used to block until the the job finishes executing or can be used to cancel the job.
   */
  def submitJob[T, U](
      rdd: RDD[T],
      func: (TaskContext, Iterator[T]) => U,
      partitions: Seq[Int],
      callSite: CallSite,
      resultHandler: (Int, U) => Unit,
      properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    val maxPartitions = rdd.partitions.length
    partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
      throw new IllegalArgumentException(
        "Attempting to access a non-existent partition: " + p + ". " +
          "Total number of partitions: " + maxPartitions)
    }

    val jobId = nextJobId.getAndIncrement()
    if (partitions.size == 0) {
      return new JobWaiter[U](this, jobId, 0, resultHandler)
    }

    assert(partitions.size > 0)
    val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
    //创建JobWaiter对象
    val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
    //类型为DAGSchedulerEventProcessLoop的对象eventProcessLoop，将任务提交JobSubmitted放置在event队列当中，eventThread后台线程将对该任务提交进行处理
    eventProcessLoop.post(JobSubmitted(
      jobId, rdd, func2, partitions.toArray, callSite, waiter,
      SerializationUtils.clone(properties)))
    waiter
  }
```

eventProcessLoop.post(JobSubmitted( 
jobId, rdd, func2, partitions.toArray, callSite, waiter, 
SerializationUtils.clone(properties)))将任务提交JobSubmitted放置在event队列当中，eventThread后台线程将对该任务提交进行处理，该eventThread被定义在DAGSchedulerEventProcessLoop的父类EventLoop当中，其源码如下：


```java
/**
 * An event loop to receive events from the caller and process all events in the event thread. It
 * will start an exclusive event thread to process all events.
 *
 * Note: The event queue will grow indefinitely. So subclasses should make sure `onReceive` can
 * handle events in time to avoid the potential OOM.
 */
private[spark] abstract class EventLoop[E](name: String) extends Logging {

  private val eventQueue: BlockingQueue[E] = new LinkedBlockingDeque[E]()

  private val stopped = new AtomicBoolean(false)

  //处理envent队列的后台线程
  private val eventThread = new Thread(name) {
    setDaemon(true)

    override def run(): Unit = {
      try {
        while (!stopped.get) {
          //获取事件处理
          val event = eventQueue.take()
          try {
            //调用onReceive方法，该方法在EventLoop中是抽象方法，由子类实现
            onReceive(event)
          } catch {
            case NonFatal(e) => {
              try {
                onError(e)
              } catch {
                case NonFatal(e) => logError("Unexpected error in " + name, e)
              }
            }
          }
        }
      } catch {
        case ie: InterruptedException => // exit even if eventQueue is not empty
        case NonFatal(e) => logError("Unexpected error in " + name, e)
      }
    }

  }

  def start(): Unit = {
    if (stopped.get) {
      throw new IllegalStateException(name + " has already been stopped")
    }
    // Call onStart before starting the event thread to make sure it happens before onReceive
    onStart()
    eventThread.start()
  }

  def stop(): Unit = {
    if (stopped.compareAndSet(false, true)) {
      eventThread.interrupt()
      var onStopCalled = false
      try {
        eventThread.join()
        // Call onStop after the event thread exits to make sure onReceive happens before onStop
        onStopCalled = true
        onStop()
      } catch {
        case ie: InterruptedException =>
          Thread.currentThread().interrupt()
          if (!onStopCalled) {
            // ie is thrown from `eventThread.join()`. Otherwise, we should not call `onStop` since
            // it's already called.
            onStop()
          }
      }
    } else {
      // Keep quiet to allow calling `stop` multiple times.
    }
  }

  //post方法，将事件放置到event 队列，由event thread处理
  /**
   * Put the event into the event queue. The event thread will process it later.
   */
  def post(event: E): Unit = {
    eventQueue.put(event)
  }

  //略去其它方法
```

org.apache.spark.scheduler.DAGScheduler.scala文件中定义了DAGSchedulerEventProcessLoop，该类继承了EventLoop，对onReceive方法进行了实现，具体源码如下：

```java
private[scheduler] class DAGSchedulerEventProcessLoop(dagScheduler: DAGScheduler)
  extends EventLoop[DAGSchedulerEvent]("dag-scheduler-event-loop") with Logging {

  private[this] val timer = dagScheduler.metricsSource.messageProcessingTimer

  /**
   * The main event loop of the DAG scheduler.
   */
  //onReceive方法
  override def onReceive(event: DAGSchedulerEvent): Unit = {
    val timerContext = timer.time()
    try {
      //调用doOnReceive方法
      doOnReceive(event)
    } finally {
      timerContext.stop()
    }
  }
  //doOnReceive方法的具体实现
  private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    //处理JobSubmitted事件
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      //调用dagScheduler的handleJobSubmitted方法
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)
    //处理其它相关事件
    case StageCancelled(stageId) =>
      dagScheduler.handleStageCancellation(stageId)

    case JobCancelled(jobId) =>
      dagScheduler.handleJobCancellation(jobId)

    case JobGroupCancelled(groupId) =>
      dagScheduler.handleJobGroupCancelled(groupId)

    case AllJobsCancelled =>
      dagScheduler.doCancelAllJobs()

    case ExecutorAdded(execId, host) =>
      dagScheduler.handleExecutorAdded(execId, host)

    case ExecutorLost(execId) =>
      dagScheduler.handleExecutorLost(execId, fetchFailed = false)

    case BeginEvent(task, taskInfo) =>
      dagScheduler.handleBeginEvent(task, taskInfo)

    case GettingResultEvent(taskInfo) =>
      dagScheduler.handleGetTaskResult(taskInfo)

    case completion @ CompletionEvent(task, reason, _, _, taskInfo, taskMetrics) =>
      dagScheduler.handleTaskCompletion(completion)

    case TaskSetFailed(taskSet, reason, exception) =>
      dagScheduler.handleTaskSetFailed(taskSet, reason, exception)

    case ResubmitFailedStages =>
      dagScheduler.resubmitFailedStages()
  }

  override def onError(e: Throwable): Unit = {
    logError("DAGSchedulerEventProcessLoop failed; shutting down SparkContext", e)
    try {
      dagScheduler.doCancelAllJobs()
    } catch {
      case t: Throwable => logError("DAGScheduler failed to cancel all jobs.", t)
    }
    dagScheduler.sc.stop()
  }

  override def onStop(): Unit = {
    // Cancel any active jobs in postStop hook
    dagScheduler.cleanUpAfterSchedulerStop()
  }
}
```

从上面的代码可以看到，最后调用的dagScheduler.handleJobSubmitted方法完成整个job的提交。后面便是如何将job划分成各个Stage及TaskSet，提交到各个Worker节点执行，这部分内容，我们在下一节中进行讲解