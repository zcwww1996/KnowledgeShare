[TOC]

# 1. Receiver（效率低）
**使用Kafka的高层次Consumer API来实现**。<br>
receiver从Kafka中获取的数据都存储在Spark Executor的内存中，然后Spark Streaming启动的job会去处理那些数据。然而，在默认的配置下，这种方式可能会因为底层的失败而丢失数据。如果要启用高可靠机制，让数据零丢失，就必须启用Spark Streaming的预写日志机制（Write Ahead Log，WAL）。该机制会同步地将接收到的Kafka数据写入分布式文件系统（比如HDFS）上的预写日志中。所以，即使底层节点出现了失败，也可以使用预写日志中的数据进行恢复。

**注意事项：**

1) Kafka中topic的partition与Spark中RDD的partition是没有关系的，因此，在KafkaUtils.createStream()中，提高partition的数量，只会增加Receiver的数量，也就是读取Kafka中topic partition的线程数量，不会增加Spark处理数据的并行度。
2) 可以创建多个Kafka输入DStream，使用不同的consumer group和topic，来通过多个receiver并行接收数据。
3) 如果基于容错的文件系统，比如HDFS，启用了预写日志机制，接收到的数据都会被复制一份到预写日志中。因此，在KafkaUtils.createStream()中，设置的持久化级别是StorageLevel.MEMORY_AND_DISK_SER。


# 2. Direct（调用了Kafka底层api）
Spark1.3中引入Direct方式，用来替代掉使用Receiver接收数据，这种方式会周期性地查询Kafka，获得每个topic+partition的最新的offset，从而定义每个batch的offset的范围。当处理数据的job启动时，就会使用Kafka的**简单consumer api**来获取Kafka指定offset范围的数据。

这种方式有如下优点：

1) 简化并行读取：<br>
如果要读取多个partition，不需要创建多个输入DStream，然后对它们进行union操作。Spark会创建跟Kafka partition一样多的RDD partition，并且会并行从Kafka中读取数据。所以在**Kafka partition和RDD partition之间，有一个一对一的映射关系**。

2) 高性能：<br>
如果要保证零数据丢失，在基于receiver的方式中，需要开启WAL机制。这种方式其实效率低下，因为数据实际上被复制了两份，Kafka自己本身就有高可靠的机制会对数据复制一份，而这里又会复制一份到WAL中。

而基于direct的方式，不依赖Receiver，不需要开启WAL机制，只要Kafka中作了数据的复制，那么就可以通过Kafka的副本进行恢复。

3) 一次且仅一次的事务机制：<br>
基于receiver的方式，是使用Kafka的高阶API来在**ZooKeeper中保存消费过的offset**的。这是消费Kafka数据的传统方式。这种方式配合着WAL机制可以保证数据零丢失的高可靠性，但是却无法保证数据被处理一次且仅一次，可能会处理两次。因为Spark和ZooKeeper之间可能是不同步的。

基于direct的方式，使用**kafka的底层api，Spark Streaming自己就负责追踪消费的offset**，并保存在checkpoint中。Spark自己一定是同步的，因此可以保证数据是消费一次且仅消费一次。由于数据消费偏移量是保存在checkpoint中，因此，如果后续想使用kafka高级API消费数据，需要手动的更新zookeeper中的偏移量

## 2.1 Direct代码


```scala
package bigdata.spark
 
import kafka.serializer.{StringDecoder, Decoder}
import org.apache.spark.streaming.kafka.KafkaUtils
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkContext, SparkConf}
 
import scala.reflect.ClassTag
 
/**
  * Created by Administrator on 2017/4/28.
  */
object SparkStreamDemo {
  def main(args: Array[String]) {
 
    val conf = new SparkConf()
    conf.setAppName("spark_streaming")
    conf.setMaster("local[*]")
 
    val sc = new SparkContext(conf)
    sc.setCheckpointDir("D:/checkpoints")
    sc.setLogLevel("ERROR")
 
    val ssc = new StreamingContext(sc, Seconds(5))
 
    // val topics = Map("spark" -> 2)
 
    val kafkaParams = Map[String, String](
      "bootstrap.servers" -> "m1:9092,m2:9092,m3:9092",
      "group.id" -> "spark",
      "auto.offset.reset" -> "smallest"
    )
    // 直连方式拉取数据，这种方式不会修改数据的偏移量，需要手动的更新
    val lines =  KafkaUtils.createDirectStream[String, String, StringDecoder, StringDecoder](ssc, kafkaParams, Set("spark")).map(_._2)
    // val lines = KafkaUtils.createStream(ssc, "m1:2181,m2:2181,m3:2181", "spark", topics).map(_._2)
 
    val ds1 = lines.flatMap(_.split(" ")).map((_, 1))
 
    val ds2 = ds1.updateStateByKey[Int]((x:Seq[Int], y:Option[Int]) => {
      Some(x.sum + y.getOrElse(0))
    })
 
    ds2.print()
 
    ssc.start()
    ssc.awaitTermination()
 
  }
}
```
