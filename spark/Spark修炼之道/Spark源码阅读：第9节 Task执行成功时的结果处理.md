[TOC]

来源： https://blog.csdn.net/lovehuangjiaju/article/details/49490341

在上一节中，给出了Task在Executor上的运行代码演示，我们知道代码的最终运行通过的是TaskRunner方法

```scala
class TaskRunner(
      execBackend: ExecutorBackend,
      val taskId: Long,
      val attemptNumber: Int,
      taskName: String,
      serializedTask: ByteBuffer)
    extends Runnable {

    //其它无关代码省略

      //向Driver端发状态更新
      execBackend.statusUpdate(taskId, TaskState.RUNNING, EMPTY_BYTE_BUFFER)

      //其它非关键代码省略
      //执行完成后，通知Driver端进行状态更新
        execBackend.statusUpdate(taskId, TaskState.FINISHED, serializedResult)

      } catch {
        //出错时，通知Driver端的状态更新
        //代码省略
  }
```

状态更新时，先调用的是CoarseGrainedExecutorBackend中的statusUpdate方法


```scala
 override def statusUpdate(taskId: Long, state: TaskState, data: ByteBuffer) {
    val msg = StatusUpdate(executorId, taskId, state, data)
    driver match {
      //将Driver端发送StatusUpdate消息
      case Some(driverRef) => driverRef.send(msg)
      case None => logWarning(s"Drop $msg because has not yet connected to driver")
    }
  }
}
```

DriverEndpoint中的receive方法接收并处理发送过来的StatusUpdate消息，具体源码如下：

```scala
 override def receive: PartialFunction[Any, Unit] = {
      //接收StatusUpdate发送过来的消息
      case StatusUpdate(executorId, taskId, state, data) =>
        //调用TaskSchedulerImpl中的statusUpdate方法
        scheduler.statusUpdate(taskId, state, data.value)

        //
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

TaskSchedulerImpl中的statusUpdate方法源码如下：


```scala
 def statusUpdate(tid: Long, state: TaskState, serializedData: ByteBuffer) {
    var failedExecutor: Option[String] = None
    synchronized {
      try {
        if (state == TaskState.LOST && taskIdToExecutorId.contains(tid)) {
          // We lost this entire executor, so remember that it's gone
          val execId = taskIdToExecutorId(tid)
          if (activeExecutorIds.contains(execId)) {
            removeExecutor(execId)
            failedExecutor = Some(execId)
          }
        }
        taskIdToTaskSetManager.get(tid) match {
          case Some(taskSet) =>
            if (TaskState.isFinished(state)) {
              taskIdToTaskSetManager.remove(tid)
              taskIdToExecutorId.remove(tid)
            }
            //任务执行成功时的处理
            if (state == TaskState.FINISHED) {
              taskSet.removeRunningTask(tid)
             //taskResultGetter为线程池，处理执行成功的情况
             taskResultGetter.enqueueSuccessfulTask(taskSet, tid, serializedData)
            //任务执行不成功，包括任务执行失败、任务丢失及任务被杀死
            } else if (Set(TaskState.FAILED, TaskState.KILLED, TaskState.LOST).contains(state)) {
              taskSet.removeRunningTask(tid)
              //处理任务执行失败的情况
              taskResultGetter.enqueueFailedTask(taskSet, tid, state, serializedData)
            }
          case None =>
            logError(
              ("Ignoring update with state %s for TID %s because its task set is gone (this is " +
                "likely the result of receiving duplicate task finished status updates)")
                .format(state, tid))
        }
      } catch {
        case e: Exception => logError("Exception in statusUpdate", e)
      }
    }
    // Update the DAGScheduler without holding a lock on this, since that can deadlock
    if (failedExecutor.isDefined) {
      dagScheduler.executorLost(failedExecutor.get)
      backend.reviveOffers()
    }
  }
```

对于Task执行成功的情况，它会调用TaskResultGetter的enqueueSuccessfulTask方法进行处理：

```scala
 def enqueueSuccessfulTask(
    taskSetManager: TaskSetManager, tid: Long, serializedData: ByteBuffer) {
    getTaskResultExecutor.execute(new Runnable {
      override def run(): Unit = Utils.logUncaughtExceptions {
        try {
          val (result, size) = serializer.get().deserialize[TaskResult[_]](serializedData) match {
             //结果为最终的计算结果
            case directResult: DirectTaskResult[_] =>
              if (!taskSetManager.canFetchMoreResults(serializedData.limit())) {
                return
              }
              // deserialize "value" without holding any lock so that it won't block other threads.
              // We should call it here, so that when it's called again in
              // "TaskSetManager.handleSuccessfulTask", it does not need to deserialize the value.
              directResult.value()
              (directResult, serializedData.limit())
            //结果保存在远程Worker节点的BlockManager当中
            case IndirectTaskResult(blockId, size) =>
              if (!taskSetManager.canFetchMoreResults(size)) {
                // dropped by executor if size is larger than maxResultSize
                sparkEnv.blockManager.master.removeBlock(blockId)
                return
              }
              logDebug("Fetching indirect task result for TID %s".format(tid))
              scheduler.handleTaskGettingResult(taskSetManager, tid)
              //从远程Worker获取结果
              val serializedTaskResult = sparkEnv.blockManager.getRemoteBytes(blockId)
              if (!serializedTaskResult.isDefined) {
                /* We won't be able to get the task result if the machine that ran the task failed
                 * between when the task ended and when we tried to fetch the result, or if the
                 * block manager had to flush the result. */
                 //获取结果时，如果远程Eexecutor对应的机器出现故障或其它错误时，可能导致结果获取失败
                scheduler.handleFailedTask(
                  taskSetManager, tid, TaskState.FINISHED, TaskResultLost)
                return
              }
              //反序列化远程获取的结果
              val deserializedResult = serializer.get().deserialize[DirectTaskResult[_]](
                serializedTaskResult.get)
             //删除远程结果sparkEnv.blockManager.master.removeBlock(blockId)
              (deserializedResult, size)
          }

          result.metrics.setResultSize(size)
          //TaskSchedulerImpl处理获取到的结果
          scheduler.handleSuccessfulTask(taskSetManager, tid, result)
        } catch {
          case cnf: ClassNotFoundException =>
            val loader = Thread.currentThread.getContextClassLoader
            taskSetManager.abort("ClassNotFound with classloader: " + loader)
          // Matching NonFatal so we don't catch the ControlThrowable from the "return" above.
          case NonFatal(ex) =>
            logError("Exception while getting task result", ex)
            taskSetManager.abort("Exception while getting task result: %s".format(ex))
        }
      }
    })
  }
```

TaskSchedulerImpl中的handleSuccessfulTask方法将最终对计算结果进行处理，具有源码如下：

```scala
def handleSuccessfulTask(
      taskSetManager: TaskSetManager,
      tid: Long,
      taskResult: DirectTaskResult[_]): Unit = synchronized {
     //调用TaskSetManager.handleSuccessfulTask方法进行处理
    taskSetManager.handleSuccessfulTask(tid, taskResult)
  }
```

TaskSetManager.handleSuccessfulTask方法源码如下：

```scala
/**
   * Marks the task as successful and notifies the DAGScheduler that a task has ended.
   */
  def handleSuccessfulTask(tid: Long, result: DirectTaskResult[_]): Unit = {
    val info = taskInfos(tid)
    val index = info.index
    info.markSuccessful()
    removeRunningTask(tid)
    // This method is called by "TaskSchedulerImpl.handleSuccessfulTask" which holds the
    // "TaskSchedulerImpl" lock until exiting. To avoid the SPARK-7655 issue, we should not
    // "deserialize" the value when holding a lock to avoid blocking other threads. So we call
    // "result.value()" in "TaskResultGetter.enqueueSuccessfulTask" before reaching here.
    // Note: "result.value()" only deserializes the value when it's called at the first time, so
    // here "result.value()" just returns the value and won't block other threads.
    //调用DagScheduler的taskEnded方法
    sched.dagScheduler.taskEnded(
      tasks(index), Success, result.value(), result.accumUpdates, info, result.metrics)
    if (!successful(index)) {
      tasksSuccessful += 1
      logInfo("Finished task %s in stage %s (TID %d) in %d ms on %s (%d/%d)".format(
        info.id, taskSet.id, info.taskId, info.duration, info.host, tasksSuccessful, numTasks))
      // Mark successful and stop if all the tasks have succeeded.
      successful(index) = true
      if (tasksSuccessful == numTasks) {
        isZombie = true
      }
    } else {
      logInfo("Ignoring task-finished event for " + info.id + " in stage " + taskSet.id +
        " because task " + index + " has already completed successfully")
    }
    failedExecutors.remove(index)
    maybeFinishTaskSet()
  }
```

进入DAGScheduler的taskEnded方法

```scala
//DAGScheduler中的taskEnded方法
/**
   * Called by the TaskSetManager to report task completions or failures.
   */
  def taskEnded(
      task: Task[_],
      reason: TaskEndReason,
      result: Any,
      accumUpdates: Map[Long, Any],
      taskInfo: TaskInfo,
      taskMetrics: TaskMetrics): Unit = {
      //调用DAGSchedulerEventProcessLoop的post方法将CompletionEvent提交到事件队列中，交由eventThread进行处理，onReceive方法将处理该事件
    eventProcessLoop.post(
      CompletionEvent(task, reason, result, accumUpdates, taskInfo, taskMetrics))
  }
```

跳转到onReceive方法当中，可以看到其调用的是onReceive

```scala
//DAGSchedulerEventProcessLoop中的onReceive方法
/**
   * The main event loop of the DAG scheduler.
   */
  override def onReceive(event: DAGSchedulerEvent): Unit = {
    val timerContext = timer.time()
    try {
      doOnReceive(event)
    } finally {
      timerContext.stop()
    }
  }
```

跳转到doOnReceive方法到当中，可以看到

```scala
//DAGSchedulerEventProcessLoop中的doOnReceive方法
private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
      dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)

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
    //处理CompletionEvent事件
    case completion @ CompletionEvent(task, reason, _, _, taskInfo, taskMetrics) =>
    //交由DAGScheduler.handleTaskCompletion方法处理
      dagScheduler.handleTaskCompletion(completion)

    case TaskSetFailed(taskSet, reason, exception) =>
      dagScheduler.handleTaskSetFailed(taskSet, reason, exception)

    case ResubmitFailedStages =>
      dagScheduler.resubmitFailedStages()
  }
```

DAGScheduler.handleTaskCompletion方法完成计算结果的处理

```scala
/**
   * Responds to a task finishing. This is called inside the event loop so it assumes that it can
   * modify the scheduler's internal state. Use taskEnded() to post a task end event from outside.
   */
  private[scheduler] def handleTaskCompletion(event: CompletionEvent) {
    val task = event.task
    val stageId = task.stageId
    val taskType = Utils.getFormattedClassName(task)

    outputCommitCoordinator.taskCompleted(stageId, task.partitionId,
      event.taskInfo.attempt, event.reason)

    // The success case is dealt with separately below, since we need to compute accumulator
    // updates before posting.
    if (event.reason != Success) {
      val attemptId = task.stageAttemptId
      listenerBus.post(SparkListenerTaskEnd(stageId, attemptId, taskType, event.reason,
        event.taskInfo, event.taskMetrics))
    }

    if (!stageIdToStage.contains(task.stageId)) {
      // Skip all the actions if the stage has been cancelled.
      return
    }

    val stage = stageIdToStage(task.stageId)
    event.reason match {
      case Success =>
        listenerBus.post(SparkListenerTaskEnd(stageId, stage.latestInfo.attemptId, taskType,
          event.reason, event.taskInfo, event.taskMetrics))
        stage.pendingTasks -= task
        task match {
           //处理ResultTask
          case rt: ResultTask[_, _] =>
            // Cast to ResultStage here because it's part of the ResultTask
            // TODO Refactor this out to a function that accepts a ResultStage
            val resultStage = stage.asInstanceOf[ResultStage]
            resultStage.resultOfJob match {
              case Some(job) =>
                if (!job.finished(rt.outputId)) {
                  updateAccumulators(event)
                  job.finished(rt.outputId) = true
                  job.numFinished += 1
                  // If the whole job has finished, remove it
                  //判断job是否已处理完毕，即所有Task是否处理完毕
                  if (job.numFinished == job.numPartitions) {
                    markStageAsFinished(resultStage)
                    cleanupStateForJobAndIndependentStages(job)
                    listenerBus.post(
                      SparkListenerJobEnd(job.jobId, clock.getTimeMillis(), JobSucceeded))
                  }

                  // taskSucceeded runs some user code that might throw an exception. Make sure
                  // we are resilient against that.
                  //通知JobWaiter，job处理完毕
                  try {
                    job.listener.taskSucceeded(rt.outputId, event.result)
                  } catch {
                    case e: Exception =>
                      // TODO: Perhaps we want to mark the resultStage as failed?
                      job.listener.jobFailed(new SparkDriverExecutionException(e))
                  }
                }
              case None =>
                logInfo("Ignoring result from " + rt + " because its job has finished")
            }
          //处理ShuffleMapTask
          case smt: ShuffleMapTask =>
            val shuffleStage = stage.asInstanceOf[ShuffleMapStage]
            updateAccumulators(event)
            val status = event.result.asInstanceOf[MapStatus]
            val execId = status.location.executorId
            logDebug("ShuffleMapTask finished on " + execId)
            if (failedEpoch.contains(execId) && smt.epoch <= failedEpoch(execId)) {
              logInfo(s"Ignoring possibly bogus $smt completion from executor $execId")
            } else {
               //结果保存到ShuffleMapStage
              shuffleStage.addOutputLoc(smt.partitionId, status)
            }

            if (runningStages.contains(shuffleStage) && shuffleStage.pendingTasks.isEmpty) {
              markStageAsFinished(shuffleStage)
              logInfo("looking for newly runnable stages")
              logInfo("running: " + runningStages)
              logInfo("waiting: " + waitingStages)
              logInfo("failed: " + failedStages)

              // We supply true to increment the epoch number here in case this is a
              // recomputation of the map outputs. In that case, some nodes may have cached
              // locations with holes (from when we detected the error) and will need the
              // epoch incremented to refetch them.
              // TODO: Only increment the epoch number if this is not the first time
              //       we registered these map outputs.
              mapOutputTracker.registerMapOutputs(
                shuffleStage.shuffleDep.shuffleId,
                shuffleStage.outputLocs.map(list => if (list.isEmpty) null else list.head),
                changeEpoch = true)

              clearCacheLocs()
              //处理部分Task失败的情况
              if (shuffleStage.outputLocs.contains(Nil)) {
                // Some tasks had failed; let's resubmit this shuffleStage
                // TODO: Lower-level scheduler should also deal with this
                logInfo("Resubmitting " + shuffleStage + " (" + shuffleStage.name +
                  ") because some of its tasks had failed: " +
                  shuffleStage.outputLocs.zipWithIndex.filter(_._1.isEmpty)
                      .map(_._2).mkString(", "))
                 //重新提交
                submitStage(shuffleStage)
              } else {
                //处理其它未提交的Stage
                val newlyRunnable = new ArrayBuffer[Stage]
                for (shuffleStage <- waitingStages) {
                  logInfo("Missing parents for " + shuffleStage + ": " +
                    getMissingParentStages(shuffleStage))
                }
                for (shuffleStage <- waitingStages if getMissingParentStages(shuffleStage).isEmpty)
                {
                  newlyRunnable += shuffleStage
                }
                waitingStages --= newlyRunnable
                runningStages ++= newlyRunnable
                for {
                  shuffleStage <- newlyRunnable.sortBy(_.id)
                  jobId <- activeJobForStage(shuffleStage)
                } {
                  logInfo("Submitting " + shuffleStage + " (" +
                    shuffleStage.rdd + "), which is now runnable")
                  submitMissingTasks(shuffleStage, jobId)
                }
              }
            }
          }

     //其它代码省略
  }
```

执行流程： 
1. org.apache.spark.executor.TaskRunner.statusUpdate方法 
2. org.apache.spark.executor.CoarseGrainedExecutorBackend.statusUpdate方法 
3. org.apache.spark.scheduler.cluster.CoarseGrainedSchedulerBackend#DriverEndpoint.recieve方法，DriverEndPoint是内部类 
4. org.apache.spark.scheduler.TaskSchedulerImpl中的statusUpdate方法 
5. org.apache.spark.scheduler.TaskResultGetter.enqueueSuccessfulTask方法 
6. org.apache.spark.scheduler.DAGScheduler.handleTaskCompletion方法