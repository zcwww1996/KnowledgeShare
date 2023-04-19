[TOC]
来源： https://www.cnblogs.com/jchubby/p/5449373.html

来源： https://blog.csdn.net/xwc35047/article/details/71039830

# 1. 配置参数的方式和观察性能的方式
## 1.1 Spark配置参数的三种方式

1. spark-env.sh文件中配置：最近常使用的配置方式，格式可以参考其中的一些官方保留的配置。
2. 程序中通过SparkConf配置：通过SparkConf对象set方法设置键值对，比较直观。
3. 程序中通过System.setProperty配置：和方法二差不多。

值得一提的是一个略显诡异的现象，有些参数在spark-env.sh中配置并不起作用，反而要在程序中设置才有效果。

Spark的参数很多，一些默认的设置可以参考官网推荐的配置参数：/docs/latest/configuration.html

## 1.2 观察Spark集群的状态和相关性能方法

1. Web UI：即8088端口进入的UI界面。
2. Driver程序日志：根据程序提交方式的不同到指定的节点上观察Driver程序日志。
3. logs文件夹下的日志：Spark集群的大部分信息都会记录在这里。
4. works文件夹下的日志：主要记录Work节点的信息。
5. Profiler工具：没有使用过。

前景交代完毕，下面进入正题：

# 2. 调度与分区优化
## 2.1 小分区合并的问题

由于程序中过度使用filter算子或者使用不当，都会造成大量的小分区出现。 

因为每次过滤得到的结果只有原来数据集的一小部分，而这些量很小的数据同样会以一定的分区数并行化分配到各个节点中执行。

**带来的问题就是：任务处理的数据量很小，反复地切换任务所消耗的资源反而会带来很大的系统开销。**

**解决方案：使用重分区函数coalesce进行数据紧缩、减少分区数并设置shuffle=true保证任务是并行计算的**

减少分区数，虽然意味着并行度降低，但是相对比之前的大量小任务过度切换的消耗，却是比较值得的。

这里也可以直接使用repartition重分区函数进行操作，因为其底层使用的是coalesce并设置Shuffle=true

## 2.2 数据倾斜问题

这是一个生产环境中经常遇到的问题，典型的场景是：大量的数据被分配到小部分节点计算，而其他大部分节点却只计算小部分数据。

问题产生的原因有很多，可能且不全部包括：

- key的数据分布不均匀
- 业务数据本身原因
- 结构化表设计问题
- 某些SQL语句会造成数据倾斜

**可选的解决方案有**：

1. 增大任务数，减少分区数量：这种方法和解决小分区问题类似。
2. 对特殊的key进行处理，如空值等：直接过滤掉空值的key以免对任务产生干扰。
3. 使用广播：小数据量直接广播，大数据量先拆分之后再进行广播。
4. 还有一种场景是任务执行速度倾斜问题：集群中其他节点都计算完毕了，但是只有少数几个节点死活运行不完。(其实这和上面的那个场景是差不多的)

**解决方案**：

- 设置spark.speculation=true将执行事件过长的节点去掉，重新分配任务
- spark.speculation.interval用来设置执行间隔

## 2.3 并行度调整

官方推荐每个CPU CORE分配2-3个任务。

- 任务数太多：并行度太高，产生大量的任务启动和切换开销。
- 任务数太低：并行度过低，无法发挥集群并行计算能力，任务执行慢

Spark会根据文件大小默认配置Map阶段的任务数，所以我们能够自行调整的就是Reduce阶段的分区数了。

- reduceByKey等操作时通过numPartitions参数进行分区数量配置。
- 通过spark.default.parallelism进行默认分区数配置。

## 2.4 DAG调度执行优化

DAG图是Spark计算的基本依赖，所以建议：

1. 同一个Stage尽量容纳更多地算子，防止多余的Shuffle。

2. 复用已经cache的数据。

尽可能地在Transformation算子中完成对数据的计算，因为过多的Action算子会产生很多多余的Shuffle，在划分DAG图时会形成众多Stage。

# 3. 网络传输优化
## 3.1 大任务分发问题

Spark采用Akka的Actor模型来进行消息传递，包括数据、jar包和相关文件等。

而Akka消息通信传递默认的容量最大为10M，一旦传递的消息超过这个限制就会出现这样的错误：

Worker任务失败后Master上会打印“Lost TID：”

根据这个信息找到对应的Worker节点后查看SparkHome/work/目录下的日志，查看Serialized size of result是否超过10M，就可以知道是不是Akka这边的问题了。

一旦确认是Akka通信容量限制之后，就可以通过配置spark.akka.frameSize控制Akka通信消息的最大容量。

## 3.2 Broadcast在调优场景的使用

Broadcast广播，主要是用于共享Spark每个Task都会用到的一些只读变量。

对于那些每个Task都会用到的变量来说，如果每个Task都为这些变量分配内存空间显然会使用很多多余的资源，使用广播可以有效的避免这个问题，广播之后，这些变量仅仅会在每个**Executor**上保存一份，有Task需要使用时就到自己所属的executor上读取就ok。

广播变量在使用的时候，是被拉取到Executor上的BlockManager中，只需要第一个task使用的时候拉取一次，之后其他task使用就可以复用blockManager中的变量，不需要重新拉取，也不需要在task中保存这个数据结构

官方推荐，Task大于20k时可以使用，可以在控制台上看Task的大小。


```Scala
val blackIp=Set(ip1,ip2...)
#sc.broadcast创建广播变量
val blackIpBC=sc.broadcast(blackIp) 
// 广播变量.value在task内获取广播变量的实际内容
rdd.filter(row=>!blackIpBC.value.contains(row.ip))
```

在使用广播变量的时候，一个partition尽量只调用一次.value方法：

```scala
rdd.mapPartition(iter=>{
  val blackIps=blackIpsBC.value
  iter.filter(t=>!blackIps.contains(t.ip))
 })
```
这种做法，可以跳过每条数据都需要做广播变量是否存在的判断，是比较好的编码习惯。


**主动清理**<br>
通过调用upersist方法即可手动更新或清理

```bash
广播变量.unpersist() #只产出executor上的广播变量,默认为false，异步清除
广播变量.destroy() #同时删除driver和executor的广播变量，默认为true，阻塞删除
```


## 3.3 Collect结果过大的问题

大量数据时将数据存储在HDFS上或者其他，不是大量数据，但是超出Akka传输的Buffer大小，通过配置spark.akka.frameSize调整。

# 4. 序列化与压缩
## 4.1 通过序列化手段优化

序列化之前说过，好处多多，所以是推荐能用就用，Spark上的序列化方式有几种，具体的可以参考官方文档。

这里只简单介绍一下Kryo。

自定义可以被Kryo序列化的类的步骤：
### 4.1.1 自定义类继承 KryoRegistrator
如果是scala类MyClass1（scala中的trait就相当于java中的接口）

```scala
class MyClass1 extends Serializable {
    ......
}
```

如果是java类MyClass1：
```java
public MyClass1 Test2 implements Serializable {
    ......
}
```

### 4.1.2 设置序列化方式Kryo
1. 可以在配置文件spark-default.conf中添加该配置项（全局生效），比如：

```properties
spark.serializer= org.apache.spark.serializer.KryoSerializer
```

2. 或者在业务代码中通过SparkConf进行配置（针对当前application生效），比如：

```scala
val spark = SparkSession.builder().master("local[*]").appName("test").getOrCreate()
val conf = new SparkConf
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
```

3. 又或者在spark-shell、spark-submit脚本中启动，可以在命令中加上：

```bash
--conf spark.serializer=org.apache.spark.serializer.KryoSerializer
```

### 4.1.3 注册自定义类（非必须，但是强烈建议做)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))

### 4.1.4 配置 spark.kryoserializer.buffer
如果注册的要序列化的自定义的Class类型，本身特别大，比如包含的属性过百，会导致要序列化的对象过大。此时，可使用SparkConf.set()方法，配置`spark.kryoserializer.buffer.mb`，值默认为2M

## 4.2 通过压缩手段优化

Spark的Job大致可以分为两种：

- I/O密集型：即存在大量读取磁盘的操作。
- CPU密集型：即存在大量的数据计算，使用CPU资源较多。

对于I/O密集型的Job，能压缩就压缩，因为读磁盘的时候数据压缩了，占用空间小了，读取速度不就快了。

对于CPU密集型的Job，看具体CPU使用情况再做决定，因为使用压缩是需要消耗一些CPU资源的，如果当前CPU已经超负荷了，再使用压缩反而适得其反。

Spark支持两种压缩算法：

- LZF：高压缩比
- Snappy：高速度

一些压缩相关的参数配置：

1. spark.broadcast.compress：推荐为true
2. spark.rdd.compress：默认为false，看情况配置，压缩花费一些时间，但是可以节省大量内存空间
3. spark.io.compression.codec：org.apache.spark.io.LZFCompressionCodec根据情况选择压缩算法
4. spark.io.compressions.snappy.block.size：设置Snappy压缩的块大小

# 5. 其他优化方式
## 5.1 对外部资源的批处理操作

如操作数据库时，每个分区的数据应该一起执行一次批处理，而不是一条数据写一次，即map=>mapPartition。

## 5.2 reduce和reduceByKey

reduce：内部调用了runJob方法，是一个action操作。 

reduceByKey：内部只是调用了combineBykey，是Transformation操作。

大量的数据操作时，reduce汇总所有数据到主节点会有性能瓶颈，将数据转换为Key-Value的形式使用reduceByKey实现逻辑，会做类似mr程序中的Combiner的操作，Transformation操作分布式进行。

## 5.3 Shuffle操作符的内存使用

使用会触发Shuffle过程的操作符时，操作的数据集合太大造成OOM，每个任务执行过程中会在各自的内存创建Hash表来进行数据分组。

可以解决的方案可能有：

- 增加并行度即分区数可以适当解决问题
- 可以将任务数量扩展到超过集群整体的CPU core数