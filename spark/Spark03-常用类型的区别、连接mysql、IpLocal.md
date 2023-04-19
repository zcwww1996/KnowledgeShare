[TOC]
# 1. 常用类型的区别
## 1.1 foreach与map的区别

- foreach是一个Action，会触发任务生成，然后将Task提交到集群中执行，foreach没有返回（Unit）
- map是一个Transformation，延迟执行，map方法返回的是RDD

## 1.2 RDD的foreachPartition和MapPartitions的区别
都是一次拿出一个分区，一个分区就是一个迭代器

- foreachPartition是Action，没有返回，输入的函数是什么类型：输入的是Iterator => Unit
- MapPartitions是Transformation。然后的返回RDD 输入的函数是什么类型：输入的是Iterator = > Iterator
一次拿出一个分区，分区是迭代器，如果想通过迭代器获取每一条内容，调用迭代器上的map方法
- MapPartitionsWithIndex 一次获取一个分区，顺带拿出分区对应的编号。可以通过MapPartitionsWithIndex获取每个分区的内容

## 1.3 aggregateByKey
aggregateByKey()()在各个分区叠加的时候加初始值，聚合时不加初始值

[![QJxQoT.png](https://s2.ax1x.com/2019/12/06/QJxQoT.png "aggregateByKey")](https://ae01.alicdn.com/kf/H740cb6cca5cd4776bd46afcefa630ee74.png)

**reduceByKey、foldByKey、aggregateByKey、combineByKey 区别？**

- reduceByKey 没有初始值，分区内和分区间逻辑相同
- foldByKey 有初始值，分区内和分区间逻辑相同
- aggregateByKey 有初始值，分区内和分区间逻辑可以不同
- combineByKey 初始值可以变化结构，分区内和分区间逻辑不同

# 2. 常用工具类
## 2.1 ip转十进制数
ip（String）-> num(Long)
```scala
def ip2Long(ip: String): Long = {
    val fragments = ip.split("[.]")
    var ipNum = 0L
    for (i <- 0 until fragments.length) {
      ipNum = fragments(i).toLong | ipNum << 8L
    }
    ipNum
  }
```
## 2.2 二分法查找
根据ip转换的十进制数，查找对应的城市
```scala
def binarySearch(lines: Array[(Long, Long, String)], ip: Long): Int = {
    var low = 0
    var high = lines.length - 1
    while (low <= high) {
      val middle = (low + high) / 2
      if ((ip >= lines(middle)._1) && (ip <= lines(middle)._2))
        return middle
      if (ip < lines(middle)._1)
        high = middle - 1
      else {
        low = middle + 1
      }
    }
    -1
  }
```
## 2.3 连接mysql，发送数据

```scala
  def data2MySQL(it: Iterator[(String, Int)]) = {
    //一个迭代器代表一个分区，分区中有多条数据
    //先获得一个JDBC连接
    val conn: sql.Connection = DriverManager.getConnection("jdbc:mysql://localhost:3306/bigdata?characterEncoding=UTF-8", "root", "pwd")
    //将数据通过Connection写入到数据库
    val pstm: sql.PreparedStatement = conn.prepareStatement("insert into access_log values(?,?)")
    //将分区中的数据一条一条写入到MySQL
    it.foreach(tp => {
      pstm.setString(1, tp._1)
      pstm.setInt(2, tp._2)
      pstm.executeUpdate()
    })
    //将分区中的数据全部写完之后，在关闭连接
    if (pstm != null) {
      pstm.close()
    }
    if (conn != null) {
      conn.close()
    }
  }
}
```

# 3. IpLocal代码
二分法获取某条ip在ip字典表的位置

```scala
import Util.MyUtil
import org.apache.spark.broadcast.Broadcast
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object IpLocal {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("IpLocal").setMaster("local[2]")
    val sc = new SparkContext(conf)
    val ipLine: RDD[String] = sc.textFile("D:/Test/ip.txt")
    //整理ip数据，获取ip十进制起始、结束界限、省份
    val ipRule: RDD[(Long, Long, String)] = ipLine.map(t => {
      val fields = t.split("[|]")
      val startNum = fields(2).toLong
      val endNum = fields(3).toLong
      val province = fields(6)
      (startNum, endNum, province)
    })

    //将整理好的ip规则收集到Driver
    val allRulesInDriver: Array[(Long, Long, String)] = ipRule.collect()

    //将全部的IP规则通过广播的方式发送到Executor
    //广播之后，在Driver端获取了到了广播变量的引用broadcast(如果没有广播完，就不往下走)
    val broadcast: Broadcast[Array[(Long, Long, String)]] = sc.broadcast(allRulesInDriver)

    val line = sc.textFile("d:/Test/access.log")
    //整理访问日志，获得ip，并将ip转为对应十进制
    val provinceAndOne: RDD[(String, Int)] = line.map(t => {
      val ip: String = t.split("[|]")(1)
      //ip转为十进制
      val iPNum: Long = MyUtil.ip2Long(ip)
      //通过广播变量的引用获取到Executor中的全部IP规则，然后进行匹配ip规则
      val allRuleaInExecutor: Array[(Long, Long, String)] = broadcast.value
      //根据Ip规则，进行省份查找   采用二分法
      var province = "???"
      //index 查找到的ip对应的位置（行数 =角标）
      val index: Int = MyUtil.binarySearch(allRuleaInExecutor, iPNum)
      if (index != -1) {
        province = allRuleaInExecutor(index)._3
      }
      (province, 1)
    })
    //按照省份访问次数进行计数
    val provinceAndNum: RDD[(String, Int)] = provinceAndOne.reduceByKey(_ + _)
    val result: RDD[(String, Int)] = provinceAndNum.sortBy(-_._2)
//    val tp = result.collect()
//    println(tp.toBuffer)

    //计算结果，将计算好的结果写入到MySQL中
    //触发一个Action，将数据写入到MySQL的逻辑通过函数传入
    //利用foreachParatition,一次取出一个分区（迭代器），减少jdbc连接次数
    result.foreachPartition(MyUtil.data2MySQL)
    sc.stop()
  }
}
```