[TOC]
# 开发调优
## 1.1 避免创建重复的RDD
对性能没有问题，但会造成代码混乱
 
## 1.2 尽可能复用同一个RDD，减少产生RDD的个数

比如说，有一个RDD的数据格式是key-value类型的，另一个是单value类型的，这两个RDD的value数据是完全一样的。那么此时我们可以只使用key-value类型的那个RDD，因为其中已经包含了另一个的数据。对于类似这种多个RDD的数据有重叠或者包含的情况，我们应该尽量复用一个RDD，这样可以尽可能地减少RDD的数量，从而尽可能减少算子执行的次数。

**一个简单的例子**

```scala
// 错误的做法。

// 有一个<Long, String>格式的RDD，即rdd1。
// 接着由于业务需要，对rdd1执行了一个map操作，创建了一个rdd2，而rdd2中的数据仅仅是rdd1中的value值而已，也就是说，rdd2是rdd1的子集。
JavaPairRDD<Long, String> rdd1 = ...
JavaRDD<String> rdd2 = rdd1.map(...)

// 分别对rdd1和rdd2执行了不同的算子操作。
rdd1.reduceByKey(...)
rdd2.map(...)

// 正确的做法。

// 上面这个case中，其实rdd1和rdd2的区别无非就是数据格式不同而已，rdd2的数据完全就是rdd1的子集而已，却创建了两个rdd，并对两个rdd都执行了一次算子操作。
// 此时会因为对rdd1执行map算子来创建rdd2，而多执行一次算子操作，进而增加性能开销。

// 其实在这种情况下完全可以复用同一个RDD。
// 我们可以使用rdd1，既做reduceByKey操作，也做map操作。
// 在进行第二个map操作时，只使用每个数据的tuple._2，也就是rdd1中的value值，即可。
JavaPairRDD<Long, String> rdd1 = ...
rdd1.reduceByKey(...)
rdd1.map(tuple._2...)

// 第二种方式相较于第一种方式而言，很明显减少了一次rdd2的计算开销。
// 但是到这里为止，优化还没有结束，对rdd1我们还是执行了两次算子操作，rdd1实际上还是会被计算两次。
// 因此还需要配合“原则三：对多次使用的RDD进行持久化”进行使用，才能保证一个RDD被多次使用时只被计算一次。
```

## 1.3 对多次使用的RDD进行持久化(cache,persist,checkpoint)
**如何选择一种最合适的持久化策略？**

默认<font color="red">**MEMORY_ONLY**</font>, 性能很高， 而且不需要复制一份数据的副本，远程传送到其他节点上（BlockManager中的BlockTransferService），但是这里必须要注意的是，在实际的生产环境中，恐怕能够直接用这种

策略的场景还是有限的，如果RDD中数据比较多时（比如几十亿），直接用这种持久化级别，会导致JVM的OOM内存溢出异常。

如果使用**MEMORY_ONLY级别时发生了内存溢出，建议尝试使用<font color="red">MEMORY_ONLY_SER级别</font>**，该级别会将RDD数据序列化后再保存在内存中，此时每个partition仅仅是一个字节数组而已，大大减少了对象数量，并降低了内存占用。**这种级别比MEMORY_ONLY多出来的性能开销，主要就是序列化与反序列化的开销**。

**如果纯内存的级别都无法使用，那么建议使用<font color="red">MEMORY_AND_DISK_SER策略</font>**，而不是MEMORY_AND_DISK策略。因为既然到了这一步，就说明RDD的数据量很大，内存无法完全放下。序列化后的数据比较少，可以节省内存和磁盘的空间开销。同时**该策略会优先尽量尝试将数据缓存在内存中，内存缓存不下才会写入磁盘**。

**通常不建议使用DISK_ONLY和后缀为_2的级别：因为完全基于磁盘文件进行数据的读写**，会导致性能急剧降低，有时还不如重新计算一次所有RDD。后缀为_2的级别，必须将所有数据都复制一份副本，并发送到其他节点上，数据复制以及网络传输会导致较大的性能开销，**除非是要求作业的高可用性，否则不建议使用**。
    
**checkpoint** 可以使数据安全，**切断依赖关系**（如果某一个rdd丢失了，重新计算的链太长？）
 
## 1.4 尽量避免使用shuffle类的算子
广播变量模拟join(一个RDD比较小，另一个RDD比较大)
 
## 1.5 使用map-side预聚合shuffle操作
    reduceByKey    aggregateByKey
 
## 1.6 使用高性能的算子
有哪些高性能的算子？
- **reduceByKey/aggregateByKey** 替代 groupByKey
- **mapPartitions** 替代普通map Transformation算子
- **foreachPartitions** 替代 foreach Action算子
- **repartitionAndSortWithinPartitions**  替代repartition与sort类操作 
- **rdd.partitionBy()** //其实自定义一个分区器  
- **repartition** coalesce(numPartitions，true) 增多分区使用这个
- **coalesce**(numPartitions，false) 减少分区 没有shuffle只是合并partition

## 1.7 广播变量
开发过程中，会遇到需要在算子函数中使用外部变量的场景（尤其是大变量，比如100M以上的大集合），那么此时就应该使用Spark的广播（Broadcast）功能来提升性能,如果使用的外部变量比较大，建议使用Spark的广播功能，对该变量进行广播。广播后的变量，**会保证每个Executor的内存中，只驻留一份变量副本，而Executor中的task执行时共享该Executor中的那份变量副本**。这样的话，可以**大大减少变量副本的数量，从而减少网络传输的性能开销，并减少对Executor内存的占用开销，降低GC的频率**
 
    Executor 2G, 2G*0.48
 
## 1.8 使用kryo序列化性能？
Spark支持使用Kryo序列化机制。Kryo序列化机制，比默认的Java序列化机制，**速度要快，序列化后的数据要更小**，大概是Java序列化机制的1/10。所以Kryo序列化优化以后，可以让网络传输的数据变少；在集群中耗费的内存资源大大减少。

对于这三种出现序列化的地方，我们都可以通过使用Kryo序列化类库，来优化序列化和反序列化的性能。**Spark默认使用的是Java的序列化机制，也就是ObjectOutputStream/ObjectInputStream API来进行序列化和反序列化**。但是<font color="red">**Spark同时支持使用Kryo序列化库，Kryo序列化类库的性能比Java序列化类库的性能要高很多**</font>。官方介绍，Kryo序列化机制比Java序列化机制，性能高10倍左右。Spark之所以默认没有使用Kryo作为序列化类库，是因为**Kryo要求最好要<font color="red">注册</font>所有需要进行序列化的自定义类型**，因此对于开发者来说，这种方式比较麻烦

自定义的类有哪些，都要注册

[![自定义类注册](https://www.helloimg.com/images/2023/01/16/oGjjeD.png "自定义类注册")](https://imageproxy.pimg.tw/resize?url=https://i.niupic.com/images/2019/12/13/_69.png)

## 1.9 优化数据结构
尽量**使用字符串代替对象，使用原始类型（Int,long）替代字符串，使用数组替代集合类型**，这样尽可能地减少内存占用，从而降低GC频率，提升性能。
 
## 1.10 使用高性能的fastutil

- **fastutil是扩展了Java标准集合框架**（Map、List、Set；HashMap、ArrayList、HashSet）的类库，提供了特殊类型的map、set、list和queue；

- **fastutil能够提供更小的内存占用**，更快的存取速度；我们使用fastutil提供的集合类，来替代自己平时使用的JDK的原生的Map、List、Set，好处在于，fastutil集合类，可以减小内存的占用，并且在进行集合的遍历、根据索引（或者key）获取元素的值和设置元素的值的时候，提供更快的存取速度；

- **fastutil最新版本要求Java 7以及以上版本**；

- **fastutil的每一种集合类型，都实现了对应的Java中的标准接口**（比如fastutil的map，实现了Java的Map接口），因此可以直接放入已有系统的任何代码中。

<font style="background-color: yellow;">**RandomExtractCars**</font>
- 提供了一些集合，性能更高
- 必须是java7以上的版本

[![fastutil](https://www.helloimg.com/images/2023/02/22/oPGFoY.png "fastutil")](https://ae01.alicdn.com/kf/Hdd45a5e76e264e1e8445f4aa960261b5x.png)

来源： https://www.cnblogs.com/haozhengfei/p/052d52fab3885adf74cbfcff05739e90.html

# 资源调优

## num-executors

*   参数说明：该参数用于设置Spark作业总共要用多少个Executor进程来执行。Driver在向YARN集群管理器申请资源时，YARN集群管理器会尽可能按照你的设置来在集群的各个工作节点上，启动相应数量的Executor进程。这个参数非常之重要，如果不设置的话，默认只会给你启动少量的Executor进程，此时你的Spark作业的运行速度是非常慢的。
*   参数调优建议：每个Spark作业的运行一般设置50~100个左右的Executor进程比较合适，设置太少或太多的Executor进程都不好。设置的太少，无法充分利用集群资源；设置的太多的话，大部分队列可能无法给予充分的资源。

## executor-memory

*   参数说明：该参数用于设置每个Executor进程的内存。Executor内存的大小，很多时候直接决定了Spark作业的性能，而且跟常见的JVM OOM异常，也有直接的关联。
*   参数调优建议：每个Executor进程的内存设置4G~8G较为合适。但是这只是一个参考值，具体的设置还是得根据不同部门的资源队列来定。可以看看自己团队的资源队列的最大内存限制是多少，num-executors乘以executor-memory，是不能超过队列的最大内存量的。此外，如果你是跟团队里其他人共享这个资源队列，那么申请的内存量最好不要超过资源队列最大总内存的1/3~1/2，避免你自己的Spark作业占用了队列所有的资源，导致别的同学的作业无法运行。

## executor-cores

*   参数说明：该参数用于设置每个Executor进程的CPU core数量。这个参数决定了每个Executor进程并行执行task线程的能力。因为每个CPU core同一时间只能执行一个task线程，因此每个Executor进程的CPU core数量越多，越能够快速地执行完分配给自己的所有task线程。
*   参数调优建议：Executor的CPU core数量设置为2~4个较为合适。同样得根据不同部门的资源队列来定，可以看看自己的资源队列的最大CPU core限制是多少，再依据设置的Executor数量，来决定每个Executor进程可以分配到几个CPU core。同样建议，如果是跟他人共享这个队列，那么num-executors * executor-cores不要超过队列总CPU core的1/3~1/2左右比较合适，也是避免影响其他同学的作业运行。

## driver-memory

*   参数说明：该参数用于设置Driver进程的内存。
*   参数调优建议：Driver的内存通常来说不设置，或者设置1G左右应该就够了。唯一需要注意的一点是，如果需要使用collect算子将RDD的数据全部拉取到Driver上进行处理，那么必须确保Driver的内存足够大，否则会出现OOM内存溢出的问题。

## spark.default.parallelism

*   参数说明：该参数用于设置每个stage的默认task数量。这个参数极为重要，如果不设置可能会直接影响你的Spark作业性能。
*   参数调优建议：Spark作业的默认task数量为500~1000个较为合适。很多同学常犯的一个错误就是不去设置这个参数，那么此时就会导致Spark自己根据底层HDFS的block数量来设置task的数量，默认是一个HDFS block对应一个task。通常来说，Spark默认设置的数量是偏少的（比如就几十个task），如果task数量偏少的话，就会导致你前面设置好的Executor的参数都前功尽弃。试想一下，无论你的Executor进程有多少个，内存和CPU有多大，但是task只有1个或者10个，那么90%的Executor进程可能根本就没有task执行，也就是白白浪费了资源！因此Spark官网建议的设置原则是，设置该参数为num-executors * executor-cores的2~3倍较为合适，比如Executor的总CPU core数量为300个，那么设置1000个task是可以的，此时可以充分地利用Spark集群的资源。

## spark.storage.memoryFraction

*   参数说明：该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就是说，默认Executor 60%的内存，可以用来保存持久化的RDD数据。根据你选择的不同的持久化策略，如果内存不够时，可能数据就不会持久化，或者数据会写入磁盘。
*   参数调优建议：如果Spark作业中，有较多的RDD持久化操作，该参数的值可以适当提高一些，保证持久化的数据能够容纳在内存中。避免内存不够缓存所有的数据，导致数据只能写入磁盘中，降低了性能。但是如果Spark作业中的shuffle类操作比较多，而持久化操作比较少，那么这个参数的值适当降低一些比较合适。此外，如果发现作业由于频繁的gc导致运行缓慢（通过spark web ui可以观察到作业的gc耗时），意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。

## spark.shuffle.memoryFraction

*   参数说明：该参数用于设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2。也就是说，Executor默认只有20%的内存用来进行该操作。shuffle操作在进行聚合时，如果发现使用的内存超出了这个20%的限制，那么多余的数据就会溢写到磁盘文件中去，此时就会极大地降低性能。
*   参数调优建议：如果Spark作业中的RDD持久化操作较少，shuffle操作较多时，建议降低持久化操作的内存占比，提高shuffle操作的内存占比比例，避免shuffle过程中数据过多时内存不够用，必须溢写到磁盘上，降低了性能。此外，如果发现作业由于频繁的gc导致运行缓慢，意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。

资源参数的调优，没有一个固定的值，需要同学们根据自己的实际情况（包括Spark作业中的shuffle操作数量、RDD持久化操作数量以及spark web ui中显示的作业gc情况），同时参考本篇文章中给出的原理以及调优建议，合理地设置上述参数。


------------

**资源参数参考示例**

以下是一份spark-submit命令的示例，大家可以参考一下，并根据自己的实际情况进行调节：

```scala
./bin/spark-submit \
  --master yarn-cluster \
  --num-executors 100 \
  --executor-memory 6G \
  --executor-cores 4 \
  --driver-memory 1G \
  --conf spark.default.parallelism=1000 \
  --conf spark.storage.memoryFraction=0.5 \
  --conf spark.shuffle.memoryFraction=0.3 \
```