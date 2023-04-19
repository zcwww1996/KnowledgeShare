[TOC]
# 一. 运维
## 1.Master挂掉,standby重启也失效

Master默认使用512M内存，当集群中运行的任务特别多时，就会挂掉，原因是master会读取每个task的event log日志去生成spark ui，内存不足自然会OOM，可以在master的运行日志中看到，通过HA启动的master自然也会因为这个原因失败。
**解决**
1. 增加Master的内存占用，在Master节点`spark-env.sh`中设置：
`export SPARK_DAEMON_MEMORY 10g # 根据你的实际情况`
2. 减少保存在Master内存中的作业信息
```shell
spark.ui.retainedJobs 500   # 默认都是1000
spark.ui.retainedStages 500
```

## 2.worker挂掉或假死

有时候我们还会在web ui中看到worker节点消失或处于dead状态，在该节点运行的任务则会报各种 lost worker 的错误，引发原因和上述大体相同，worker内存中保存了大量的ui信息导致gc时失去和master之间的心跳。

**解决**
1. 增加Master的内存占用，在Worker节点`spark-env.sh` 中设置：
`export SPARK_DAEMON_MEMORY 2g # 根据你的实际情况`
2. 减少保存在Worker内存中的Driver,Executor信息
```shell
spark.worker.ui.retainedExecutors 200   # 默认都是1000
spark.worker.ui.retainedDrivers 200
```

# 二. 运行错误
## 1.shuffle FetchFailedException

Spark Shuffle FetchFailedException解决方案

**错误提示**
### 1. missing output location
```java
org.apache.spark.shuffle.MetadataFetchFailedException:
Missing an output location for shuffle 0
```
[![missing output location](https://s2.ax1x.com/2019/12/10/QBkStU.jpg)](https://img-blog.csdn.net/20151015114309842)

### 2. shuffle fetch faild
```java
org.apache.spark.shuffle.FetchFailedException:
Failed to connect to spark047215/192.168.47.215:50268
```

[![shuffle fetch faild](https://s2.ax1x.com/2019/12/10/QBkeAK.jpg)](https://img-blog.csdn.net/20151015114008642)

当前的配置为每个executor使用1core,5G RAM,启动了20个executor

**解决**
这种问题一般发生在有大量shuffle操作的时候,task不断的failed,然后又重执行，一直循环下去，直到application失败。

[![task不断的failed](https://s2.ax1x.com/2019/12/10/QBkN4S.jpg)](https://img-blog.csdn.net/20151015114958881)

一般遇到这种问题提高executor内存即可,同时增加每个executor的cpu,这样不会减少task并行度。
- spark.executor.memory 15G
- spark.executor.cores 3
- spark.cores.max 21

启动的execuote数量为:7个
`execuoterNum = spark.cores.max/spark.executor.cores`

每个executor的配置：
`3core,15G RAM`

消耗的内存资源为:105G RAM
`15G*7=105G`

可以发现使用的资源并没有提升，但是同样的任务原来的配置跑几个小时还在卡着，改了配置后几分钟就能完成。

## 2.Executor&Task Lost

**错误提示**
1. executor lost
```java
WARN TaskSetManager: Lost task 1.0 in stage 0.0 (TID 1, aa.local):
ExecutorLostFailure (executor lost)
```

2. task lost
```java
WARN TaskSetManager: Lost task 69.2 in stage 7.0 (TID 1145, 192.168.47.217):
java.io.IOException: Connection from /192.168.47.217:55483 closed
```

3. 各种timeout
```java
java.util.concurrent.TimeoutException: Futures timed out after [120 second]
ERROR TransportChannelHandler: Connection to /192.168.47.212:35409
has been quiet for 120000 ms while there are outstanding requests.
Assuming connection is dead; please adjust spark.network.
timeout if this is wrong
```

**解决**
由网络或者gc引起,worker或executor没有接收到executor或task的心跳反馈。 
提高 `spark.network.timeout` 的值，根据情况改成300(5min)或更高。 
默认为 120(120s),配置所有网络传输的延时，如果没有主动设置以下参数，默认覆盖其属性
- spark.core.connection.ack.wait.timeout
- spark.akka.timeout
- spark.storage.blockManagerSlaveTimeoutMs
- spark.shuffle.io.connectionTimeout
- spark.rpc.askTimeout or spark.rpc.lookupTimeout

## 3.倾斜

**错误提示**

1. 数据倾斜

[![数据倾斜](https://s2.ax1x.com/2019/12/10/QBFblj.jpg)](https://img-blog.csdn.net/20151015152750199)

2. 任务倾斜

差距不大的几个task,有的运行速度特别慢。

**解决**

大多数任务都完成了，还有那么一两个任务怎么都跑不完或者跑的很慢，分为数据倾斜和task倾斜两种。

1. 数据倾斜

数据倾斜大多数情况是由于大量的无效数据引起，比如null或者""，也有可能是一些异常数据，比如统计用户登录情况时，出现某用户登录过千万次的情况，无效数据在计算前需要过滤掉。 
数据处理有一个原则，多使用filter，这样你真正需要分析的数据量就越少，处理速度就越快。
`sqlContext.sql("...where col is not null and col != ''")`

具体可参考:
[解决spark中遇到的数据倾斜问题](https://blog.csdn.net/lsshlsw/article/details/52025949 "解决spark中遇到的数据倾斜问题")

2. 任务倾斜

task倾斜原因比较多，网络io,cpu,mem都有可能造成这个节点上的任务执行缓慢，可以去看该节点的性能监控来分析原因。以前遇到过同事在spark的一台worker上跑R的任务导致该节点spark task运行缓慢。
或者可以开启spark的推测机制，开启推测机制后如果某一台机器的几个task特别慢，推测机制会将任务分配到其他机器执行，最后Spark会选取最快的作为最终结果。
- spark.speculation **true**
- spark.speculation.interval 100 --- 检测周期，单位毫秒；
- spark.speculation.quantile 0.75 --- 完成task的百分比时启动推测
- spark.speculation.multiplier 1.5 --- 比其他的慢多少倍时启动推测。

## 4.OOM

**错误提示**

堆内存溢出

`java.lang.OutOfMemoryError: Java heap space`

**解决**

内存不够，数据太多就会抛出OOM的Exeception，主要有driver OOM和executor OOM两种
1. driver OOM
一般是使用了collect操作将所有executor的数据聚合到driver导致。尽量不要使用collect操作即可。
2. executor OOM
可以按下面的内存优化的方法增加code使用内存空间
增加executor内存总量,也就是说增加`spark.executor.memory`的值

增加任务并行度（大任务就被分成小任务了)，参考下面优化并行度的方法

## 5.task not serializable

**错误提示**
```java
org.apache.spark.SparkException: Job aborted due to stage failure:
Task not serializable: java.io.NotSerializableException: ...
```

**解决**

如果你在worker中调用了driver中定义的一些变量，Spark就会将这些变量传递给Worker，这些变量并没有被序列化，所以就会看到如上提示的错误了。
```java
val x = new X()  //在driver中定义的变量
dd.map{r => x.doSomething(r) }.collect  //map中的代码在worker(executor)中执行
```
除了上文的map,还有filter,foreach,foreachPartition等操作，还有一个典型例子就是在foreachPartition中使用数据库创建连接方法。这些变量没有序列化导致的任务报错。

下面提供三种解决方法：
1. 将所有调用到的外部变量直接放入到以上所说的这些算子中，这种情况最好使用foreachPartition减少创建变量的消耗。
2. 将需要使用的外部变量包括`sparkConf`,`SparkContext`,都用 @transent进行注解，表示这些变量不需要被序列化
3. 将外部变量放到某个class中对类进行序列化。

## 6.driver.maxResultSize太小

**错误提示**
```java
Caused by: org.apache.spark.SparkException:
 Job aborted due to stage failure: Total size of serialized 
 results of 374 tasks (1026.0 MB) is bigger than
  spark.driver.maxResultSize (1024.0 MB)
```

**解决**

spark.driver.maxResultSize默认大小为1G 每个Spark action(如collect)所有分区的序列化结果的总大小限制，简而言之就是executor给driver返回的结果过大，报这个错说明需要提高这个值或者避免使用类似的方法，比如countByValue，countByKey等。

将值调大即可
`spark.driver.maxResultSize 2g`

## 7.taskSet too large

**错误提示**
`WARN TaskSetManager: Stage 198 contains a task of very large size (5953 KB). The maximum recommended task size is 100 KB.`


这个WARN可能还会导致ERROR

```java
Caused by: java.lang.RuntimeException: Failed to commit task

Caused by: org.apache.spark.executor.CommitDeniedException: attempt_201603251514_0218_m_000245_0: Not committed because the driver did not authorize commit
```

**解决**

如果你比较了解spark中的stage是如何划分的，这个问题就比较简单了。
一个Stage中包含的task过大，一般由于你的transform过程太长，因此driver给executor分发的task就会变的很大。

所以解决这个问题我们可以通过拆分stage解决。也就是在执行过程中调用cache.count缓存一些中间数据从而切断过长的stage。

## 8. driver did not authorize commit

启动Spark Speculative后，有时候运行任务会发现如下提示：
```java
WARN TaskSetManager: Lost task 55.0 in stage 15.0 (TID 20815, spark047216)

org.apache.spark.executor.CommitDeniedException: attempt_201604191557_0015_m_000055_0: Not committed because the driver did not authorize commit
```

启动 Speculative 后，运行较慢的task会在其他executor上同时再启动一个相同的task,如果其中一个task执行完毕，相同的另一个task就会被禁止提交。因此产生了这个WARN。

这个WARN是因为task提交commit被driver拒绝引发，这个错误不会被统计在stage的failure中，这样做的目的是防止你看到一些具有欺骗性的提示。
## 9. No lease on /目录: File does not exist
**错误提示：**
```java
WARN TaskSetManager: Lost task:org.apache.hadoop.ipc.RemoteException(org.apache.hadoop.hdfs.server.namenode.LeaseExpiredException): 
 No lease onNo lease on /user/hadoop/IPSY/Output/ONet/20180614/09/50_tmp/_temporary/0/_temporary/attempt_201806141050_0003_m_000034_43/part-00034 (inode 8126330):
 File does not exist. Holder DFSClient_NONMAPREDUCE_1239207978_115 does not have any open files.
```
**原因：**

为文件操作超租期，临时文件发生变更，hdfs找不到临时文件从而报错。多发于由于多个task操作写一个文件，其中某个task完成任务后删除了临时文件引起。
该参数和`dfs.datanode.max.xcievers`有关，默认为256。

`dfs.datanode.max.xcievers`表示每个datanode任一时刻可以打开的文件数量上限。

**解决**

有两个解决方法，一种是修改spark代码，一种是修改hdfs参数配置。

1.  避免太高的并发度同时写一个文件。
    所以在调用`write.parquet`前，先使用`repartition`合并碎片分区。
     因为减少了分区数，下次再读取这份数据进行处理时，减少了启动task的开销。

2.  提高同时写的上限。

    在`hdfs-site.xml`中修改`dfs.datanode.max.xcievers`,将其设置为4096

    ```java
     <property>
        <name>dfs.datanode.max.xcievers</name>
        <value>4096</value>
      </property>
    ```
需要重启dataNode生效。
## 10. 环境报错

1. driver节点内存不足

driver内存不足导致无法启动application，将driver分配到内存足够的机器上或减少driver-memory
```java
Java HotSpot(TM) 64-Bit Server VM warning: INFO:
os::commit_memory(0x0000000680000000, 4294967296, 0) failed;
error=’Cannot allocate memory’ (errno=12)
```

2. hdfs空间不够

hdfs空间不足，event_log无法写入，所以 ListenerBus会报错 ,增加hdfs空间（删除无用数据或增加节点）
```java
Caused by: org.apache.hadoop.ipc.RemoteException(java.io.IOException):
 File /tmp/spark-history/app-20151228095652-0072.inprogress

 could only be replicated to 0 nodes instead of minReplication (=1)
ERROR LiveListenerBus: Listener EventLoggingListener threw an exception
java.lang.reflect.InvocationTargetException
```

3. spark编译包与hadoop版本不一致

下载对应hadoop版本的spark包或自己编译。
```java
java.io.InvalidClassException: org.apache.spark.rdd.RDD;
local class incompatible: stream classdesc serialVersionUID
```
4. driver机器端口使用过多

在一台机器上没有指定端口的情况下，提交了超过15个任务。
```java
16/03/16 16:03:17 ERROR SparkUI: Failed to bind SparkUI
java.net.BindException: 地址已在使用: Service 'SparkUI' failed after 16 retries!
```
提交任务时指定app web ui端口号解决:
`-conf spark.ui.port=xxxx`

5. 中文乱码

使用write.csv等方法写出到hdfs的文件，中文乱码。JVM使用的字符集如果没有指定，默认会使用系统的字符集，因为各个节点系统字符集并不都是UTF8导致，所以会出现这个问题。直接给JVM指定字符集即可。

**spark-defaults.conf**

`spark.executor.extraJavaOptions -Dfile.encoding=UTF-8`

# 三. 一些优化

## 1. 部分Executor不执行任务
有时候会发现部分executor并没有在执行任务，为什么呢？

(1) 任务partition数过少

要知道每个partition只会在一个task上执行任务。改变分区数，可以通过 repartition 方法，即使这样，在 repartition 前还是要从数据源读取数据，此时（读入数据时）的并发度根据不同的数据源受到不同限制，常用的大概有以下几种：
- hdfs --- block数就是partition数
- mysql --- 按读入时的分区规则分partition
- es --- 分区数即为 es 的分片数（shard）

(2) 数据本地性的副作用

taskSetManager在分发任务之前会先计算数据本地性，优先级依次是：

**process(同一个executor) -> node_local(同一个节点) -> rack_local(同一个机架) -> any(任何节点)**

Spark会优先执行高优先级的任务，任务完成的速度很快（小于设置的spark.locality.wait时间），则数据本地性下一级别的任务则一直不会启动，这就是Spark的延时调度机制。

举个极端例子：运行一个count任务，如果数据全都堆积在某一台节点上，那将只会有这台机器在长期计算任务，集群中的其他机器则会处于等待状态（等待本地性降级）而不执行任务，造成了大量的资源浪费。
判断的公式为：

`curTime – lastLaunchTime >= localityWaits(currentLocalityIndex)`

其中 curTime 为系统当前时间，lastLaunchTime 为在某优先级下最后一次启动task的时间
如果满足这个条件则会进入下一个优先级的时间判断，直到 any，不满足则分配当前优先级的任务。
数据本地性任务分配的源码在 `taskSetManager.scala` 。
如果存在大量executor处于等待状态，可以降低以下参数的值（也可以设置为0），默认都是3s。

```java
spark.locality.wait
spark.locality.wait.process
spark.locality.wait.node
spark.locality.wait.rack
```
当你数据本地性很差，可适当提高上述值，当然也可以直接在集群中对数据进行balance。

## 2. spark task 连续重试失败

有可能哪台worker节点出现了故障，task执行失败后会在该 executor 上不断重试，达到最大重试次数后会导致整个 application 执行失败，我们可以设置失败黑名单(task在该节点运行失败后会换节点重试)，可以看到在源码中默认设置的是 0,

在`spark-default.sh`中设置
```java
private val EXECUTOR_TASK_BLACKLIST_TIMEOUT =  conf.getLong("spark.scheduler.executorTaskBlacklistTime", 0L)
```

在 `spark-default.sh` 中设置
`spark.scheduler.executorTaskBlacklistTime 30000`

当 task 在该 executor 运行失败后会在其它 executor 中启动，同时此 executor 会进入黑名单30s（不会分发任务到该executor）。

## 3. 内存
如果你的任务shuffle量特别大，同时rdd缓存比较少可以更改下面的参数进一步提高任务运行速度。

`spark.storage.memoryFraction` --- 分配给rdd缓存的比例，默认为0.6(60%)，如果缓存的数据较少可以降低该值。

`spark.shuffle.memoryFraction` --- 分配给shuffle数据的内存比例，默认为0.2(20%)
剩下的20%内存空间则是分配给代码生成对象等。
如果任务运行缓慢，jvm进行频繁gc或者内存空间不足，或者可以降低上述的两个值。

`spark.rdd.compress = true` --- 默认为false，压缩序列化的RDD分区,消耗一些cpu减少空间的使用

## 4. 并发
mysql读取并发度优化

spark 读取 hdfs 数据分区规则

`spark.default.parallelism`
发生shuffle时的并行度，在standalone模式下的数量默认为core的个数，也可手动调整，数量设置太大会造成很多小任务，增加启动任务的开销，太小，运行大数据量的任务时速度缓慢。

`spark.sql.shuffle.partitions`
sql聚合操作(发生shuffle)时的并行度，默认为200，如果该值太小会导致OOM,executor丢失，任务执行时间过长的问题

相同的两个任务：

spark.sql.shuffle.partitions=300:
[![300分区](https://s2.ax1x.com/2019/12/10/QBCj0K.jpg)](https://img-blog.csdn.net/20151015165917751)

spark.sql.shuffle.partitions=500:
[![500分区](https://s2.ax1x.com/2019/12/10/QBP6AO.jpg)](https://img-blog.csdn.net/20151015165959710)

速度变快主要是大量的减少了gc的时间。

但是设置过大会造成性能恶化，过多的碎片task会造成大量无谓的启动关闭task开销，还有可能导致某些task hang住无法执行。

[![分区过大](https://s2.ax1x.com/2019/12/10/QBiZK1.jpg)](https://img-blog.csdn.net/20151210161217631)
修改map阶段并行度主要是在代码中使用rdd.repartition(partitionNum)来操作。

## 5. shuffle
[spark-sql join优化](http://lswyyy.github.io/2015/09/24/spark-join-broadcast%E4%BC%98%E5%8C%96/)

[map-side-join 关联优化](https://blog.csdn.net/lsshlsw/article/details/50834858 "map-side-join 关联优化")

[spark range join 优化](https://blog.csdn.net/lsshlsw/article/details/79798805 "spark range join 优化")

## 6. 磁盘
[磁盘IO优化](https://blog.csdn.net/lsshlsw/article/details/50055599 "磁盘IO优化")

## 7.序列化
参考:[kryo Serialization](https://blog.csdn.net/lsshlsw/article/details/50856842 "kryo Serialization")

从Spark 2.0.0版本开始，简单类型、简单类型数组、字符串类型的Shuffling RDDs 已经默认使用Kryo序列化方式了。
**Kryo序列化库**

|Property Name|Default|Meaning|
| ------------ | ------------ | ------------ |
|spark.serializer|org.apache.spark.serializer.JavaSerializer|申明序列化时用的类,这个设置不仅控制各个worker节点之间的混洗数据序列化格式，同时还控制RDD存到磁盘上的序列化格式及广播变量的序列化格式|
|spark.kryoserializer.buffer|64k|每个Executor中的每个core对应着一个序列化buffer。如果你的对象很大，可能需要增大该配置项。其值不能超过spark.kryoserializer.buffer.max|
|spark.kryoserializer.buffer.max|64m|允许使用序列化buffer的最大值|
|spark.kryo.classesToRegister|(none)|向Kryo注册自定义的的类型，类名间用逗号分隔|
|spark.kryo.referenceTracking|true|跟踪对同一个对象的引用情况，这对发现有循环引用或同一对象有多个副本的情况是很有用的。设置为false可以提高性能|
|spark.kryo.registrationRequired|false|是否需要在Kryo登记注册？如果为true，则序列化一个未注册的类时会抛出异常|
|spark.kryo.registrator|(none)|为Kryo设置这个类去注册你自定义的类。最后，如果你不注册需要序列化的自定义类型，Kryo也能工作，不过每一个对象实例的序列化结果都会包含一份完整的类名，这有点浪费空间|
|spark.kryo.unsafe|false|如果想更加提升性能，可以使用Kryo unsafe方式|
### scala
```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer") // worker之间，rdd的序列化方式
conf.registerKryoClasses(Array(classOf[MyClass1],classOf[MyClass2])) // 自定义类必须手动的注册Kryo序列化方式
```
### java
主要的使用过程就三步：

1. 设置序列化使用的库
```java
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer");  //使用Kryo序列化库
conf.set("spark.kryo.registrator","MyRegistrator");       //在Kryo序列化库中注册自定义的类集合
```
2. 自定义的类集合实现KryoRegistrator接口
```java
public static class MyRegistrator implements KryoRegistrator {
    public void registerClasses(Kryo kryo) {
        kryo.register(MyClass1.class);  //在Kryo序列化库中注册自定义的类
        kryo.register(MyClass2.class);  //在Kryo序列化库中注册自定义的类
    }
}
```
3. 被注册的类要实现java.io.Serializable
```java
public class MyClass1 implements Serializable{
    int a;
    public int getA() {
        return a;
    }
    public void setA(int a) {
        this.a = a;
    }
}
```

## 8.数据本地性
[Spark不同Cluster Manager下的数据本地性表现](https://blog.csdn.net/lsshlsw/article/details/52215947 "Spark不同Cluster Manager下的数据本地性表现")

[spark读取hdfs数据本地性异常](https://blog.csdn.net/lsshlsw/article/details/48711519 "spark读取hdfs数据本地性异常")

## 9.代码
编写Spark程序的几个优化点

来源：https://blog.csdn.net/lsshlsw/article/details/49155087