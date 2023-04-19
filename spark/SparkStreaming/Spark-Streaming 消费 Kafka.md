[TOC]

参考：<br>
[Spark-Streaming 消费 Kafka 多 Topic 多 Partition](https://blog.csdn.net/weixin_43215250/article/details/97006313?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

[kafka(十八):Streaming消费多个topic实例，并分别处理对应消息](https://blog.csdn.net/u010886217/article/details/103549856?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.channel_param)

多个topic，处理逻辑不一致，利用redis进行偏移量事务控制
```Scala
import org.apache.kafka.clients.consumer.ConsumerRecord
import org.apache.kafka.common.serialization.StringDeserializer
import org.apache.spark.streaming.dstream.InputDStream
import org.apache.spark.streaming.kafka010._
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.{SparkConf, TaskContext}

object SparkKafka {
  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("SparkKafka0_10").setMaster("local[*]")


    //设定批次时间为10s
    val ssc = new StreamingContext(conf, Seconds(10))

    val kafkaParams: Map[String, Object] = Map[String, Object](
      "bootstrap.servers" -> "linux01:9092,linux02:9092,linux03:9092",
      "key.deserializer" -> classOf[StringDeserializer],
      "value.deserializer" -> classOf[StringDeserializer],
      "group.id" -> "g001",
      "auto.offset.reset" -> "earliest",
      "enable.auto.commit" -> (false: java.lang.Boolean) // true -> offset将会被记录到kafka [__consumer_offsets]
    )


    val topics = Array("Mytest", "Song_of_the_wind")

    val stream: InputDStream[ConsumerRecord[String, String]] = KafkaUtils.createDirectStream(ssc, LocationStrategies.PreferConsistent,
      ConsumerStrategies.Subscribe[String, String](topics, kafkaParams))

    stream.foreachRDD(rdd => { //rdd[KafkaRDD]
      //将rdd[KafkaRDD]进行强制转换(只有KafkaRDD类型的能够转换)，转为HasOffsetRanges类型  ——> 最后拿到rdd里面每条数据偏移量offsetRanges
      val offsetRanges: Array[OffsetRange] = rdd.asInstanceOf[HasOffsetRanges].offsetRanges

      val partitions: Int = rdd.getNumPartitions
      println("partitions num: " + partitions)


      //计算逻辑
      rdd.foreachPartition(partition => {
        val offset: OffsetRange = offsetRanges(TaskContext.get.partitionId)
        // println(s"${offset.topic} ${offset.partition} ${offset.fromOffset} ${offset.untilOffset}")

        /*   //开启事务
           val pipline: Pipeline = jedisClient.pipelined();
           pipline.multi()*/

        if (offset.topic == "Mytest") {
          //topic1 处理逻辑
          println("Mytest logic:" + s"${offset.topic} ${offset.partition} ${offset.fromOffset} ${offset.untilOffset}")
        }

        if (offset.topic == "Song_of_the_wind") {
          //topic2 处理逻辑
          partition.foreach(record => {
            println(record)
          })
          //  println("Song_of_the_wind logic:" + s"${offset.topic} ${offset.partition} ${offset.fromOffset} ${offset.untilOffset}")
        }

        //自己管理，更新Offset到redis
        println("更新偏移量" + s"${offset.topic} ${offset.partition} ${offset.fromOffset} ${offset.untilOffset}")


        /*   //提交事务
           pipline.exec
           //关闭pipeline
           pipline.sync
           //关闭连接
           jedisClient.close()*/

      })

    })

    ssc.start()
    ssc.awaitTermination()
  }
}
```
