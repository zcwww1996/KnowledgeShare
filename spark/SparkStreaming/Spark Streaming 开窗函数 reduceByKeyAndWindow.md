https://blog.csdn.net/zimiao552147572/article/details/88558177

https://blog.csdn.net/hexinghua0126/article/details/80196640?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromBaidu-1

[![streaming-dstream-window](https://ftp.bmp.ovh/imgs/2020/05/4a34cfdacd058083.png)](https://s1.ax1x.com/2020/05/08/YnlwTS.png)

reduceByKeyAndWindow(_+_,Seconds(15), Seconds(10))

可以看到我们定义的window窗口大小Seconds(15s) ，是指每10s滑动时，需要统计前15s内所有的数据。

对于他的重载函数reduceByKeyAndWindow（_+_,_-_,Seconds(15s),seconds(10)）

设计理念是，当 滑动窗口的时间Seconds(10) < Seconds(15)（窗口大小）时，两个统计的部分会有重复，那么我们就可以不用重新获取或者计算，而是通过获取旧信息来更新新的信息，这样即节省了空间又节省了内容，并且效率也大幅提升。
    
如上图所示，2次统计重复的部分为time3对用的时间片内的数据，这样对于window1，和window2的计算可以如下所示

```bash
    win1 = time1 + time2 + time3
    win2 = time3 + time4 + time5
     
    更新为
    win1 = time1 + time2 + time3
    win2 = win1+ time4 + time5 - time2 - time3
```

这样就理解了吧,  _+_是对新产生的时间分片（time4,time5内RDD）进行统计，而_-_是对上一个窗口中，过时的时间分片(time1,time2) 进行统计   

```scala
import org.apache.log4j.{Level, Logger}
import org.apache.spark.SparkConf
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.dstream.ReceiverInputDStream
import org.apache.spark.streaming.kafka.KafkaUtils
 
object SparkReduceByKeyAndWindow {
  def main(args: Array[String]): Unit = {
 
    //由于日志信息较多，只打印错误日志信息
    Logger.getLogger("org.apache.spark").setLevel(Level.ERROR)
 
    val conf = new SparkConf().setAppName("dstream").setMaster("local[*]")
    //把批处理时间设置为5s一次
    val ssc = new StreamingContext(conf,Seconds(5))
    //使用updateStateByKey前需要设置checkpoint，将数据进行持久化保存，不然每次执行都是新的，不会与历史数据进行关联
        ssc.checkpoint("f:/spark_out")
    //将数据保存在hdfs中
//    ssc.checkpoint("hdfs://192.168.200.10:9000/spark_out")
    //与kafka做集成，使用KafkaUtils类进行参数配置
    val(zkQuorum,groupId,topics)=("192.168.200.10:2181","kafka_group",Map("sparkTokafka"->1))
    val value: ReceiverInputDStream[(String, String)] = KafkaUtils.createStream(ssc,zkQuorum,groupId,topics)
    //对滑动窗口中新的时间间隔内数据增加聚合并移去最早的与新增数据量的时间间隔内的数据统计量
    //Seconds(10)表示窗口的宽度   Seconds(15)表示多久滑动一次(滑动的时间长度)
    //综上所示：每5s进行批处理一次，窗口时间为15s范围内，每10s进行滑动一次
    //批处理时间<=滑动时间<=窗口时间，如果三个时间相等，表示默认处理单个的time节点
    value.flatMap(_._2.split(" ")).map((_,1)).reduceByKeyAndWindow(((x:Int,y:Int)=>x+y),Seconds(15),Seconds(10)).print()
    //启动数据接收
    ssc.start()
    //来等待计算完成
    ssc.awaitTermination()
  }
}
```



```java
reduceByKeyAndWindow函数的三个参数的意义：
1.第一个参数reduceFunc：需要一个函数
2.第二个参数windowDuration：
    窗口长度：窗口框住的是一段时间内的数据，即窗口负责框住多少个批次的数据。
    比如：设置窗口长度为10秒，然后如果以5秒内的数据划分为一个批次的话，
          那么这个10秒长度的窗口便框住了2个批次的数据。
3.第三个参数slideDuration：
    窗口滑动的时间长度：每隔“滑动指定的”时间长度计算一次，并且每当计算完毕后，窗口就往前滑动指定的时间长度。
    比如：窗口滑动的时间长度为10秒的话，那么窗口每隔10秒计算一次，并且每当计算完毕后，窗口就往前滑动10秒。
    比如：设置窗口长度为10秒，并且以5秒内的数据划分为一个批次的话，
          那么这个10秒长度的窗口便框住了2个批次的数据，而如果又设置了窗口滑动的时间长度为10秒的话，
          意味着窗口每隔10秒就计算一次，并且每当计算完毕后，窗口就往前滑动10秒。
4.结论：窗口长度和窗口滑动的时间长度必须为批次时间间隔的整数倍。
        窗口长度和窗口滑动的时间长度必须一致相同。
        窗口长度大于窗口滑动的时间长度，那么窗口仍然框住了部分已经计算过的批次数据，最终部分批次数据便会被重复计算
        窗口长度小于窗口滑动的时间长度，那么窗口并无法框住部分还没计算过的批次数据，最终部分批次数据便丢失
```

总的来说：

SparkStreaming提供这个方法主要是出于效率考虑。 比如说我要每10秒计算一下前15秒的内容，（每个batch 5秒）， 可以想象每十秒计算出来的结果和前一次计算的结果其实中间有5秒的时间值是重复的。 
那么就是通过如下步骤 
1. 存储上一个window的reduce值 
2. 计算出上一个window的begin 时间到 重复段的开始时间的reduce 值 =》 oldRDD 
3. 重复时间段的值结束时间到当前window的结束时间的值 =》 newRDD 
4. 重复时间段的值等于上一个window的值减去oldRDD 

这样就不需要去计算每个batch的值， 只需加加减减就能得到新的reduce出来的值。 


从代码上面来看， 入口为： 


```scala
reduceByKeyAndWindow(_+_, _-_, Duration, Duration)
```


先计算oldRDD 和newRDD 

： 
我们要计算的new rdd就是15秒-25秒期间的值， oldRDD就是0秒到10秒的值， previous window的值是1秒 - 15秒的值 

然后最终结果是 重复区间（previous window的值 - oldRDD的值） =》 也就是中间重复部分， 再加上newRDD的值， 这样的话得到的结果就是10秒到25秒这个时间区间的值


```java
// 0秒                  10秒     15秒                25秒  
//  _____________________________  
// |  previous window   _________|___________________  
// |___________________|       current window        |  --------------> Time  
//                     |_____________________________|  
//  
// |________ _________|          |________ _________|  
//          |                             |  
//          V                             V  
//       old RDDs                     new RDDs  
//
```
