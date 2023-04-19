[TOC]

# 1. kafka生产者

```java
import org.apache.kafka.clients.producer.Callback;
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;
import org.apache.kafka.clients.producer.RecordMetadata;

import java.util.Properties;
import java.util.Random;

public class Product {
    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "192.168.134.111:9092,192.168.134.112:9092,192.168.134.113:9092");
        props.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

        KafkaProducer<String, String> kafkaProducer = new KafkaProducer<String, String>(props);
        int count = 1;
        while (count <= 10) {
            Thread.sleep(500);
            int random = new Random().nextInt(3); //产生0-3的随机数，不包括3
            ProducerRecord<String, String> record =
                    new ProducerRecord<String, String>("Mytest", random, String.valueOf(random), "message");//主题，分区，key, 消息
         //消息如果不指定分区，通过key选择分区，如果也不指定key，落入随机分区
         
            kafkaProducer.send(record, new Callback() {
                public void onCompletion(RecordMetadata metadata, Exception exception) {

                    System.out.println("metadata = " + metadata.toString());
                }
            });

            count++;
        }


    }
}
```


<br>
Scala读文件写入Kafka<br>

```Scala
import java.util.{Properties, Random}

import org.apache.kafka.clients.producer.{Callback, KafkaProducer, ProducerRecord, RecordMetadata}

import scala.io.Source

object Product_scala {
  def main(args: Array[String]): Unit = {
    val areaInfo = Source.fromFile("E:\\data\\Input\\IPri4G\\20180901\\09\\123\\123_1.txt")
    val topic = "ipsy"

    val props = new Properties
    props.setProperty("bootstrap.servers", "192.168.134.111:9092,192.168.134.112:9092,192.168.134.113:9092")
    props.setProperty("key.serializer", "org.apache.kafka.common.serialization.StringSerializer")
    props.setProperty("value.serializer", "org.apache.kafka.common.serialization.StringSerializer")

    val kafkaProducer = new KafkaProducer[String, String](props)
    for (line <- areaInfo.getLines) {
      Thread.sleep(50)
      val random = new Random().nextInt(3) //产生0-3的随机数，不包括3
      //消息如果不指定分区，通过key选择分区，如果也不指定key，落入随机分区
      val record = new ProducerRecord[String, String](topic, random, String.valueOf(random), line) //主题, 分区, key, 消息

      kafkaProducer.send(record, new Callback() {
        override def onCompletion(metadata: RecordMetadata, exception: Exception): Unit = {
          println("metadata = " + metadata.toString)
        }
      })

    }
  }
}
```

## 1.1 生产者运行的时候只显示4行代码

[![图一](https://s1.ax1x.com/2020/08/11/aOiVIS.png "图一")](https://i.ibb.co/Y0Y4rz5/fe2c9ba9-fe2b-4797-9b33-9cfaa74d3477.png)<br>
图一

[![图二](https://s1.ax1x.com/2020/08/11/aOFY0P.png "图二")](https://i.ibb.co/FxphctB/70871481-d229-4f6c-9328-669ee41a6f09.png)<br>
图二

[![图三](https://s1.ax1x.com/2020/08/11/aOFrXn.png "图三")](https://i.ibb.co/jwV5hbZ/46b4d1c6-f339-4c5e-a0b5-3b568c3e0726.png)<br>
图三

生产者运行的时候只显示4行代码，加入log4j后，发现详细错误，如图2，连接主机linux01出错

经排查，发现linux里面kafaka配置文件配置的为主机名linux01，而windows里面没有配映射，idea里面写的是ip（图3），导致window里面找不到Linux01，报错

# 2. kafka消费者
```java
import org.apache.kafka.clients.consumer.ConsumerRecord;
import org.apache.kafka.clients.consumer.ConsumerRecords;
import org.apache.kafka.clients.consumer.KafkaConsumer;
import java.util.Arrays;
import java.util.Properties;


public class Consumer {
    public static void main(String[] args)  {
        Properties props = new Properties();
        props.setProperty("bootstrap.servers", "192.168.134.111:9092,192.168.134.112:9092,192.168.134.113:9092");
        props.setProperty("key.deserializer", org.apache.kafka.common.serialization.StringDeserializer.class.getName());
        props.setProperty("value.deserializer", org.apache.kafka.common.serialization.StringDeserializer.class.getName());
        props.setProperty("group.id", "class_26_day02_002");
        props.setProperty("auto.offset.reset", "earliest");
        props.setProperty("enable.auto.commit", "true");
        // 自动提交（where）消费到的数据偏移量? where = 在kafka的__consumer_offsets
        props.setProperty("auto.commit.interval.ms", "1000");
        KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        consumer.subscribe(Arrays.asList("myTopic"));
        while (true) {
            ConsumerRecords<String, String> records = consumer.poll(3000);
            // 一次拉取所有数据
            for (ConsumerRecord<String, String> record : records) {
                System.out.println("---------------------------------------");
                System.out.println("record = " + record.toString());
            }
            // Thread.sleep(15000);
            consumer.close();
        }
    }
}
```