[TOC]
# 1. Spark整合kafka_0.10_Direct
## 1.1 偏移量存储在kafka [__consumer_offsets]中
```scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.dstream.InputDStream
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.{Seconds, StreamingContext}

object SparkKafka0_10 {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("SparkKafka0_10").setMaster("local[*]")


    //设定批次时间为2s
    val ssc = new StreamingContext(conf,Seconds(2))

    val kafkaParams: Map[String, Object] = Map[String, Object](
      "bootstrap.servers" -> "linux01:9092,linux02:9092,linux03:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "class_26_day02_02",
      "auto.offset.reset" -> "earliest",
      "enable.auto.commit" -> (false: java.lang.Boolean)  // true -> offset将会被记录到kafka [__consumer_offsets]
    )


    val topics = Array("myTopic")

    val stream: InputDStream[ConsumerRecord[String, String]] = KafkaUtils.createDirectStream(ssc, LocationStrategies.PreferConsistent,
      ConsumerStrategies.Subscribe[String, String](topics, kafkaParams))

    stream.foreachRDD(rdd => {  //rdd[KafkaRDD]
      //将rdd[KafkaRDD]进行强制转换(只有KafkaRDD类型的能够转换)，转为HasOffsetRanges类型  ——> 最后拿到rdd里面每条数据偏移量offsetRanges
     val offsetRanges: Array[OffsetRange] = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
      val maped = rdd.map(record => (record.key(),record.value()))

      //计算逻辑
      maped.foreach(println)

      //自己存储起来，自己管理

      for (o <- offsetRanges ) {
        println(o.topic+"\t"+o.partition+"\t"+o.fromOffset+"\t"+o.untilOffset)
      }

      // 将偏移量数据写回到kafka Topic: __coonsumer_offsets
      //stream.asInstanceOf[CanCommitOffsets].commitAsync(offsetRanges)

    })

    ssc.start()
    ssc.awaitTermination()
  }
}

```

## 1.2 偏移量存储在zookeeper中

```scala
import kafka.utils.{ZKGroupTopicDirs, ZkUtils}
import org.I0Itec.zkclient.ZkClient
import org.apache.kafka.common.TopicPartition
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.SparkConf
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.{Seconds, StreamingContext}

object SparkZk0_10 {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("SparkZk0_10").setMaster("local[*]")


    //设定批次时间为2s
    val ssc = new StreamingContext(conf, Seconds(2))
    val groupId = "class_26_day02_03"
    val kafkaParams: Map[String, Object] = Map[String, Object](
      "bootstrap.servers" -> "linux01:9092,linux02:9092,linux03:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> groupId,
      "auto.offset.reset" -> "earliest",
      "enable.auto.commit" -> (false: java.lang.Boolean)  // true -> offset将会被记录到kafka [__consumer_offsets]
    )


    val topic = "myTopic"
    val topics = Array(topic)

    val zKGroupTopicDirs = new ZKGroupTopicDirs(groupId, topic)
    //   /consumers/class_26_day02_02/offsets/myTopic
    val zkCGOTPath: String = zKGroupTopicDirs.consumerOffsetDir

    //创建一个ZKClien，连上去
    val zkQuorum = "linux01:2181,linux02:2181,linux03:2181"
    val zkClient = new ZkClient(zkQuorum)
    //判断 consumers/class_26_day02_02/offsets/myTopic
    // 下是否有子节点，如果有说明之前维护过偏移量，有的话返回分区个数
    // 如果没有说明程序是第一次启动
    val childrenNum: Int = zkClient.countChildren(zkCGOTPath)
    val stream = if (childrenNum > 0) { //非第一次启动
      println("————非第一次启动—————")

      //如果之前维护过偏移量，消费数据的时候需要指定数据消费的偏移量
      // （接着上一次的Offset往后）位置

      //用来存储我们读到的偏移量数据信息
      var fromOffsets: Map[TopicPartition, Long] = Map[TopicPartition, Long]()
      (0 until childrenNum).foreach(partitionId => {
        val offset: String = zkClient.readData[String](zkCGOTPath + s"/${partitionId}")
        fromOffsets += (new TopicPartition(topic, partitionId) -> offset.toLong)
      })

      KafkaUtils.createDirectStream(ssc, LocationStrategies.PreferConsistent,
        ConsumerStrategies.Assign[String, String](fromOffsets.keys.toList, kafkaParams, fromOffsets))
    } else {
      //第一次启动
      println("********first************")
      KafkaUtils.createDirectStream(ssc, LocationStrategies.PreferConsistent,
        ConsumerStrategies.Subscribe[String, String](topics, kafkaParams))
    }


    stream.foreachRDD(rdd => {
      //将rdd[KafkaRDD]进行强制转换，转为HasOffsetRanges类型  ——> 最后拿到rdd里面每条数据偏移量offsetRanges
      val offsetRanges: Array[OffsetRange] = rdd.asInstanceOf[HasOffsetRanges].offsetRanges
      val maped = rdd.map(record => (record.key(), record.value()))

      //计算逻辑
      maped.foreach(println)

      //自己存储起来，自己管理

      for (o <- offsetRanges) {
        println(o.topic + "\t" + o.partition + "\t" + o.fromOffset + "\t" + o.untilOffset)

        // 写入到Zookeeper
        ZkUtils(zkClient, false).updatePersistentPath(zkCGOTPath + "/" + o.partition, o.untilOffset.toString)
      }


    })

    ssc.start()
    ssc.awaitTermination()
  }
}
```