[TOC]

# 1. 背景

业务上有一份行车轨迹的数据 carRecord.csv 如下：

```csv
id;carNum;orgId;capTime
1;粤A321;0002;10:20:10
2;云A321;0001;10:20:10
3;粤A321;0001;10:30:10
4;云A321;0002;10:30:10
5;粤A321;0003;11:40:10
6;京A321;0003;11:50:10
```

其中各字段含义分别为记录id，车牌号，抓拍卡口，抓拍时间。现在需要筛选出所有车辆最后出现的一条记录，得到每辆车最后经过的抓拍点信息，也就是要将其他日期的数据过滤掉，我们可以使用选择去重。下面分别展示通过 dataframe 和 rdd 如果实现。

# 2. DataFrame实现

具体实现：

1.  导入行车数据；
2.  首先使用 withColumn() 添加 num 字段，num 字段是由 row\_number() + Window() + orderBy() 实现的：开窗函数中进行去重，先对车牌carNum 进行分组，倒序排序，然后取窗口内排在第一位的则为最后的行车记录，使用 where 做过滤，最后drop掉不再使用的 num 字段；
3.  通过 explain 打印 dataFrame 的物理执行过程，show() 作为 action算子触发了以上的系列运算。


```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.Window
// This import is needed to use the $-notation
import spark.implicits._

val carDF = spark.read.format("csv")
.option("sep", ";")
.option("inferSchema", "true")
.option("header", "true")
.csv(basePath + "/car.csv")

val lastPassCar = carDF.withColumn("num",
   row_number().over(
     Window.partitionBy($"carNum")
           .orderBy($"capTime" desc)
   )
).where($"num" === 1).drop($"num")
lastPassCar.explain()
lastPassCar.show()
```

结果如下：

```scala
// 获得其中每辆车最后经过的卡口等信息
    +---+------+-----+--------+
    | id|carNum|orgId| capTime|
    +---+------+-----+--------+
    |  5|粤A321|    3|11:40:10|
    |  6|京A321|    3|11:50:10|
    |  4|云A321|    2|10:30:10|
    +---+------+-----+--------+
```

# 3. RDD实现

思路：

1.  加载源数据并封装到 CarRecord 样例类中，生成RDD；
2.  首先通过 groupBy 对 数据做分组后生成 RDD\[(String, Iterable\[CarRecord\])\]对象，随即使用 map 对每个 key 对应的多组记录（Iterable\[CarRecord\]）进行reduce操作（maxBy），最后在 maxBy 算子传入一个字面量函数（也可写为x=>x.capTime），即提取该carNum下每条记录中的 capTime 进行比对，然后选出最新时间记录（maxBy 为高阶函数，依赖 reduceLeft 实现）；


```Scala
case class CarRecord(id: Int, carNum: String, orgId: Int, capTime: String)
    
// 构造 schema RDD
val carRDD: RDD[CarRecord] =
    carDF.rdd.map(x => 
        CarRecord(x.getInt(0), x.getString(1), x.getInt(2), x.getString(3)))
val res = carRDD.groupBy(_.carNum).map{
    x => {
        // x._2 是 iter，取其中 capTime 最大的记录
        x._2.maxBy { _.capTime }
    }
}
res.toDebugString
res.collect.foreach(x => println(x))
```

> 优化版<br>
> 将指定列作为key，将源数据转换为一个mapPartionsRDD，然后再reduceByKey进行去重保留最后加入的数。<br>
> `val unRdd = dwellCaseDs.rdd.map(line=>((line.getAs[String]("col1")+line.getAs[String]("col2")),line)).reduceByKey((x,y)=>y,6).map(_._2)`


# 4. 总结

实现选择去重的两种常用方法：

1.  通过开窗函数 row\_number+window+orderBy 进行聚合后去重；
2.  通过 groupby + maxBy 等算子进行聚合后去重。

