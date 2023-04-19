[TOC]

最近在测试UDAF性能的时候，本来以为UDAF的原理和aggregateByKey算子底层算法相同，但是实际在性能测试发现时才发现多了很多额外的开销，导致性能有一定的下降。后来发现Spark 3.0提供的实现方式才是合理的。基于此，就好好地去了解了一下Spark 3.0。  
在Spark 3.0中，引入了一些新的函数，并且对Spark 2.x性能做了优化。下文会详细介绍。  
# AQE  
AQE即为Adaptive Query Execution。Spark在执行之前会先生成DAG图，生成执行计划，AQE就是在执行时，在对这个执行计划根据统计数据重新做进一步的优化。  
引入AQE之后的执行流程如下:  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503825221-7336050c-5df6-4abe-9014-e98a856efe04.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
在Spark 3.0引入AQE之后，性能有了很大的提升。  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503825446-7cb4c847-3e5e-45a1-bb0a-2ddc36e961ba.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
## 动态coalesc分区  
Spark 2.x中，如果我们设置了一个很大的分区数量，那么每个分区数据处理的数据量不多，就会降低性能。在Spark 3.0之后，AQE会自动将小分区合并到大分区。这样能一定程度上能够提升性能。  
没有动态coalesc分区的时候如下:  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503825725-bee514ab-9269-4240-9c3f-b190a2cd5795.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
有动态coalesc分区的时候如下:  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503826022-02b56cc7-cc2d-4ed3-a099-bcfa66cf78a2.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
## 动态切换join策略  
在Spark 2.x中，broadcast-hash join只能通过参数控制。但这个参数不太好控制。在Spark 3.x之后，AQE能够根据运行时的统计信息自动将sort-merge join切换到broadcast-hash join。  
应用动态切换join策略之后的改变如下:  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503826283-63f701e6-4d42-49d0-9683-ae02ef6bb491.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
## 动态优化数据倾斜的join  
在Hive中可以通过参数控制数据倾斜的join，本质上就是先加盐后join。但Spark 2.x中没有这个功能，我们每次都需要手动处理数据倾斜问题。在Spark 3.x之后，可以自动将倾斜的分区分成一个个小的分区去进行join。极大优化了性能。  
没有优化数据倾斜的时候如下:  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503826572-414156f0-5d49-4bc1-92c1-85452330972d.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
有优化数据倾斜的时候如下:  
  

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503826813-bba73170-c7b0-4408-a3cf-5ca9ed8a9d99.png?x-oss-process=image%2Fresize%2Cw_700%2Climit_0)

  
## 动态裁剪分区  
在行星模型中，Spark 2.x的优化器很难在编译时就确定哪些分区可以跳过不读。这就会导致读了一些不需要的数据。但Spark 3.0之后，Spark会首先过滤维表，根据过滤后的结果找到只需要读事实表的哪些分区。这样就能极大提升性能。在TPC-DS基准性能测试中，102中的60个SQL性能提升了2倍到18倍，性能得到了显著提升。

# UDAF性能优化

在Spark 2.x中，实现UDAF有两种方式，一种是实现**UserDefinedAggregationFunction**，另一种是实现**Aggregator**。但是实**现Aggregator**的UDAF无法被注册。  

前者适应于类型不固定的数据，而后者适应于类型固定的数据。所以在前者的实现中，input/output/buffer都是Row。而后者则是定义好的类型。  

如果是处理基础类型，没有什么问题，但是如果要处理复杂类型，比如Map，那么要先放到Row中，显然不如直接更新本地变量要快。  

在我们的性能测试中，如果用Row，那么函数执行1s，有250ms的时间在读Row中的map和把map写到Row中。但是换成Aggregator则没有这个开销。

在Spark 3.0中，基于**Aggregator**的实现才能被注册，性能有了大幅提升。

更详细的对比可以在这篇文章里面找到:UDAF and Aggregators: Custom Aggregation Approaches for Datasets in Apache Spark

# 用于DataFrame转换的新函数
## from_csv 
和Spark 2.x中的from_json差不多，这个函数会将一个CSV字符串转换成一个Struct结构。如果这个CSV字符串无法被解析，就会返回null。  
用法如下:  

```scala
val exployeeInfo = "1,张三,前端" :: "2,李四,后端" :: "3,王五,计算端" :: Nil
val schema = new StructType().add("IdCard", IntegerType).add("Name", StringType).add("DeptName", StringType)
val options = Map("delimiter" -> ",")
val exployeeDF = exployeeInfo.toDF("Exployee_Info").withColumn("struct_csv", from_csv('Exployee_Info, schema, options))
exployeeDF.show()
```

## to_csv  
和from_csv相反，将一个Struct类型的数据转换为CSV字符串。  
用法如下：  

```scala
exployeeDF.withColumn("csv_string",to_csv($"struct_csv",Map.empty[String, String].asJava)).show
```

## schema_of_csv  
查询CSV字符串的schema。  
用法如下：  

```scala
exployeeDF.withColumn("schema",schema_of_csv("csv_string")).show
```

## forall  
对数组中的每个元素都执行特定操作，如果全部都成功就返回true，否则返回false.  
用法如下，下面这个例子检查每个元素是否为奇数：  

```scala
val  df = Seq(Seq(2,4,6),Seq(5,10,3)).toDF("integer_array")
df.withColumn("number",forall($"integer_array",(x:Column)=>(lit(x%2==1)))).show
```

## transform  
对数组中的每个元素都执行特定操作，返回一个新的数组。  
用法如下，下面这个例子将数组中每个数字都乘2：  

```scala
val df = Seq((Seq(1,2,3)),(Seq(4,5,6))).toDF("integer_array")
df.withColumn("integer_array",transform($"integer_array",x=>x*2)).show
```

## overlay  
用于替换某一列的数据，将指定位置的数据替换成特定数据。  
用法如下，下面这个例子将Hello World换成Hello 张三：  

```scala
 val message = "Hello World"::Nil
 val messageDF = greetingMsg.toDF("message")
 messageDF.withColumn("message",overlay($"message",lit("World"),lit("7"),lit("12"))).show
```

## split  
按照给定正则对字符串进行拆分。  
用法如下，下面这个例子将张三，李四一个字符串拆分为张三和李四两个字符串:  

```scala
val names = "张三,李四"
val namesDF = names.toDF("names")
namesDF.withColumn("names", split($"names", ",", 2)).show()
```

## map_entries  
类似于Java中的map.entrySet()函数。  
用法如下:  

```scala
val df = Seq(Map(1->"张三",2->"李四")).toDF("id_name")
df.withColumn("id_name_entry", map_entries($"id_name")).show()
```

## map_zip_with  
用指定函数将两个map合并成一个。这个算子特别有用。之前将两个map相同key合并到一起要写好长一段代码，就像老太太的裹脚布，又臭又长。  
用法如下，下面的例子是拿到每个员工的提成总和:  

```scala
val df = Seq((Map("张三"->10000,"李四"->25000),Map("张三"->1000,"李四"->2500))).toDF("emp_sales_dept1","emp_sales_dept2")
df.withColumn("total_emp_sales",map_zip_with($"emp_sales_dept1",$"emp_sales_dept2",(k,v1,v2)=>(v1+v2))).show
```

## map_filter  
对map进行过滤，只保留满足特定条件的数据。会返回一个新的map。  
用法如下，下面这个例子找出提成大于1000的员工：  

```scala
val df = Seq(Map("张三"->100,"李四"->2500)) .toDF("emp_sales")
df.withColumn("filtered_sales",map_filter($"emp_sales",(k,v)=>(v\>1000))).show
```

## transform_values  
对map中的value用指定函数进行转换。  
用法如下，下面的例子给每位员工薪资+1000块:  

```scala
val df = Seq(Map("张三"->1000,"李四"->2500)).toDF("employee_salary")
df.withColumn("employee_salary",transform_values($"employee_salary",(name,salary)=>(salary+5100))).show
```
  
## transform_keys  
和transform_values类似，只是对key进行操作。

## xhash64
对给定列用64bit的xxhash算法计算hash值。

# 从Spark SQL迁移到Scala API的算子
有些Spark SQL中的函数，在Spark中也用的很频繁。所以Spark 3.0将这一些函数从Spark SQL迁移到了Scala API。以前只能通过Spark SQL或者Spark callUDF调用。但是现在可以很方便的通过Scala API进行调用啦。

## date_sub  
日期/时间戳/字符串中减去特定天数。如果是字符串，格式需要是yyyy-MM-dd or yyyy-MM-dd HH:mm:ss.SSSS。这个函数也很有用。 
用法如下，下面这个例子将日期减去一天: 

```scala
var df = Seq("2021-01-02 01:01:01","2021-01-03 01:01:01").toDF("time")
df.withColumn("subtracted_time", date_sub($"time",1)).show()
```

## date_add  
和**date_sub**相反，日期加上特定天数。

## months_add  
和**date_add**类似，但是是日期加上特定月数。

## zip_with  
合并两个数组。

用法如下:

```scala
val df = Seq((Seq(2,4,6),Seq(5,10,3))).toDF("array_1","array_2")
df.withColumn("merged_array",zip_with($"array_1",$"array_2",(x,y)=>(x+y))).show
```

## filter  
从数组中过滤出来满足条件的数据。  
## aggregate  
对数组中的数据进行合并。

# Spark SQL 新函数
## acosh
查找给定表达式的双曲余弦的倒数。  
## asinh
求给定表达式的双曲正弦的倒数。  
## atanh
求给定表达式的双曲正切的倒数。  
## bit_and/bit_or/bit_xor  
计算比特与，比特或，比特异或。  
## bit_count
返回特定字符串的bit长度  
bool_and and bool_or  
## count_if
返回特定列中满足特定函数的个数  
## date_part
从date/timestamp中提取特定部分，比如小时，分钟等。  
## div
做除法操作  
## every/some
every: 如果列中的每个数据满足条件，返回true。some: 如果列中的只要有一个满足条件，返回true。
## make_date/make_interval/make_timestamp
构造日期
## max_by/min_by
比较两列，返回最大/最小值。
## typeof
返回给定列的数据类型
## version
返回Spark版本
## justify_days/justify_hours/justify_interval
调整时间