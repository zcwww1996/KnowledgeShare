[TOC]
# 1. 写回Kafka场景-广播变量广播KafkaProducer
在实际的项目中，有时候我们需要把一些数据实时的写回到kafka中去，一般的话我们是这样写的，如下：

```scala
kafkaStreams.foreachRDD(rdd => {
      if (!rdd.isEmpty()) {
        rdd.foreachPartition(partition => {
          val properties = new Properties()
          properties.put("group.id", "jaosn_")
          properties.put("acks", "all")
          properties.put("retries ", "1")
          properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, PropertiesScalaUtils.loadProperties("broker"))
          properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName) //key的序列化;
          properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)
          val producer = new KafkaProducer[String, String](properties)
          partition.foreach(line => {
            producer.send(new ProducerRecord(PropertiesScalaUtils.loadProperties("topic") + "_error", line.value()))
          })
        })
      }
    })
```

但是这种写法有很严重的缺点，对于每个rdd的每一个partition的数据，每一次都需要创建一个KafkaProducer，显然这种做法是不太合理的，而且会带来性能问题，导致写的速度特别慢，那怎么解决这个问题呢？ 

**1、首先，我们需要将KafkaProducer利用lazy val的方式进行包装如下：**

```scala
package kafka

import java.util.concurrent.Future
import org.apache.kafka.clients.producer.{ KafkaProducer, ProducerRecord, RecordMetadata }


class broadcastKafkaProducer[K,V](createproducer:() => KafkaProducer[K,V]) extends Serializable{
  lazy val producer = createproducer()
  def send(topic:String,key:K,value:V):Future[RecordMetadata] = producer.send(new ProducerRecord[K,V](topic,key,value))
  def send(topic: String, value: V): Future[RecordMetadata] = producer.send(new ProducerRecord[K, V](topic, value))
}



object broadcastKafkaProducer {
  import scala.collection.JavaConversions._
  def apply[K,V](config:Map[String,Object]):broadcastKafkaProducer[K,V] = {
    val createProducerFunc  = () => {
      val producer = new KafkaProducer[K,V](config)
      sys.addShutdownHook({
        producer.close()
      })
      producer
    }
    new broadcastKafkaProducer(createProducerFunc)
  }
  def apply[K, V](config: java.util.Properties): broadcastKafkaProducer[K, V] = apply(config.toMap)
}
```

**2、之后我们利用广播变量的形式，将KafkaProducer广播到每一个executor，如下：**

```Scala
// 广播 broadcastKafkaProducer 到每一个excutors上面;
    val kafkaProducer: Broadcast[broadcastKafkaProducer[String, String]] = {
      val kafkaProducerConfig = {
        val p = new Properties()
        p.put("group.id", "jaosn_")
        p.put("acks", "all")
        p.put("retries ", "1")
        p.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, PropertiesScalaUtils.loadProperties("broker"))
        p.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName) //key的序列化;
        p.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, classOf[StringSerializer].getName)
        p
      }
      scc.sparkContext.broadcast(broadcastKafkaProducer[String, String](kafkaProducerConfig))
    }
```

**3、然后我们就可以在每一个executor上面将数据写入到kafka中了**

```Scala
kafkaStreams.foreachRDD(rdd => {
      if (!rdd.isEmpty()) {
        rdd.foreachPartition(partition => {
          partition.foreach(line => {
            kafkaProducer.value.send("topic_name",line.value())
          })
        })
      }
    })
```

这样的话，就不需要每次都去创建了，经过测试优化过的写法性能是之前的几十倍。

# 2. SparkStreaming集成kafka时比较重要的参数：

## 2.1 背压参数
- (1) **spark.streaming.stopGracefullyOnShutdown （true / false）默认fasle**<br>
确保在kill任务时，能够处理完最后一批数据，再关闭程序，不会发生强制kill导致数据处理中断，没处理完的数据丢失

- (2) **spark.streaming.backpressure.enabled （true / false） 默认false**<br>
开启后spark自动根据系统负载选择最优消费速率

- (3) **spark.streaming.backpressure.initialRate （整数）** 默认直接读取所有<br>
在（2）开启的情况下，限制第一次批处理应该消费的数据，因为程序冷启动 队列里面有大量积压，防止第一次全部读取，造成系统阻塞

- (4) **spark.streaming.kafka.maxRatePerPartition （整数）** 默认直接读取所有<br>
限制每秒每个消费线程读取每个kafka分区最大的数据量(   若maxRatePerPartition=500,Streaming批次为10秒，topic最大分区为3，则每批次最大接收消息数为500×3×10=15000)

==**注意**：==
> 
> 只有（4）激活的时候，每次消费的最大数据量，就是设置的数据量，如果不足这个数，就有多少读多少，如果超过这个数字，就读取这个数字的设置的值
> 
> 只有（2）+（4）激活的时候，每次消费读取的数量最大会等于（4）设置的值，最小是spark根据系统负载自动推断的值，消费的数据量会在这两个范围之内变化根据系统情况，但第一次启动会有多少读多少数据。此后按（2）+（4）设置规则运行
> 
> （2）+（3）+（4）同时激活的时候，跟上一个消费情况基本一样，但**第一次消费会得到限制，因为我们设置第一次消费的频率了**。


## 2.2 时间参数
- **spark.streaming.kafka.consumer.poll.ms=10000**<br>
设置从Kafka拉取数据的超时时间，超时则抛出异常重新启动一个task
- **spark.speculation=true**<br>
开启推测，防止某节点网络波动或数据倾斜导致处理时间拉长(推测会导致无数据处理的批次，也消耗与上一批次相同的执行时间，但不会超过批次最大时间，可能导致整体处理速度降低

- **session.timeout.ms<= coordinator检测失败的时间**
  - 默认值是10s
  - 该参数是 Consumer Group 主动检测 (组内成员comsummer)崩溃的时间间隔。若设置10min，那么Consumer Group的管理者（group coordinator）可能需要10分钟才能感受到。太漫长了是吧。

- **kafka.max.poll.interval.ms=3000000  <=轮询间隔(处理逻辑最大时间)**
  - 这个参数是0.10.1.0版本后新增的。需要根据实际业务处理时间进行设置，一旦Consumer处理不过来，就会被踢出Consumer Group 。
  - 注意：如果业务平均处理逻辑为1分钟，那么max. poll. interval. ms需要设置稍微大于1分钟即可，但是session. timeout. ms可以设置小一点（如10s），用于快速检测Consumer崩溃。
- **`request.timeout.ms`**
  - 这个配置控制一次请求响应的最长等待时间。如果在超时时间内未得到响应，kafka要么重发这条消息，要么超过重试次数的情况下直接置为失败。
  - 消息发送的最长等待时间，需大于session.timeout.ms这个时间

## 2.3 拉取大小
- **kafka.max.poll.records=10000<= 吞吐量**<br>
  - 默认值为500
  - 指定单次最大消费消息数量，如果处理逻辑很轻量，可以适当提高该值。
  - 一次从kafka中poll出来的数据条数,max.poll.records条数据需要在在session.timeout.ms这个时间内处理完
    
