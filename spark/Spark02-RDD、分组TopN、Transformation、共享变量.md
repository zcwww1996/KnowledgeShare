[toc]
# 1. RDD
## 1.1 定义
RDD 叫做分布式数据集，是Spark中最基本的数据抽象，它代表一个不可变、可分区、里面的元素可并行计算的集合。
RDD具有数据流模型的特点：自动容错、位置感知性调度和可伸缩性。

## 1.2 RDD的”弹性”（Resilient）
1. 自动的进行内存与磁盘数据存储的切换
2. 基于Lineage的高效容错
3. Task如果失败会自动进行特定次数的重试
4. Stage如果失败会自动进行特定次数的重试，而且只会计算失败的分片
5. checkpoint和persist:对于长链接的操作，把中间的数据放到磁盘上去
6. 数据分片的高度弹性（提高、降低并行度）
7. 数据调度弹性：DAG TASK和资源管理无关

## 1.3 创建RDD
**（1）将scala数组转为(并行化)转为rdd数组**

    val arr = Array(1,2,3,4,5,6,7,8)
    sc.parallelize(arr)
    //将数组分两块
    sc.parallelize(arr,2)
**（2）由外部存储系统的数据集创建**

包括本地的文件系统，还有所有Hadoop支持的数据集，比如HDFS、Cassandra、HBase等

    val rdd2 = sc.textFile("hdfs:/linux01:9000/words.txt")
**（3）RDD调用Transformation后会生成新的RDD**

## 1.4 RDD的5个特点
1. RDD里面有多个分区，分区是有编号的
2. 每一个输入切片都有一个功能作用在上面（功能：在Stage中有很多的计算逻辑合并在一个Task中了）
3. RDD和RDD直接存在依赖关系（有根据最后一个触发Action的RDD的依赖关系从后往前推断，然后划分Stage，最终生成Task）
4. （可选）如果RDD里面装的是K-V类型的RDD，会有一个分区器，默认是HashPartitioner
5. (可选）如果是读取HDFS中数据会有一个最优位置（在生成Task之前，先要跟NameNode逬行通信，请求元数据信息，然后告诉TaskScheduler)

## 1.5 宽依赖、窄依赖
**宽依赖**

宽依赖是划分Stage的依据
宽依赖其实是每个分区的数据打散重组了，父RDD一个分区的数据给了子RDD的多个分区，即使只存在这种可能也是宽依赖（实际情况可能由于hash碰撞，父RDD全给了某一个分区）
例：reduceBykey、groupByKey和join
宽依赖就是有shuffle产生，shuffle—定网络传输

**窄依赖**

窄依赖没有将数据打散，父RDD一个分区的数据，只给了子RDD的一个分区
例：map，flatMap，还有filter，不涉及Shuffle操作
## 1.6 RDD的Lineage

RDD只支持粗粒度转换，即在大量记录上执行的单个操作。将创建RDD的一系列Lineage（即血统）记录下来，以便恢复丢失的分区。RDD的Lineage会记录RDD的元数据信息和转换行为，当该RDD的部分分区数据丢失时，它可以根据这些信息来重新运算和恢复丢失的数据分区。

1. **Transformation（转换）**，会记录你对RDD做了什么，但是是延迟执行的
2. **Action（动作）** 会触发任务生成，然后将任务提交到集群进行计算，最后产生计算结果


### 常见Transformation和Action

|Transformation|Action|
| ------------ | ------------ |
|map|reduce|
|filter|collect|
|flatMap|count|
|mapPartitions|first|
|mapPartitionsWithIndex|take|
|sample|takeSample|
|union|takeOrdered|
|distinct|saveAsTextFile|
|groupByKey|saveAsSequenceFile|
|reduceByKey|saveAsObjectFile|
|sortByKey|countByKey|
|join|foreach|
|repartition||
|repartitionAndSortWithinPartitions||

[**Transformation和Action**](http://spark.apache.org/docs/latest/rdd-programming-guide.html "Transformation")

# 2. 共享变量
## 2.1 Broadcast-广播变量（只读）

- 广播变量用来把变量在所有节点的内存之间进行共享，在**每个Executor**内存中上缓存一个只读的变量，而不是为机器上的每个任务都生成一个副本；
- 同一进程内资源共享 —— 一个executor内的多个RDD分区共享(本地Executor上的BlockManager中)
- 节省内存和网络IO

> 在算子函数中，使用广播变量时，首先会判断当前task所在Executor内存中，是否有变量副本。<br>
> 如果有则直接使用；如果没有则从Driver或者其他Executor节点上远程拉取一份放到本地Executor上的BlockManager中<br>
> 只需要第一个task使用的时候拉取一次，之后其他task使用就可以复用blockManager中的变量，不需要重新拉取，也不需要在task中保存这个数据结构

## 2.2 Accumulator-累加器（只写）

- task对累加器只写，无法读取累加器的值
- driver 可以读取累加器的值累加器支持在所有不同节点之间进行累加计算（比如计数或者求和）

特性：
1) 累加器能保证在Spark任务出现问题被重启的时候不会出现重复计算
2) 累加器只有在Action执行的时候才会被触发
3) 累加器只能在Driver端定义，在Executor端更新，不能在Executor端定义，不能在Executor端.value获取值

# 3. 分组TopN

1. 将学科与老师组合为一个元组
2. 在聚合时就使用自定义的分区器，将相同学科数据分到同一个分区中，减少shuffle
3. mapPartitions 一次拿出来一个分区（迭代器)

**切分字符串的方法**

        val line = "http://bigdata.edu360.cn/laozhang"
        //方法一
        val fields: Array[String] = line.split("/")
        val host = fields(2)
        val teacher = fields(3)
        val subject = host.split("[.]")(0)
    
        //方法二
        val teacher = line.substring(line.lastIndexOf("/") + 1)
        val url = new URL(line).getHost
        val subject = url.substring(0, url.indexOf("."))

代码

```scala
import java.net.URL

import org.apache.spark.{Partitioner, SparkConf, SparkContext}
import org.apache.spark.rdd.RDD

import scala.collection.mutable

object TeacherTopN3 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("TeacherTopN3").setMaster("local[2]")
    val sc = new SparkContext(conf)
    val lines: RDD[String] = sc.textFile("d:/teacher.log")
    //处理数据 切字符串->映射
    val courseTeacherOne: RDD[((String, String), Int)] = lines.map(lines => {
      val course: String = new URL(lines).getHost.split("[.]")(0)
      val teacher: String = lines.substring(lines.lastIndexOf("/") + 1)
      ((course, teacher), 1)
    })

    //统计学科种类
    val courses: Array[String] = courseTeacherOne.map(_._1._1).distinct().collect()
    //自定义分区器
    val coursePartitioner = new CoursePartitioner1(courses)

    //在聚合阶段就使用自定义的分区器，相同学科数据会到同一个分区中，减少shuffle
    val reduced: RDD[((String, String), Int)] = courseTeacherOne.reduceByKey(coursePartitioner, _ + _)
    //调用map方法是一条一条的处理
    //mapPartitions 一次拿出来一个分区（迭代器)
    val sorted: RDD[((String, String), Int)] = reduced.mapPartitions(_.toList.sortBy(-_._2).iterator)
    val result: Array[((String, String), Int)] = sorted.collect()
    println(result.toBuffer)

  }
}

class CoursePartitioner1(courses: Array[String]) extends Partitioner {
  val rules = new mutable.HashMap[String, Int]()
  //初始化规则
  var i = 0
  for (course <- courses) {
    rules(course) = i
    i += 1
  }

//分组数量=学科数量
  override def numPartitions: Int = courses.length
//根据传入的key，计算返回分区的编号
//定义一个计算规则
  override def getPartition(key: Any): Int = {
    //key要求是一个元组(学科，老师),将key强转成元组
    val tp: (String, String) = key.asInstanceOf[Tuple2[String, String]]
    var course = tp._1
    rules(course)
  }
}
```