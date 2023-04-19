[TOC]
# 1. checkpoint、Cache、persist
## 1.1 checkpoint
为当前RDD设置检查点。该函数将会创建一个二进制的文件，并存储到checkpoint目录中，该目录是用SparkContext.setCheckpointDir()设置的

checkpoint 怎么实现？

RDD 需要经过 [ Initialized --> marked for checkpointing --> checkpointing in progress --> checkpointed ] 这几个阶段才能被 checkpoint。

**在进行 checkpoint 前，要先对 RDD 进行 cache**。checkpoint 会等到 job 结束后另外启动专门的 job 去完成 checkpoint，**<font color="red">也就是说需要 checkpoint 的 RDD 会被计算两次</font>**

### 1.1.1 checkpoint与persist的区别

rdd.persist(StorageLevel.DISK_ONLY) 与 checkpoint 也有区别。前者虽然可以将 RDD 的 partition 持久化到磁盘，但该 partition 由 blockManager 管理。一旦 driver program 执行结束，也就是executor) 所在进程 CoarseGrainedExecutorBackend stop，blockManager 也会 stop，<font color="red">**被persist到磁盘上的 RDD 也会被清空**</font>（整个 blockManager 使用的 local 文件夹被删除）。而  <font color="red">**checkpoint 将 RDD 持久化到 HDFS 或本地文件夹**</font>，如果不被手动 remove 掉，是一直存在的，也就是说可以被下一个 driver program 使用，而 persisted RDD 不能被其他 dirver program 使用。

## 1.2 Cache
哪些 RDD 需要 cache？
会被重复使用的（但不能太大），**cache 只使用 memory**

将要计算 RDD partition 的时候（而不是已经计算得到第一个 record 的时候）就去判断 partition 要不要被 cache。如果要被 cache 的话，先将 partition 计算出来，然后 cache 到内存，放在内存的Heap中。cache 只使用 memory，写磁盘的话那就叫 checkpoint 了。

cache 底层调用的是 persist 方法,调用 rdd.cache() 后，rdd 就变成 persistRDD 了，其 StorageLevel 为 MEMORY_ONLY。persistRDD 会告知 driver 说自己是需要被 persist 的。cache通过unpersit强制把数据从内存中清除掉。

### 1.2.1 cache与persist的区别
cache 底层调用的是 persist 方法，persist 的默认存储级别也是 memory only
**persist 与 cache 的主要区别是 persist 可以自定义存储级别。**

### 1.2.2 cache与checkpoint的区别
 **<font color="red">cache 将 RDD 以及 RDD 的血统(记录了这个RDD如何产生)缓存到内存中</font>**，当缓存的RDD失效的时候(如内存损坏)，它们可以通过血统重新计算来进行恢复。但是 **<font color="red">checkpoint 将 RDD 缓存到了 HDFS 中，同时斩断了它的血统(也就是RDD之前的那些依赖)</font>**。为什么要丢掉依赖？因为可以利用 HDFS 多副本特性保证容错！

## 1.3 persist
persist 的默认存储级别也是persist(StorageLevel.MEMORY_ONLY)。

RDD都有哪些缓存级别，查看 StorageLevel 类的源码：
```scala
object StorageLevel {
  val NONE = new StorageLevel(false, false, false, false)
  val DISK_ONLY = new StorageLevel(true, false, false, false)
  val DISK_ONLY_2 = new StorageLevel(true, false, false, false, 2)
  val MEMORY_ONLY = new StorageLevel(false, true, false, true)
  val MEMORY_ONLY_2 = new StorageLevel(false, true, false, true, 2)
  val MEMORY_ONLY_SER = new StorageLevel(false, true, false, false)
  val MEMORY_ONLY_SER_2 = new StorageLevel(false, true, false, false, 2)
  val MEMORY_AND_DISK = new StorageLevel(true, true, false, true)
  val MEMORY_AND_DISK_2 = new StorageLevel(true, true, false, true, 2)
  val MEMORY_AND_DISK_SER = new StorageLevel(true, true, false, false)
  val MEMORY_AND_DISK_SER_2 = new StorageLevel(true, true, false, false, 2)
  val OFF_HEAP = new StorageLevel(false, false, true, false)
  ......
}
```

详细的存储级别介绍如下：

> - **MEMORY_ONLY** : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，部分数据分区将不再缓存，在每次需要用到这些数据时重新进行计算。这是默认的级别。
> - **MEMORY_AND_DISK** : 将 RDD 以反序列化 Java 对象的形式存储在 JVM 中。如果内存空间不够，将未缓存的数据分区存储到磁盘，在需要使用这些分区时从磁盘读取。
> - **MEMORY_ONLY_SER** : 将 RDD 以序列化的 Java 对象的形式进行存储（每个分区为一个 byte 数组）。这种方式会比反序列化对象的方式节省很多空间，尤其是在使用 fast serializer时会节省更多的空间，但是在读取时会增加 CPU 的计算负担。
> - **MEMORY_AND_DISK_SER** : 类似于 MEMORY_ONLY_SER ，但是溢出的分区会存储到磁盘，而不是在用到它们时重新计算。
> - **DISK_ONLY** : 只在磁盘上缓存 RDD。
> - **MEMORY_ONLY_2**，**MEMORY_AND_DISK_2**，等等 : 与上面的级别功能相同，只不过每个分区在集群中两个节点上建立副本。
> - **OFF_HEAP**（实验中）: 类似于 MEMORY_ONLY_SER ，但是将数据存储在 off-heap memory，这需要启动 off-heap 内存

> 对于上述任意一种持久化策略，如果加上**后缀_2**，代表的是将每个持久化的数据，都**复制一份副本**，并将副本保存到**其他节点**上。这种基于副本的持久化机制主要用于进行容错。假如某个节点挂掉，节点的内存或磁盘中的持久化数据丢失了，那么后续对RDD计算时还可以使用该数据在其他节点上的副本。如果没有副本的话，就只能将这些数据从源头处重新计算一遍了

StorageLevel类的主构造器包含了5个参数：

*   **useDisk**：使用硬盘（外存）
*   **useMemory**：使用内存
*   **useOffHeap**：使用堆外内存，这是Java虚拟机里面的概念，堆外内存意味着把内存对象分配在Java虚拟机的堆以外的内存，这些内存直接受操作系统管理（而不是虚拟机）。这样做的结果就是能保持一个较小的堆，以减少垃圾收集对应用的影响。
*   **deserialized**：反序列化，其逆过程序列化（Serialization）是java提供的一种机制，将对象表示成一连串的字节；而反序列化就表示将字节恢复为对象的过程。序列化是对象永久化的一种机制，可以将对象及其属性保存起来，并能在反序列化后直接恢复这个对象
*   **replication**：备份数（在多个节点上备份）

# 2. spark任务执行的四个重要过程
1. 构建DAG,记录RDD的转换关系，就将记录了 RDD调用了什么方法，传入了什么函数，结束就是调用了 Action
2. DAGScheduler切分Stage:按照shuffle进行切分的，如果有shuffle就切一刀（逻辑上的切分）会切分成多个Stage, —个Stege対应一个taskset (装着相同业务逻辑的Task集合，Taskset中Task的数量跟该Stage中的分区数据量一致）
3. 将TaskSet中的Task调度到Executor中执行
4. 在Executor中执行Taskset的业务逻辑

## 1.4 persist `MEMORY_AND_DISK` & `DISK_ONLY`
链接：https://cloud.tencent.com/developer/article/1674515

假设程序中需要对一个接近 3T 的模型文件进行 cache

```scala
object Persona {

  def main(args: Array[String]): Unit = {

    val spark = SparkSession
      .builder
      .appName("模型 cache 测试")
      .getOrCreate()

    val actions = spark.sparkContext.textFile(args(0)).persist(StorageLevel.MEMORY_AND_DISK).setName("model")

    // 触发 cache，没有实际意义
    println(s"number of actions: ${actions.count()}")

    // 10 mins
    Thread.sleep(1000 * 60 * 10)
  }
}
```

测试思路，3T 的模型，如果要 cache 住，50G 的 Executor，**至少需要 3T * 1024G/T / 50G * 2 = 125个左右**。（乘以2是因为 Executor 的 JVM 默认大概会用 50% 的 Host 内存）。测试中使用20个。

**一、 `StorageLevel.MEMORY_AND_DISK`模式**

代码如果使用 `StorageLevel.MEMORY_AND_DISK`，会有个问题，因为20个 Executor，纯内存肯定是不能 Cache 整个模型的，模型数据会 spill 到磁盘，同时 JVM 会处于经常性的 GC，这样这个操作肯定是非常耗时的。

如下图，560G 基本是可用于 Cache 的内存了，其余时间一直在刷盘。

[![image](https://pic.imgdb.cn/item/622071c55baa1a80aba100e2.png)](https://s6.jpg.cm/2022/03/03/Lz1RDp.png)

所有 Executor 一直处于频繁的 GC。

[![](https://pic.imgdb.cn/item/622071c65baa1a80aba100e6.png)](https://s6.jpg.cm/2022/03/03/Lz1WV6.png)
Memory 撑爆，CPU 一直繁忙

光是一个 Job 引发的 cache 模型，17分钟完成17%，目测至少需要一个小时

[![](https://pic.imgdb.cn/item/622075c75baa1a80aba47b14.png)](https://s6.jpg.cm/2022/03/03/Lz1k9t.png)

**二、 `StorageLevel.DISK_ONLY`模式**

以下是调整了 cache 级别，改为 `StorageLevel.DISK_ONLY`。没有了 GC 消耗。

[![](https://pic.imgdb.cn/item/622071c65baa1a80aba100f2.png)](https://s6.jpg.cm/2022/03/03/Lz1nFT.png)

10分钟已经完成30%的 task 了

[![](https://pic.imgdb.cn/item/622075c75baa1a80aba47b18.png)](https://s6.jpg.cm/2022/03/03/Lz1FkC.png)


**三、 总结**

针对大数据集，如果在 Memory 不足够的情况下（TB 级别的基本都很难有匹配的资源），可以让其直接落到磁盘，通过减少 GC Time 来改善程序的 Performance。

# 3. jdbcRDD关系型数据库连接驱动
```scala
import java.sql.{Connection, DriverManager}

import org.apache.spark.rdd.JdbcRDD
import org.apache.spark.{SparkConf, SparkContext}

object JdbcRDDDemo {

  //这个getConn是在，在类加载时被定义了
    
//  val getConn = () => {
//    //函数在定义时，里面的逻辑不会被执行
//    DriverManager.getConnection("jdbc:mysql://localhost:3306/bigdata?characterEncoding=UTF-8","root","123568")
//
//  }

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("JdbcRDDDemo").setMaster("local[*]")

    val sc = new SparkContext(conf)

    val jdbcRDD = new JdbcRDD(
      sc,
      () => {
        DriverManager.getConnection("jdbc:mysql://localhost:3306/bigdata?characterEncoding=UTF-8","root","123568")
      },
      "SELECT id, name, age FROM logs WHERE id >= ? AND id < ?",
      1,
      5,
      2, //分区数量
      rs => {
        val id = rs.getLong(1)
        val name = rs.getString(2)
        val age = rs.getInt(3)
        (id, name,age)
      }
    )

    //对jdbcRDD在进行操作
    //jdbcRDD.filter.map
//    val rdd1 = jdbcRDD.map(t => {
//      (t._1, t._2, t._3 * 10)
//    })

    val r = jdbcRDD.collect()

    println(r.toBuffer)

    sc.stop()
  }

}
```
# 4. 自定义排序
**要网络传输的数据一定要实现序列化Serializable**
自定义对象实现Ordered
自定义排序时，要排序的数据和排序规则都要shuffle，实现序列化（要排序的数据和规则绑在了一起）
- 将比较规则封装到一个类里面（需new，实现Serializable）
- 将比较规则定义在一个样例类（不需new，已经实现序列化，元素不需要val、var修饰）
- 将属性定义在元组中（元组的比较的特点：先比较第一个，如果不相等，就按照第一个属性排序，如果相等，比较下一个）
- 利用隐式转换定义排序规则
```scala
import java.sql.{Connection, DriverManager}

import org.apache.spark.rdd.{JdbcRDD, RDD}
import org.apache.spark.{SparkConf, SparkContext}

object CustomSort4 {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("CustomSort4").setMaster("local[*]")

    val sc = new SparkContext(conf)

    //用spark对数据进行排序，首先安装颜值的从高到低进行排序，如果颜值相等，再按照年龄的升序进行排序
    val users = Array("1,laoduan,99,34", "2,laozhao,9999,18", "3,laoyang,99,29", "4,laoxu,100,27")

    //makeRDD并行化成RDD,底层调用parallelize
    val userLines = sc.makeRDD(users)

    //整理数据
    val tpRDD: RDD[(Long, String, Int, Int)] = userLines.map(line => {
      val fields = line.split(",")
      val id = fields(0).toLong
      val name = fields(1)
      val fv = fields(2).toInt
      val age = fields(3).toInt
      (id, name, fv, age)
    })

/*
    //利用了元组的比较的特点：先比较第一个，如果不相等，就按照第一个属性排序，如果相等，再比较下一个属性
    val sorted = tpRDD.sortBy(t => (-t._3, t._4))
*/

 //利用隐式转换
    implicit val rules = Ordering[(Int, Int)].on[(Long, String, Int, Int)](t => (-t._3, t._4))
//注掉implicit，按t类型比，就是按元组默认方法比（先比较第一个，第一个相同比第二个），比较id
    val sorted = tpRDD.sortBy(t => t)

    println(sorted.collect().toBuffer)

    sc.stop()
  }

}
```