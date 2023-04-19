[TOC]

注：本文使用的源码版本为flume-1.7.0
链接：https://www.jianshu.com/p/4f43780c82e9

一般使用hdfs sink都会采用滚动生成文件的方式，hdfs sink滚动生成文件的策略有：

*   基于时间
*   基于文件大小
*   基于hdfs文件副本数（一般要规避这种情况）
*   基于event数量
*   基于文件闲置时间

下面将详细讲解这些策略的配置以及原理

# 基于时间策略

配置项：hdfs.rollInterval
 默认值：30秒
 说明：如果设置为0表示禁用这个策略
 原理：
 在`org.apache.flume.sink.hdfs.BucketWriter.append`方法中打开一个文件，都会调用`open`方法，如果设置了hdfs.rollInterval，那么hdfs.rollInterval秒之内只要其他策略没有关闭文件，文件会在hdfs.rollInterval秒之后关闭。

```java
// if time-based rolling is enabled, schedule the roll
if (rollInterval > 0) {
  Callable<Void> action = new Callable<Void>() {
    public Void call() throws Exception {
      LOG.debug("Rolling file ({}): Roll scheduled after {} sec elapsed.",
          bucketPath, rollInterval);
      try {
        // Roll the file and remove reference from sfWriters map.
        close(true);
      } catch (Throwable t) {
        LOG.error("Unexpected error", t);
      }
      return null;
    }
  };
  timedRollFuture = timedRollerPool.schedule(action, rollInterval,
      TimeUnit.SECONDS);
}

```

# 基于文件大小和event数量策略

配置项：

1.  文件大小策略：hdfs.rollSize
2.  event数量策略：hdfs.rollCount

默认值：

1.  文件大小策略：1024字节
2.  event数量策略：10

说明：如果设置为0表示禁用这些策略
 原理：
 这2种策略都是在`org.apache.flume.sink.hdfs.BucketWriter.shouldRotate`方法中进行判断的，只要`doRotate`的值为true，那么当前文件就会关闭，即滚动到下一个文件。

```java
private boolean shouldRotate() {
  boolean doRotate = false;

//判断文件副本数
  if (writer.isUnderReplicated()) {
    this.isUnderReplicated = true;
    doRotate = true;
  } else {
    this.isUnderReplicated = false;
  }

//判断event数量
  if ((rollCount > 0) && (rollCount <= eventCounter)) {
    LOG.debug("rolling: rollCount: {}, events: {}", rollCount, eventCounter);
    doRotate = true;
  }

//判断文件大小
  if ((rollSize > 0) && (rollSize <= processSize)) {
    LOG.debug("rolling: rollSize: {}, bytes: {}", rollSize, processSize);
    doRotate = true;
  }

  return doRotate;
}

```

注意：如果同时配置了时间策略和文件大小策略，那么会先判断时间，如果时间没到再判断其他的条件。

# 基于hdfs文件副本数

配置项：hdfs.minBlockReplicas
 默认值：和hdfs的副本数一致
 原理：
 从上面的代码中可以看到，判断副本数的关键方法是`writer.isUnderReplicated()`，即

```java
public boolean isUnderReplicated() {
  try {
    int numBlocks = getNumCurrentReplicas();
    if (numBlocks == -1) {
      return false;
    }
    int desiredBlocks;
    if (configuredMinReplicas != null) {
      desiredBlocks = configuredMinReplicas;
    } else {
      desiredBlocks = getFsDesiredReplication();
    }
    return numBlocks < desiredBlocks;
  } catch (IllegalAccessException e) {
    logger.error("Unexpected error while checking replication factor", e);
  } catch (InvocationTargetException e) {
    logger.error("Unexpected error while checking replication factor", e);
  } catch (IllegalArgumentException e) {
    logger.error("Unexpected error while checking replication factor", e);
  }
  return false;
}

```

也就是说，如果当前正在写的文件的副本数小于hdfs.minBlockReplicas，此方法返回`true`，其他情况都返回`false`。**假设这个方法返回`true`**，那么看一下会发生什么事情。
 首先就是上面代码提到的`shouldRotate`方法肯定返回的是`true`。再继续跟踪，下面的代码是关键

```java
// check if it's time to rotate the file
if (shouldRotate()) {
  boolean doRotate = true;

  if (isUnderReplicated) {
    if (maxConsecUnderReplRotations > 0 &&
        consecutiveUnderReplRotateCount >= maxConsecUnderReplRotations) {
      doRotate = false;
      if (consecutiveUnderReplRotateCount == maxConsecUnderReplRotations) {
        LOG.error("Hit max consecutive under-replication rotations ({}); " +
            "will not continue rolling files under this path due to " +
            "under-replication", maxConsecUnderReplRotations);
      }
    } else {
      LOG.warn("Block Under-replication detected. Rotating file.");
    }
    consecutiveUnderReplRotateCount++;
  } else {
    consecutiveUnderReplRotateCount = 0;
  }

  if (doRotate) {
    close();
    open();
  }
}

```

这里`maxConsecUnderReplRotations`是固定的值30，也就是说，文件滚动生成了30个之后，就不会再滚动了，因为将`doRotate`设置为了`false`。所以，从这里可以看到，如果`isUnderReplicated`方法返回的是`true`，可能会导致文件的滚动和预期的不一致。规避这个问题的方法就是将hdfs.minBlockReplicas设置为1，一般hdfs的副本数肯定都是大于等于1的，所以`isUnderReplicated`方法一定会返回`false`。
 **所以一般情况下，要规避这种情况，避免影响文件的正常滚动。**

# 基于文件闲置时间策略

配置项：hdfs.idleTimeout
 默认值：0
 说明：默认启动这个功能
 这种策略很简单，如果文件在hdfs.idleTimeout秒的时间里都是闲置的，没有任何数据写入，那么当前文件关闭，滚动到下一个文件。

```java
public synchronized void flush() throws IOException, InterruptedException {
  checkAndThrowInterruptedException();
  if (!isBatchComplete()) {
    doFlush();

    if (idleTimeout > 0) {
      // if the future exists and couldn't be cancelled, that would mean it has already run
      // or been cancelled
      if (idleFuture == null || idleFuture.cancel(false)) {
        Callable<Void> idleAction = new Callable<Void>() {
          public Void call() throws Exception {
            LOG.info("Closing idle bucketWriter {} at {}", bucketPath,
                     System.currentTimeMillis());
            if (isOpen) {
              close(true);
            }
            return null;
          }
        };
        idleFuture = timedRollerPool.schedule(idleAction, idleTimeout,
            TimeUnit.SECONDS);
      }
    }
  }
}

```
