[TOC]
# 1.SparkStreaming
流式wordcount

启动netcat(nc)监听9999端口
```bash
1、 启动netcat
nc -lk -p 9999
2、 如果nc命令没有安装，yum安装nc
yum install -y nc
```

## 1.1 可以统计累计单词数目
```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}


object SparkStreaming02 {

  // 屏蔽日志
  Logger.getLogger("org.apache").setLevel(Level.WARN)

  val ckp = "./ckp"
  def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount =newValues.sum + runningCount.getOrElse(0)
    // add the new values with the previous running count to get the new count
    Some(newCount)
  }

  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setMaster("local[2]").setAppName("SparkStreaming02")
    val ssc = new StreamingContext(conf, Seconds(5))
    ssc.checkpoint(ckp)
    // 创建一个将要连接到 hostname:port 的 DStream，如 localhost:9999
    val lines = ssc.socketTextStream("192.168.134.114", 9999)
    // 缓存数据的时间间隔 默认是10s
    lines.checkpoint(Seconds(10))
    // 将每一行拆分成 words（单词）
    val words = lines.flatMap(_.split(" "))
    // 计算每一个 batch（批次）中的每一个 word（单词）
    val pairs = words.map(word => (word, 1))
    val wordCounts = pairs.updateStateByKey[Int](updateFunction _)

    // 在控制台打印出在这个离散流（DStream）中生成的每个 RDD 的前十个元素
    // 注意: 必需要触发 action（很多初学者会忘记触发 action 操作，导致报错：No output operations registered, so nothing to execute）
    wordCounts.print()
    ssc.start()             // 开始计算
    ssc.awaitTermination()  // 等待计算被中断
  }
}
```


## 1.2 可以统计累计单词数目、可以断点续传
```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}


object SparkStreaming02 {

  // 屏蔽日志
  Logger.getLogger("org.apache").setLevel(Level.WARN)

  val ckp = "./ckp"

  def updateFunction(newValues: Seq[Int], runningCount: Option[Int]): Option[Int] = {
    val newCount = newValues.sum + runningCount.getOrElse(0)
    // add the new values with the previous running count to get the new count
    Some(newCount)
  }

  // 如果从checkpoint目录中恢复不了上一个job的ssc实例，就创建一个新的ssc
  def functionToCreateContext(): StreamingContext = {
    println("create a new ssc")

    val conf = new SparkConf().setMaster("local[2]").setAppName("SparkStreaming02")
    val ssc = new StreamingContext(conf, Seconds(1))
    //设置checkpoint
    ssc.checkpoint(ckp)
    // 创建一个将要连接到 hostname:port 的 DStream，如 localhost:9999
    val lines = ssc.socketTextStream("192.168.134.114", 9999)
    
    // 缓存数据的时间间隔 默认是10s
    lines.checkpoint(Seconds(1))
    
    // 将每一行拆分成 words（单词）
    val words = lines.flatMap(_.split(" "))
    // 计算每一个 batch（批次）中的每一个 word（单词）
    val pairs = words.map(word => (word, 1))
    val wordCounts = pairs.updateStateByKey[Int](updateFunction _)

    // 在控制台打印出在这个离散流（DStream）中生成的每个 RDD 的前十个元素
    // 注意: 必需要触发 action（很多初学者会忘记触发 action 操作，导致报错：No output operations registered, so nothing to execute）
    wordCounts.print()
    ssc
  }

  def main(args: Array[String]): Unit = {
    val ssc = StreamingContext.getOrCreate(ckp, functionToCreateContext _)
    ssc.start() // 开始计算
    ssc.awaitTermination() // 等待计算被中断
  }
}
```

## 1.3 checkpoint、broadcast、accumulater
如果检查点跟广播变量、累加器一起使用，累加器和广播变量无法从 Spark Streaming中的检查点恢复，则必须为累加器和广播变量创建延迟实例化的单一实例，以便在驱动程序在失败时重新启动后可以重新实例化它们。

http://spark.apache.org/docs/latest/streaming-programming-guide.html#accumulators-broadcast-variables-and-checkpoints

**将broadcast和accumulater进行lazy、singleton操作**
[![dropped.jpg](https://s1.ax1x.com/2023/02/24/pSzfoT0.jpg)](https://pic.imgdb.cn/item/63f884adf144a010074ff107.jpg)

[完整代码](https://github.com/apache/spark/blob/v3.3.2/examples/src/main/scala/org/apache/spark/examples/streaming/RecoverableNetworkWordCount.scala)
```scala
import com.google.common.io.Files
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.rdd.RDD
import org.apache.spark.streaming.{Seconds, StreamingContext, Time}
import org.apache.spark.util.LongAccumulator
import org.apache.spark.{SparkConf, SparkContext}

import java.io.File
import java.nio.charset.Charset
import scala.util.Try

/**
 * Use this singleton to get or register a Broadcast variable.
 */
object WordExcludeList {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordExcludeList = Seq("a", "b", "c")
          instance = sc.broadcast(wordExcludeList)
        }
      }
    }
    instance
  }
}

/**
 * Use this singleton to get or register an Accumulator.
 */
object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("DroppedWordsCounter")
        }
      }
    }
    instance
  }
}

/**
 * Counts words in text encoded with UTF8 received from the network every second. This example also
 * shows how to use lazily instantiated singleton instances for Accumulator and Broadcast so that
 * they can be registered on driver failures.
 *
 * Usage: RecoverableNetworkWordCount <hostname> <port> <checkpoint-directory> <output-file>
 * <hostname> and <port> describe the TCP server that Spark Streaming would connect to receive
 * data. <checkpoint-directory> directory to HDFS-compatible file system which checkpoint data
 * <output-file> file to which the word counts will be appended
 *
 * <checkpoint-directory> and <output-file> must be absolute paths
 *
 * To run this on your local machine, you need to first run a Netcat server
 *
 * `$ nc -lk 9999`
 *
 * and run the example as
 *
 * `$ ./bin/run-example org.apache.spark.examples.streaming.RecoverableNetworkWordCount \
 * localhost 9999 ~/checkpoint/ ~/out`
 *
 * If the directory ~/checkpoint/ does not exist (e.g. running for the first time), it will create
 * a new StreamingContext (will print "Creating new context" to the console). Otherwise, if
 * checkpoint data exists in ~/checkpoint/, then it will create StreamingContext from
 * the checkpoint data.
 *
 * Refer to the online documentation for more details.
 */
object RecoverableNetworkWordCount {

  def createContext(ip: String, port: Int, outputPath: String, checkpointDirectory: String)
  : StreamingContext = {

    // If you do not see this printed, that means the StreamingContext has been loaded
    // from the new checkpoint
    println("Creating new context")
    val outputFile = new File(outputPath)
    if (outputFile.exists()) outputFile.delete()
    val sparkConf = new SparkConf().setAppName("RecoverableNetworkWordCount")
    // Create the context with a 1 second batch size
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ssc.checkpoint(checkpointDirectory)

    // Create a socket stream on target ip:port and count the
    // words in input stream of \n delimited text (e.g. generated by 'nc')
    val lines = ssc.socketTextStream(ip, port)
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map((_, 1)).reduceByKey(_ + _)
    wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
      // Get or register the excludeList Broadcast
      val excludeList = WordExcludeList.getInstance(rdd.sparkContext)
      // Get or register the droppedWordsCounter Accumulator
      val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
      // Use excludeList to drop words and use droppedWordsCounter to count them
      val counts = rdd.filter { case (word, count) =>
        if (excludeList.value.contains(word)) {
          droppedWordsCounter.add(count)
          false
        } else {
          true
        }
      }.collect().mkString("[", ", ", "]")
      val output = s"Counts at time $time $counts"
      println(output)
      println(s"Dropped ${droppedWordsCounter.value} word(s) totally")
      println(s"Appending to ${outputFile.getAbsolutePath}")
      Files.append(output + "\n", outputFile, Charset.defaultCharset())
    }
    ssc
  }

  def main(args: Array[String]): Unit = {
    if (args.length != 4) {
      System.err.println(s"Your arguments were ${args.mkString("[", ", ", "]")}")
      System.exit(1)
    }
    val Array(ip, portStr, checkpointDirectory, outputPath) = args
    val port = Try(portStr.toInt).get
    val ssc = StreamingContext.getOrCreate(checkpointDirectory,
      () => createContext(ip, port, outputPath, checkpointDirectory))
    ssc.start()
    ssc.awaitTermination()
  }
}import com.google.common.io.Files
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.rdd.RDD
import org.apache.spark.streaming.{Seconds, StreamingContext, Time}
import org.apache.spark.util.LongAccumulator
import org.apache.spark.{SparkConf, SparkContext}

import java.io.File
import java.nio.charset.Charset
import scala.util.Try

/**
 * Use this singleton to get or register a Broadcast variable.
 */
object WordExcludeList {

  @volatile private var instance: Broadcast[Seq[String]] = null

  def getInstance(sc: SparkContext): Broadcast[Seq[String]] = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          val wordExcludeList = Seq("a", "b", "c")
          instance = sc.broadcast(wordExcludeList)
        }
      }
    }
    instance
  }
}

/**
 * Use this singleton to get or register an Accumulator.
 */
object DroppedWordsCounter {

  @volatile private var instance: LongAccumulator = null

  def getInstance(sc: SparkContext): LongAccumulator = {
    if (instance == null) {
      synchronized {
        if (instance == null) {
          instance = sc.longAccumulator("DroppedWordsCounter")
        }
      }
    }
    instance
  }
}

/**
 * Counts words in text encoded with UTF8 received from the network every second. This example also
 * shows how to use lazily instantiated singleton instances for Accumulator and Broadcast so that
 * they can be registered on driver failures.
 *
 * Usage: RecoverableNetworkWordCount <hostname> <port> <checkpoint-directory> <output-file>
 * <hostname> and <port> describe the TCP server that Spark Streaming would connect to receive
 * data. <checkpoint-directory> directory to HDFS-compatible file system which checkpoint data
 * <output-file> file to which the word counts will be appended
 *
 * <checkpoint-directory> and <output-file> must be absolute paths
 *
 * To run this on your local machine, you need to first run a Netcat server
 *
 * `$ nc -lk 9999`
 *
 * and run the example as
 *
 * `$ ./bin/run-example org.apache.spark.examples.streaming.RecoverableNetworkWordCount \
 * localhost 9999 ~/checkpoint/ ~/out`
 *
 * If the directory ~/checkpoint/ does not exist (e.g. running for the first time), it will create
 * a new StreamingContext (will print "Creating new context" to the console). Otherwise, if
 * checkpoint data exists in ~/checkpoint/, then it will create StreamingContext from
 * the checkpoint data.
 *
 * Refer to the online documentation for more details.
 */
object RecoverableNetworkWordCount {

  def createContext(ip: String, port: Int, outputPath: String, checkpointDirectory: String)
  : StreamingContext = {

    // If you do not see this printed, that means the StreamingContext has been loaded
    // from the new checkpoint
    println("Creating new context")
    val outputFile = new File(outputPath)
    if (outputFile.exists()) outputFile.delete()
    val sparkConf = new SparkConf().setAppName("RecoverableNetworkWordCount")
    // Create the context with a 1 second batch size
    val ssc = new StreamingContext(sparkConf, Seconds(1))
    ssc.checkpoint(checkpointDirectory)

    // Create a socket stream on target ip:port and count the
    // words in input stream of \n delimited text (e.g. generated by 'nc')
    val lines = ssc.socketTextStream(ip, port)
    val words = lines.flatMap(_.split(" "))
    val wordCounts = words.map((_, 1)).reduceByKey(_ + _)
    wordCounts.foreachRDD { (rdd: RDD[(String, Int)], time: Time) =>
      // Get or register the excludeList Broadcast
      val excludeList = WordExcludeList.getInstance(rdd.sparkContext)
      // Get or register the droppedWordsCounter Accumulator
      val droppedWordsCounter = DroppedWordsCounter.getInstance(rdd.sparkContext)
      // Use excludeList to drop words and use droppedWordsCounter to count them
      val counts = rdd.filter { case (word, count) =>
        if (excludeList.value.contains(word)) {
          droppedWordsCounter.add(count)
          false
        } else {
          true
        }
      }.collect().mkString("[", ", ", "]")
      val output = s"Counts at time $time $counts"
      println(output)
      println(s"Dropped ${droppedWordsCounter.value} word(s) totally")
      println(s"Appending to ${outputFile.getAbsolutePath}")
      Files.append(output + "\n", outputFile, Charset.defaultCharset())
    }
    ssc
  }

  def main(args: Array[String]): Unit = {
    if (args.length != 4) {
      System.err.println(s"Your arguments were ${args.mkString("[", ", ", "]")}")
      System.exit(1)
    }
    val Array(ip, portStr, checkpointDirectory, outputPath) = args
    val port = Try(portStr.toInt).get
    val ssc = StreamingContext.getOrCreate(checkpointDirectory,
      () => createContext(ip, port, outputPath, checkpointDirectory))
    ssc.start()
    ssc.awaitTermination()
  }
}
```
