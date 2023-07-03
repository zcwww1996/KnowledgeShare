[TOC]

# 1. 累加器

Accumulator是spark提供的累加器，顾名思义，该变量只能够增加。

只有driver能获取到Accumulator的值（使用value方法），Task只能对其做增加操作（使用 +=）。你也可以在为Accumulator命名（不支持Python），这样就会在spark web ui中显示，可以帮助你了解程序运行的情况。

## 1.1 Accumulator使用
### 1.1.1 简单使用
```Scala
//在driver中定义

val accum = sc.accumulator(0, "Example Accumulator")

//在task中进行累加

sc.parallelize(1 to 10).foreach(x=> accum += 1)

//在driver中输出

accum.value

//结果将返回10

res: 10
```
### 1.1.2 使用Accumulator的陷阱

```
object AccumulatorTrapDemo {
  def main(args: Array[String]): Unit = {
    val session = SparkSession.builder.master("local[2]").getOrCreate
    val sc = session.sparkContext
    sc.setLogLevel("WARN")
    val longAccumulator = sc.longAccumulator("long-account")

    // ------------------------------- 在transform算子中的错误使用 -------------------------------------------
    import session.implicits._
    val num1: Dataset[Int] = session.createDataset(List(1, 2, 3, 4, 5))
    val nums2 = num1.map(x => {
      longAccumulator.add(1)
      x
    })

    // 因为没有Action操作，nums.map并没有被执行，因此此时累加器的值还是0
    println("num2 1: " + longAccumulator.value) // 0

    // 调用一次action操作，num.map得到执行，广播变量被改变
    nums2.count
    println("num2 2: " + longAccumulator.value) // 5

    // 又调用了一次Action操作，累加器所在的map又被执行了一次，所以累加器又被累加了一遍，就悲剧了
    nums2.count
    println("num2 3: " + longAccumulator.value) // 10


    // ------------------------------- 在transform算子中的正确使用 -------------------------------------------

    // 累加器不应该被重复使用，或者在合适的时候进行cache断开与之前Dataset的血缘关系，因为cache了就不必重复计算了
    longAccumulator.reset()
    val nums3 = num1.map(x => {
      longAccumulator.add(1)
      x
    }).cache // 注意这个地方进行了cache
    println("num3 1: " + longAccumulator.value) // 未执行action，0

    // 调用一次action操作，累加器被改变
    nums3.count
    println("num3 2: " + longAccumulator.value) // 5

    // 又调用了一次action操作，因为前一次调用count时num3已经被cache，num1.map不会被再执行一遍，所以这里的值还是5
    nums3.count
    println("num3 3: " + longAccumulator.value) // 5


    // ------------------------------- 在action算子中的使用 -------------------------------------------
    longAccumulator.reset()
    num1.foreach(x => {
      longAccumulator.add(1)
    })
    // 因为是action操作，会被立即执行所以打印的结果是符合预期的
    println("num4: " + longAccumulator.value) // 5

    session.stop()
  }
}
```

==**使用Accumulator时，为了保证准确性，只使用一次action操作。如果需要使用多次则使用cache或persist操作切断依赖。**==

### 1.1.3 Accumulator使用的奇淫技巧

累加器并不是只能用来实现加法，也可以用来实现减法，直接把要累加的数值改成负数就可以了：

使用累加器实现减法
```Scala
  longAccumulator.reset()
    num1.foreach(x => {
      if (x % 3 == 0) longAccumulator.add(-2)
      else longAccumulator.add(1)
    })
    println("longAccumulator: " + longAccumulator.value) // 0
```


## 1.2 自定义累加器（Accumulator）

自定义累加器，可以任意累加不同类型的值，同时也可以在内部进行计算，或者逻辑编写，如果继承自定义累加器，那么需要实现内部的抽象方法，然后在每个抽象方法内部去累加变量值即可，主要是在全局性累加起到决定性作用。

累加器作为spark的一个共享变量的实现，在用于累加计数计算计算指标的时候可以有效的减少网络的消耗

累加器可以在每个节点上面进行Task的值，累加操作，有一个Task的共享性操作

新版累加器使用步骤： 1. 创建累加器 2. 注册累加器 3. 使用累加器

1、类继承extends AccumulatorV2\[String, String\]，第一个为输入类型，第二个为输出类型<br>
2、覆写抽象方法：

- `isZero`: 当AccumulatorV2中存在类似数据不存在这种问题时，是否结束程序。
- `copy`: 拷贝一个新的AccumulatorV2  
- `reset`: 重置AccumulatorV2中的数据  
- `add`: 操作数据累加方法实现  
- `merge`: 合并数据  
- `value`: AccumulatorV2对外访问的数据结果

### 1.2.1 元组累加器

```scala
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.util.AccumulatorV2
//在继承的时候需要执行泛型 即 可以计算IN类型的输入值，产生Out类型的输出值
//继承后必须实现提供的方法
class MyAccumulator extends  AccumulatorV2[Int,Int]{
  //创建一个输出值的变量
  private var sum:Int = _ 

  //必须重写如下方法:
  //检测方法是否为空
  override def isZero: Boolean = sum == 0
  //拷贝一个新的累加器
  override def copy(): AccumulatorV2[Int, Int] = {
    //需要创建当前自定累加器对象
    val myaccumulator = new MyAccumulator()
    //需要将当前数据拷贝到新的累加器数据里面
   //也就是说将原有累加器中的数据拷贝到新的累加器数据中
    //ps:个人理解应该是为了数据的更新迭代
    myaccumulator.sum = this.sum
    myaccumulator
  }
  //重置一个累加器 将累加器中的数据清零
  override def reset(): Unit = sum = 0
  //每一个分区中用于添加数据的方法(分区中的数据计算)
  override def add(v: Int): Unit = {
    //v 即 分区中的数据
     //当累加器中有数据的时候需要计算累加器中的数据
     sum += v
  }
  //合并每一个分区的输出(将分区中的数进行汇总)
  override def merge(other: AccumulatorV2[Int, Int]): Unit = {
          //将每个分区中的数据进行汇总
            sum += other.value

  }
 //输出值(最终累加的值)
  override def value: Int = sum
}

object  MyAccumulator{
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("MyAccumulator").setMaster("local[*]")
    //2.创建SparkContext 提交SparkApp的入口
    val sc = new SparkContext(conf)
    val numbers = sc .parallelize(List(1,2,3,4,5,6),2)
    val accumulator = new MyAccumulator()
    //需要注册 
    sc.register(accumulator,"acc")
    //切记不要使用Transformation算子 会出现无法更新数据的情况
    //应该使用Action算子
    //若使用了Map会得不到结果
    numbers.foreach(x => accumulator.add(x))
    println(accumulator.value)
  }
}

```

### 1.2.2 map累加器


```Scala
import org.apache.spark.util.AccumulatorV2

import scala.collection.mutable

/**
 * map类型累加器
 */
class MapAccumulator extends AccumulatorV2[String, mutable.HashMap[String, Int]] {
  private val hashMap = new mutable.HashMap[String, Int]()

  /**
   * 数据初始化
   *
   * @return 判断是否为空
   */
  override def isZero: Boolean = hashMap.isEmpty

  /**
   * 复制一个新的对象
   *
   * @return 返回新的累加器
   */
  override def copy(): AccumulatorV2[String, mutable.HashMap[String, Int]] = {
    val acc = new MapAccumulator()
    hashMap.synchronized {
      acc.hashMap ++= hashMap
    }
    acc
  }

  /**
   * 重置每个分区的数值
   */
  override def reset(): Unit = hashMap.clear

  /**
   * 各分区按指定key进行累加，key存在更新，key不存在插入
   *
   * @param k 需要累计的key
   */
  override def add(k: String): Unit = {
    hashMap.get(k) match {
      case Some(v) => hashMap += ((k, v + 1))
      case None => hashMap += ((k, 1))
    }

  }

  /**
   * 把所有分区的map进行合并，求得总值
   *
   * @param other 其他分区的map累加器
   */
  override def merge(other: AccumulatorV2[String, mutable.HashMap[String, Int]]): Unit = {
    other match {
      case acc: AccumulatorV2[String, mutable.HashMap[String, Int]] =>
        for ((k, v) <- acc.value) {
          hashMap.get(k) match {
            case Some(x) => hashMap += ((k, x + v))
            case None => hashMap += ((k, v))
          }
        }
    }
  }

  /**
   * 返回结果
   *
   * @return
   */
  override def value: mutable.HashMap[String, Int] = hashMap
}
```


总结:

1. 累加器的创建

```bash
1.1.创建一个累加器的实例 

1.2.通过sc.register()注册一个累加器

1.3.通过累加器实名.add来添加数据

1.4.通过累加器实例名.value来获取累加器的值
```

2. 最好不要在转换操作中访问累加器(因为血统的关系和转换操作可能执行多次),最好在行动操作中访问

**累加器作用:**

1. 能够精确的统计数据的各种数据，例如:<br>
可以统计出符合userID的记录数,在同一个时间段内产生了多少次购买,可以使用ETL进行数据清洗,并使用Accumulator来进行数据的统计

2. 作为调试工具,能够观察每个task的信息,通过累加器可以在sparkIUI观察到每个task所处理的记录数

# 2. 自定义排序

## 2.1 自定义多重排序
**自定义多重排序规则，先按字符串字典排序，再根据数字大小排序**

```scala
class SeveralSortKey(val first: String, val second: String, val third: String) extends Ordered[SeveralSortKey] with Serializable {
  //重写Ordered类的compare方法
  def compare(other: SeveralSortKey): Int = {
    if (!this.first.equals(other.first)) {
      this.first.compareTo(other.first)
    } else if (!this.second.equals(other.second)) {
      -(this.second.toInt - other.second.toInt)
    } else {
      -(this.third.toInt - other.third.toInt)
    }
  }
}
```

```Scala
    val mapRDD: RDD[(SeveralSortKey, String)] = result_noMerge.map(arr => {
      // val arr = line.split("\\|")
      (new SeveralSortKey(arr(2), arr(1), arr(0)), arr.mkString("|"))
    })
    // sortByKey(true)是按升序排列，可以在自定义排序类里面通过this与other的加减值控制每一项排序，sortByKey(true)时，this-other为升序
    // 调用sortByKey的RDD一定要是K-V形式
    val sorted: RDD[(SeveralSortKey, String)] = mapRDD.sortByKey(false)
    val finalResult = sorted.map(t => t._2)
```

## 2.2 常用自定义排序
比如有数据“name，age，颜值”，我们想按照颜值降序，颜值相同按照年龄升序排。

参考：https://blog.csdn.net/lv_yishi/article/details/83933391
### 2.2.1 方法一：自定义类，重写这个类的compare方法
思路：写个类，将切分出来的数据放入到这个类中，重写这个类的compare方法，用这个类调用sortBy方法。

```Scala
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}
/**
  * 实现自定义排序
  */
object CustomSort1 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("IpLocation").setMaster("local[*]")
    val sc: SparkContext = new SparkContext(conf)
    //排序规则，先按颜值降序，相等再按年龄升序
    val users=Array("laoduan 30 99","laozhao 29 9999", "laozhang 28 98", "laoyang 28 99")
    //将driver的数据并行化变成 RDD
    val lines  = sc.parallelize(users)
    //切分整理数据，将需要的数据切分出来，放到一个类中，
    //这个类继承ordered，然后重写compare，这个类在调用相应的排序方法的时候就会执行重写后的排序规则
    val userRDD: RDD[User] = lines.map(line => {
      val fields: Array[String] = line.split(" ")
      val name: String = fields(0)
      val age: Int = fields(1).toInt
      val fv: Int = fields(2).toInt
      new User(name, age, fv)
    })
    //u => u表示不传入排序规则，即会调用这个类重写的排序规则
    val sorted: RDD[User] = userRDD.sortBy(u => u)
    println(sorted.collect().toBuffer)
  }
 
  class User(val name:String,val age:Int,val fv:Int)extends Ordered[User] with Serializable {
    override def compare(that: User): Int = {
      if(this.fv == that.fv){
        this.age - that.age // 按照age升序
      }else{
        -(this.fv - that.fv) // 按照fv降序
      }
    }
    override def toString: String = s"name:$name,age:$age,facavalue:$fv"
  }
}

```
### 2.2.2 方法二，自定义排序规则<font style="color: red;">★★★</font>
传入一个排序规则，重写Ordered的compare方法。<br>
自定义的object在运行时会shuffle有网络传输会涉及序列化的问题。所以需要同时继承Serializable

对比方法一，就是new的位置不一样了

```Scala
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}
 
object CustomSort2 {
  def main(args: Array[String]): Unit = {
    val conf = new SparkConf().setAppName("IpLocation").setMaster("local[*]")
    val sc: SparkContext = new SparkContext(conf)
    val users = Array("nanyang 25 99","thy 26 9999", "mingming 28 98", "wazi 22 99")
    val lines: RDD[String] = sc.parallelize(users)
    
    val tpRDD: RDD[(String, Int, Int)] = lines.map(line => {
      val fields: Array[String] = line.split(" ")
      val name: String = fields(0)
      val age: Int = fields(1).toInt
      val fv: Int = fields(2).toInt
      (name, age, fv)
    })
    
    // 传入一个排序规则，不改变数据格式，只改变顺序
    val sorted: RDD[(String, Int, Int)] = tpRDD.sortBy(tp => new MySort(tp._2,tp._3))
    println(sorted.collect().toBuffer)
    sc.stop()
  }
  //排序规则，将age和fv两个排序用到的属性传入，然后进行重写compare方法，当调用sortBy时，new一个本规则传入相应比较属性即可实现排序。
  class MySort(val age:Int,val fv:Int)extends Ordered[MySort] with Serializable {
    override def compare(that: MySort): Int = {
      if(this.fv == that.fv){
        this.age - that.age
      }else{
        -(this.fv - that.fv)
      }
    }
  }
}

```

### 2.2.3 方法三：使用case class
case class与class区别：<br>
case class 默认实现了序列化，不用再 `with Serializable`<br>
排序后的rdd输出不能使用foreach
```Scala
object CustomSort3 {
  def main(args: Array[String]): Unit = {
   
  /**
   * 此段代码同上 tpRDD为：(name,age,fv)
   * (String, Int, Int)
   */
    
    val sorted: RDD[(String, Int, Int)] = tpRDD.sortBy(tp => MySort(tp._2,tp._3))
    // sorted.foreach(println)
    //不能使用foreach
    println(sorted.collect().toBuffer)
  }
  //case class 默认实现了序列化，不用在 with seria...
  case  class MySort(age:Int,fv:Int) extends Ordered[MySort]{
    override def compare(that: MySort): Int = {
      if(this.fv == that.fv){
        this.age - that.age
      }else{
        -(this.fv - that.fv)
      }
    }
  }
}

```

### 2.2.4 方法四：利用隐式转换，定义排序规则
单独写个SortRules.scala，继承ordering，重写比较规则。排序时import SortRules.类，


```
利用隐式转换时，类可以不实现Ordered的特质，普通的类或者普通的样例类即可。
隐式转换支持，隐式方法，隐式函数，隐式的object和隐式的变量，
如果都同时存在，优先使用隐式的object，隐式方法和隐式函数中，会优先使用隐式函数。
隐式转换可以写在任意地方（当前对象中，外部的类中，外部的对象中），如果写在外部，需要导入到当前的对象中即可。
```


```scala
import com.thy.d20190417.CustomSort4.MySort
 
object SortRules {
    //隐式的object方式
      implicit object OrderMySort extends Ordering[MySort]{
        override def compare(x: MySort, y: MySort): Int = {
          if(x.fv == y.fv) {
            x.age - y.age
          } else {
            y.fv - x.fv
          }
        }
      }
    // 如果类没有继承 Ordered 特质
    // 可以利用隐式转换  隐式方法  隐式函数  隐式值  隐式object都可以  implicit ord: Ordering[K]
    implicit def ordMethod(p: MySort): Ordered[MySort] = new Ordered[MySort] {
      override def compare(that: MySort): Int = {
        if (p.fv == that.fv) {
          -(p.age - that.age)
        } else {
          that.fv - p.fv
        }
      }
    }
 
    //利用隐式的函数方式
    implicit val ordFunc = (p: MySort) => new Ordered[MySort] {
      override def compare(that: MySort): Int = {
        if (p.fv == that.fv) {
          -(p.age - that.age)
        } else {
          that.fv - p.fv
        }
      }
    }

}
 
//-----------------------------------------------------------
 
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.rdd.RDD
 
object CustomSort4 {
  def main(args: Array[String]): Unit = {

  /**
   * 此段代码同上 tpRDD为：(name,age,fv)
   * (String, Int, Int)
   */
      
    import SortRules.OrderingMySort
    // 导入隐式对象OrderMySort
    val sorted: RDD[(String, Int, Int)] = tpRDD.sortBy(tp => MySort(tp._2,tp._3))
    println(sorted.collect().toBuffer)
    sc.stop()
  }
  case class MySort(age:Int, fv: Int)
}

```

### 2.2.5 方法五：利用元组的排序特性即可
元组的比较规则：先比第一，相等再比第二个
```
//先降序比较fv，在升序比较age
val sorted: RDD[(String, Int, Int)] = tpRDD.sortBy(tp => (-tp._3,tp._2))
/**
    其他代码同上
*/

```

### 2.2.6 方法六：利用Ordering的on方法

无需借助任何的类或者对象

只需要利用Ordering特质的on方法即可

```scala
 // Ordering[(Int, Int)] 最终比较的规则格式
 // on[(String, Int, Int)] 未比较之前的数据格式
 // (t =>(-t._3, t._2)) 怎样将规则转换成想要比较的格式
   implicit val rules: Ordering[(String, Int, Int)] = Ordering[(Int,Int)].on[(String,Int,Int)](t => (-t._3,t._2))
   val sorted: RDD[(String, Int, Int)] = tpRDD.sortBy(tp =>tp)

```

原文链接：https://blog.csdn.net/weixin\_39043567/article/details/89354891

# 3. 自定义分区
## 3.1 先自定义分区，再在分区内自定义排序

**主函数**
```Scala
object CustomPartition {

   val conf: SparkConf = new SparkConf().setAppName("test1107_2").setMaster("local[2]")
   val sc = new SparkContext(conf)
   val dataRdd = sc.textFile("F:\\test\\test\\ssort.txt")

   // 待分区的RDD必须是K-V格式
    val source: RDD[(String, Array[String])] = dataRdd.map { x =>
      val arr = x.split(" ")
      (arr(0), arr)
    }
    
     //先分区, 再在分区内排序
    val finalResult: RDD[String] = source.partitionBy(new UserPartitioner(2)).mapPartitions(iter => {
      iter.map(t => (new SeveralSortKey(t._2(2), t._2(1), t._2(0)), t._2.mkString("|"))).toList.sortBy(t => t._1).reverse.iterator
    }).map(t => t._2)
  
  // 获取每个分组的top3
    val topN: RDD[String] = source.partitionBy(new UserPartitioner(2)).mapPartitions(iter => {
      iter.map(t => (new SeveralSortKey(t._2(2), t._2(1), t._2(0)), t._2.mkString("|"))).toList.sortBy(t => t._1).reverse.take(3).iterator
    }).map(t => t._2)
  println(topN.collect().toBuffer)
  
  // topN.saveAsTextFile("hdfs://server/topN")
  sc.stop()
}

```

程序中还是会出现一个问题，当数据量太大的时候可能会导致内存溢出的情况，因为我们是将数据放到了list中进行排序，而list是存放于内存中。所以会导致内存溢出。那么怎么才能避免这个情况呢。我们可以在mapPartitions内部定义一个集合，不加载所有数据。，每次将这个集合排序后最小的值移除，通过多次循环后最终集合中剩下的就是需要的结果

> **可以使用spark新算子`repartitionAndSortWithinPartitions`解决排序内存溢出问题**

**自定义分区器**
```scala
import org.apache.spark.Partitioner

/**
 * 自定义的partition分组函数
 */
class UserPartitioner(numParts: Int) extends Partitioner {

  override def numPartitions: Int = numParts

  override def hashCode: Int = numPartitions

  /**
   * 根据用户ID、HDFS所在目录的文件数，获取的用户的分区数量
   * @param key 用户的UserID ,不是IMSI
   * @return 返回对应的分区数
   * @note HDFS目录的文件数，是指总的文件数量；从下标为1开始记数
   */
  override def getPartition(key: Any): Int =
    key match {
      case key: String => nonSSNegativeMod(key.toLong, numPartitions)
      case key: Long => nonSSNegativeMod(key, numPartitions)
    }

  /**
   * 获得用户所在文件编号
   * @param x 用户的USERID编号
   * @param mod 对应HDFS的文件总数
   * @return 用户USERID对应文件编号
   */
  protected def nonSSNegativeMod(x: Long, numPartitions: Int): Int = {
    val v: Long = if (x < 0) -x else x
    // Math.abs(v.hashCode() % numPartitions)
    (v % numPartitions).toInt
  }

  /**
   * 判断分区是否相同
   * @param other 对比的分区
   * @return True OR False
   */
  override def equals(other: Any): Boolean =
    other match {
      case user: UserPartitioner =>
        user.numPartitions == numPartitions
      case _ =>
        false
    }

}
```

## 3.2 <font style="color: red;">repartitionAndSortWithinPartitions</font>
repartitionAndSortWithinPartitions 主要是通过给定的分区器，将相同KEY的元素发送到指定分区，并根据KEY 进行排排序。<br>
*==Tips: 我们可以按照自定义的排序规则，进行二次排序。==*

官方建议，如果需要在repartition重分区之后，还要进行sort 排序，建议直接使用repartitionAndSortWithinPartitions算子。因为 ==**该算子可以一边进行重分区的shuffle操作，一边进行排序。**== shuffle与sort两个操作同时进行，比先shuffle再sort来说，性能可能是要高的。 

**主函数**
```Scala
  // 创建一个包含字符串键和整数值的 RDD
    val data: RDD[(String, Int)] = sc.parallelize(Seq(
      ("apple", 4), ("banana", 2), ("orange", 1), ("banana", 5),  ("watermelon", 3),
      ("kiwi", 1), ("strawberry", 3), ("coconut", 2), ("olive", 3)
    ))


    // import test1.customComparator
    /**
     * key怎么排序，在这里定义
     * 为什么在这里声明一个隐式变量呢，是因为在源码中，方法中有一个隐式参数；不设置是按照默认的排序规则进行排序；
     * 如果隐士变量写在主类外部，需要导入到当前的对象中。import [object对象名].[隐式变量名]
     */
    // 定义一个自定义比较器，按照数字的大小降序排列，相同再按字符串升序
    implicit val customComparator: Ordering[(String, Int)] = new Ordering[(String, Int)] {
      override def compare(a: (String, Int), b: (String, Int)): Int = {
        if (a._2 == b._2) {
          a._1.compareTo(b._1)
        } else {
          -a._2.compareTo(b._2)
        }
      }
    }
    
    // 通过自定义分区器和自定义比较器进行 repartitionAndSortWithinPartitions 操作
    // data是K-V类型的RDD，把需要分区和排序的元素都要放到RDD的K中
    val numPartitions = 3
    val sorted: RDD[(String, Int)] = data.map(t => (t, t)).repartitionAndSortWithinPartitions(new CustomPartitioner(numPartitions)).map(_._2)
    
    
    
    // 按分区打印结果
    // ----------------------------------
    // 使用mapPartitionsWithIndex方法对每个分区的数据应用一个函数
    val partitionedRDD = sorted.mapPartitionsWithIndex((index, iter) => {
      // 将当前分区的数据转换为列表
      val partitionList = iter.toList
      // 返回包含当前分区编号和分区内容的元组
      Iterator((index, partitionList))
    })

    // 整合所有分区的数据到一个数组中
    val allData = partitionedRDD.glom().collect()

    // 遍历数组，打印分区编号和分区内容
    allData.foreach(partition => {
      partition.foreach(println)
    })
```

结果数据

```scala
// ----------------------
(0,List((olive,3)))
(1,List((watermelon,3), (coconut,2), (kiwi,1)))
(2,List((banana,5), (apple,4), (strawberry,3), (banana,2), (orange,1)))
```



**自定义分区器**

`getPartition(key: Any)`方法需要注意，**传入的key是K-V元组的K，需要将`key: Any`（本身是Any类型）转换成其他基本类型，从中获取分区依赖的partitionKey**

> 这里是把`key: Any`转换成了`Tuple2[String, Int]`，以元组第一个元素为分区key

```Scala
import org.apache.spark.Partitioner

    /**
     * 定义一个自定义分区器，它使用key的哈希值将数据分区
     */
    class CustomPartitioner(number: Int) extends Partitioner {

      override def numPartitions: Int = number

      override def hashCode: Int = numPartitions

      /**
       * 根据每条数据key的哈希值与自定义分区数量，获取每条数据的所在分区
       *
       * @param tupleKey 包含分区key的元组
       * @return 返回对应的分区数
       */
      override def getPartition(tupleKey: Any): Int = {
        val tuple: (String, Int) = tupleKey.asInstanceOf[(String, Int)]
        val partitionKey = tuple._1
        nonNegativeMod(partitionKey, numPartitions)
      }

      /**
       * 获得key所在分区编号
       *
       * @param partitionKey  分区需要的key
       * @param numPartitions 自定义分区数
       */
      protected def nonNegativeMod(partitionKey: String, numPartitions: Int): Int = {
        val hash = partitionKey.hashCode % numPartitions
        if (hash < 0) hash + numPartitions else hash
      }

      /**
       * 判断分区是否相同
       *
       * @param other 对比的分区
       * @return True OR False
       */
      override def equals(other: Any): Boolean =
        other match {
          case third: CustomPartitioner =>
            third.numPartitions == numPartitions
          case _ =>
            false
        }

    }
```
