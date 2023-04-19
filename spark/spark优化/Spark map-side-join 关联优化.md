[TOC]
# 1. 何时使用

在海量数据中匹配少量特定数据
# 2. 原理

关于spark-sql中利用broadcast join进行优化的文章，原理相同</br>
http://blog.csdn.net/lsshlsw/article/details/48694893

reduce-side-join 的缺陷在于会将key相同的数据发送到同一个partition中进行运算，大数据集的传输需要长时间的IO，同时任务并发度收到限制，还可能造成数据倾斜。

reduce-side-join 运行图如下:

![reduce-side-join](https://c-ssl.duitang.com/uploads/item/201911/25/20191125113637_fXRP3.thumb.700_0.png "reduce-side-join")


map-side-join 运行图如下:

![map-side-join](https://c-ssl.duitang.com/uploads/item/201911/25/20191125113637_RuGZf.thumb.700_0.png "map-side-join")

# 3. 代码说明
数据1（个别人口信息）:
> 身份证 姓名 ...
> 
> 110   lsw
> 
> 222   yyy

数据2（全国学生信息）:
> 身份证 学校名称 学号 ... 
>
> 110   s1      211
>
> 111   s2      222
>
> 112   s3      233
>
> 113   s2      244

期望得到的数据 :
> 身份证 姓名 学校名称
> 
> 110 lsw s1

## 3.1 广播变量
将少量的数据转化为Map进行广播，广播会将此 Map 发送到每个节点中，如果不进行广播，每个task执行时都会去获取该Map数据，造成了性能浪费。

```scala
val people_info = sc.parallelize(Array(("110","lsw"),("222","yyy"))).collectAsMap()
val people_bc = sc.broadcast(people_info)
```

对大数据进行遍历，使用mapPartition而不是map，因为mapPartition是在每个partition中进行操作，因此可以减少遍历时新建broadCastMap.value对象的空间消耗，同时匹配不到的数据也不会返回（）。

```scala
val res = student_all.mapPartitions(iter =>{
    val stuMap = people_bc.value
    val arrayBuffer = ArrayBuffer[(String,String,String)]()
    iter.foreach{case (idCard,school,sno) =>{
        if(stuMap.contains(idCard)){
        arrayBuffer.+= ((idCard, stuMap.getOrElse(idCard,""),school))
    }
    }}
    arrayBuffer.iterator
})
```
## 3.2 守卫机制
也可以使用for的守卫机制来实现上述代码

```scala
val res1 = student_all.mapPartitions(iter => {
    val stuMap = people_bc.value
    for{
        (idCard, school, sno) <- iter
        if(stuMap.contains(idCard))
        } yield (idCard, stuMap.getOrElse(idCard,""),school)
})
```

## 3.3 完整代码
```scala
import org.apache.spark.{SparkContext, SparkConf}
import scala.collection.mutable.ArrayBuffer

object joinTest extends App{

  val conf = new SparkConf().setMaster("local[2]").setAppName("test")
  val sc = new SparkContext(conf)

  /**
   * map-side-join
   * 取出小表中出现的用户与大表关联后取出所需要的信息
   * */
  //部分人信息(身份证,姓名)
  val people_info = sc.parallelize(Array(("110","lsw"),("222","yyy"))).collectAsMap()
  //全国的学生详细信息(身份证,学校名称,学号...)
  val student_all = sc.parallelize(Array(("110","s1","211"),
                                              ("111","s2","222"),
                                              ("112","s3","233"),
                                              ("113","s2","244")))

  //将需要关联的小表进行关联
  val people_bc = sc.broadcast(people_info)

  /**
   * 使用mapPartition而不是用map，减少创建broadCastMap.value的空间消耗
   * 同时匹配不到的数据也不需要返回（）
   * */
  val res = student_all.mapPartitions(iter =>{
    val stuMap = people_bc.value
    val arrayBuffer = ArrayBuffer[(String,String,String)]()
    iter.foreach{case (idCard,school,sno) =>{
      if(stuMap.contains(idCard)){
        arrayBuffer.+= ((idCard, stuMap.getOrElse(idCard,""),school))
      }
    }}
    arrayBuffer.iterator
  })

  /**
   * 使用另一种方式实现
   * 使用for的守卫
   * */
  val res1 = student_all.mapPartitions(iter => {
    val stuMap = people_bc.value
    for{
      (idCard, school, sno) <- iter
      if(stuMap.contains(idCard))
    } yield (idCard, stuMap.getOrElse(idCard,""),school)
  })

  res.foreach(println)
```
