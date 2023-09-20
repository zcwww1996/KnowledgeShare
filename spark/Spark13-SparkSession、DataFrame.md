[TOC]
# 1. 启动spark session
spark shell

```bash
spark-shell --queue ss_deploy

……
val parquetFile = spark.read.parquet("/user/ss_deploy/workspace/ss-ng/beijing/cell_model/dwell/0/2021/05/15/csv_1_0/")
```
其中`spark`为已存在的spark session

# 2. 创建DataFrame
## 2.1 第一种：通过Seq生成

**仅支持多列，单列情况可以利用Seq对偶元组生成dataframe后drop一列或者通过方法3**

```scala
val spark = SparkSession
  .builder()
  .appName(this.getClass.getSimpleName).master("local")
  .getOrCreate()

val seq=Seq(
  ("ming", 20, 15552211521L),
  ("hong", 19, 13287994007L),
  ("zhi", 21, 15552211523L)
)

// 方法1：
val df1 = spark.createDataFrame(seq).toDF("name", "age", "phone")

// 方法2：
val df2 = seq.toDF("name", "age", "phone")

// 方法3：支持单列
val rdd = spark.sparkContext.parallelize(seq)
val df = rdd.toDF("name", "age", "phone") column

df1.show()
```

## 2.2 第二种：读取Json文件生成

json文件内容

```json
{"name":"ming","age":20,"phone":15552211521}
{"name":"hong", "age":19,"phone":13287994007}
{"name":"zhi", "age":21,"phone":15552211523}
```

代码

```scala
    val dfJson = spark.read.format("json").load("/Users/shirukai/Desktop/HollySys/Repository/sparkLearn/data/student.json")
    dfJson.show()
```

## 2.3 第三种：读取csv文件生成

csv文件 students.csv

```scala
name,age,phone
ming,20,15552211521
hong,19,13287994007
zhi,21,15552211523
```

代码：

方法1：
```scala
val dfCsv = spark.read.format("csv").option("header", true).load("d:\\data\\students.csv")
    dfCsv.show()
```

方法2：
```Scala
  // inferSchema 自动推断属性列的数据类型
    val dataFrame = session.read.option("delimiter", ",").
    option("header", true).option("inferSchema","true").
    csv("d:\\data\\students.csv").toDF("name", "p_name", "age", "phone")
      
    //  打印数据格式
    dataFrame.printSchema()
```


方法3：

构建数据格式
```Scala
object Schema extends Serializable {

  final val SSNG_LOCATION_ZONE_SCHEMA = new StructType().
    add("geoHashn", StringType).
    add("zoneId", LongType).
    add("zoneName", StringType).
    add("cityCode", StringType).
    add("cityName", StringType).
    add("provinceCode", StringType).
    add("provinceName", StringType).
    add("POLYGON", StringType)
}
```

```Scala
 val locationZone: DataFrame = spark.read.option("delimiter", "|").schema(Schema.SSNG_LOCATION_ZONE_SCHEMA)
      .csv(globalGridPath)
```


> ps: 输出csv文件
>
> ```scala
> frame_str.write.option("delimiter", "|").option("nullValue", "?").csv(output_csv)
> ```

**==spark读写csv对空字符串的处理==**

**写csv:**<br>
1. spark写出csv，**null字段**默认写成`""`，如：`a,b,"",d`，需要生成``a,b,"?",d``形式，在write时添加 `option(“nullValue”,"?")`
2. spark写出到csv时,**空字符串**会写成 `""`,例如: `a,b,"",d`,如果想生成这样的形式: `a,b,d`,在write时添加 `option(“emptyValue”,"")`

读csv:<br>
1. spark读取csv时,对空字符串会翻译成null值,如果不想翻译成null,可以在fill()中替换成自己想要的字符,例如替换成空字符串:`spark.read.csv(“path”).na.fill("")`
2. 某一列替换空值，`val newDf = df.na.fill("e",Seq("blank"))`


**spark读取csv文件有许多参数可以设置**

参考：https://blog.csdn.net/whgyxy/article/details/88879145

例如inferSchema”表示是否开启类型自动推测“，“header”时候有表头，下面列举所有的可选参数及其解释

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| sep | `,` | 将单个字符设置为每个字段和值的分隔符 |
| encoding | `UTF-8` | 按给定的编码类型对CSV文件进行解码 |
| quote | `"` | 设置用于转义带引号的值的单个字符，其中分隔符可以是值的一部分。如果你想关闭引号，你需要设置一个空字符串（不是`null`值）。这与`com.databricks.spark.csv`不同 |
| escape | `\` | 在已引用的值中设置用于转义引号的单个字符 |
| comment |  | 设置用于跳过行的单个字符。从这个角色开始。默认情况下，它是禁用的 |
| header | `false` | 使用第一行作为列的名称 |
| inferSchema | `false` | 从数据中自动推断输入模式。它需要对数据进行一次额外的传递 |
| ignoreLeadingWhiteSpace | `false` | 定义是否应跳过正在读取的值中的前导空格 |
| ignoreTrailingWhiteSpace | `false` | 定义是否应跳过正在读取的值的尾随空格 |
| nullValue |  | 设置null值的字符串表示形式。从2.0.1开始，这适用于所有支持的类型，包括字符串类型 |
| nanValue | `NaN` | 设置非数字 “值的字符串表示形式 |
| positiveInf | `Inf` | 设置正无穷大值的字符串表示 |
| negativeInf | `-Inf` | 设置负无穷大值的字符串表示 |
| dateFormat | `yyyy-MM-dd` | 设置表示日期格式的字符串。自定义日期格式遵循 以下格式。这适用于日期类型 `java.text.SimpleDateFormat` |
| timestampFormat | `yyyy-MM-dd'T'HH:mm:ss.SSSZZ` | 设置指示时间戳格式的字符串。自定义日期格式遵循以下格式。这适用于时间戳类型 `java.text.SimpleDateFormat` |
| maxColumns | `20480` | 定义一个记录可以有多少列的硬限制 |
| maxCharsPerColumn | `-1` | 定义允许的最大字符数 |



## 2.4 第四种：通过Json格式的RDD生成（弃用）

```scala
    val sc = spark.sparkContext
    import spark.implicits._
    val jsonRDD = sc.makeRDD(Array(
      "{\"name\":\"ming\",\"age\":20,\"phone\":15552211521}",
      "{\"name\":\"hong\", \"age\":19,\"phone\":13287994007}",
      "{\"name\":\"zhi\", \"age\":21,\"phone\":15552211523}"
    ))

    val jsonRddDf = spark.read.json(jsonRDD)
    jsonRddDf.show()
```

## 2.5 第五种：通过Json格式的DataSet生成

```scala
val jsonDataSet = spark.createDataset(Array(
  "{\"name\":\"ming\",\"age\":20,\"phone\":15552211521}",
  "{\"name\":\"hong\", \"age\":19,\"phone\":13287994007}",
  "{\"name\":\"zhi\", \"age\":21,\"phone\":15552211523}"
))
val jsonDataSetDf = spark.read.json(jsonDataSet)

jsonDataSetDf.show()
```

## 2.6 第六种: 通过csv格式的DataSet生成

```scala
   val scvDataSet = spark.createDataset(Array(
      "ming,20,15552211521",
      "hong,19,13287994007",
      "zhi,21,15552211523"
    ))
    spark.read.csv(scvDataSet).toDF("name","age","phone").show()
```

## 2.7 第七种：动态创建schema

```scala
    val schema = StructType(List(
      StructField("name", StringType, true),
      StructField("age", IntegerType, true),
      StructField("phone", LongType, true)
    ))
    val dataList = new util.ArrayList[Row]()
    dataList.add(Row("ming",20,15552211521L))
    dataList.add(Row("hong",19,13287994007L))
    dataList.add(Row("zhi",21,15552211523L))
    spark.createDataFrame(dataList,schema).show()
```
// https://blog.csdn.net/qq_35022142/article/details/79800394

Spark SQL和DataFrames支持的数据格式如下：

- **数值类型**
  - ByteType: 代表1字节有符号整数. 数值范围： -128 到 127.
  - ShortType: 代表2字节有符号整数. 数值范围： -32768 到 32767.
  - IntegerType: 代表4字节有符号整数. 数值范围： -2147483648 t到 2147483647.
  - LongType: 代表8字节有符号整数. 数值范围： -9223372036854775808 到 9223372036854775807.
  - FloatType: 代表4字节单精度浮点数。
  - DoubleType: 代表8字节双精度浮点数。
  - DecimalType: 表示任意精度的有符号十进制数。内部使用java.math.BigDecimal.A实现。
  - BigDecimal：由一个任意精度的整数非标度值和一个32位的整数组成。
- String类型
  - StringType: 表示字符串值。
- Binary类型
  - BinaryType: 代表字节序列值。
- Boolean类型
  - BooleanType: 代表布尔值。
- Datetime类型
  - TimestampType: 代表包含的年、月、日、时、分和秒的时间值
  - DateType: 代表包含的年、月、日的日期值


- **复杂类型**
  - ArrayType(elementType, containsNull): 代表包含一系列类型为elementType的元素。如果在一个将ArrayType值的元素可以为空值，containsNull指示是否允许为空。
  - MapType(keyType, valueType, valueContainsNull): 代表一系列键值对的集合。key不允许为空，valueContainsNull指示value是否允许为空
  - StructType(fields): 代表带有一个StructFields（列）描述结构数据。
  - StructField(name, dataType, nullable): 表示StructType中的一个字段。name表示列名、dataType表示数据类型、nullable指示是否允许为空。

Spark SQL所有的数据类型在 org.apache.spark.sql.types 包内。不同语言访问或创建数据类型方法不一样：

Scala 代码中添加`import org.apache.spark.sql.types._`，再进行数据类型访问或创建操作。


Java 可以使用 org.apache.spark.sql.types.DataTypes 中的工厂方法

```scala
val result: RDD[(Long, List[String], mutable.LinkedHashMap[String, Int], List[List[Int]])]

import org.apache.spark.sql.types._
val structFields_2: StructType = StructType(Seq(StructField("userId", LongType), StructField("windowDateList", ArrayType(StringType)), StructField("sourceCellIdPlaceId", MapType(StringType,IntegerType)), StructField("timeMatrix", ArrayType(ArrayType(IntegerType)))))
val result_row: RDD[Row] = result.map(t => Row(t._1,t._2,t._3,t._4))

val frame = session.createDataFrame(result_row, structFields_2)
frame.show()
```

## 2.8 第八种：通过jdbc创建

```scala
    //第八种：读取数据库（mysql）
    val options = new util.HashMap[String,String]()
    options.put("url", "jdbc:mysql://localhost:3306/spark")
    options.put("driver","com.mysql.jdbc.Driver")
    options.put("user","root")
    options.put("password","hollysys")
    options.put("dbtable","user")

    spark.read.format("jdbc").options(options).load().show()
```

## 2.9 第九种：通过样例类创建
**生成样例类**

case class
```Scala
import scala.collection.mutable

case class TimeMatrixCase(
                       userId: Long,
                       windowDateList: List[String],
                       sourceCellIdPlaceId: mutable.LinkedHashMap[String, Int],
                       timeMatrix: List[List[Int]],
                       time_map: mutable.LinkedHashMap[String, Int]
                     )
```

生成DataFrame
```Scala
 val result: RDD[(Long, List[String], mutable.LinkedHashMap[String, Int], List[List[Int]], mutable.LinkedHashMap[String, Int], mutable.LinkedHashMap[String, List[List[(String, Int)]]], mutable.LinkedHashMap[String, List[List[((String, Int), List[String])]]])] 
// result 类型

    val TimeMatrixRDD: RDD[TimeMatrixCase] = result.map(t => TimeMatrixCase(t._1, t._2, t._3, t._4, t._5, t._6, t._7))
    val frame = session.createDataFrame(TimeMatrixRDD)
    frame.write.parquet("D:\\output\\dwell\\parquet\\2")
```


**读取样例类**

```Scala
//注意导入隐士转换
    import session.implicits._
    val unersList: RDD[TimeMatrixCase] = session.read.parquet("D:\\output\\dwell\\parquet\\2").as[TimeMatrixCase].rdd
    unersList.foreach(println)
```


# 3. 修改DataFrame

## 3.1 DataFrame更改列column的名称
参考：https://www.it1352.com/1933676.html

```Scala
val df = Seq((2L, "a", "foo", 3.0)).toDF
df.printSchema
// root
//  |-- _1: long (nullable = false)
//  |-- _2: string (nullable = true)
//  |-- _3: string (nullable = true)
//  |-- _4: double (nullable = false)
```

**方法一：重命名多列：`toDF`方法**

```Scala
val schemas= Seq("id", "x1", "x2", "x3")
val dfRenamed = df.toDF(schemas: _*)
 
dfRenamed.printSchema
// root
// |-- id: long (nullable = false)
// |-- x1: string (nullable = true)
// |-- x2: string (nullable = true)
// |-- x3: double (nullable = false)
```

**方法二：重命名单列：`withColumnRenamed`**

```Scala
df.withColumnRenamed("_1", "x1")
```


**方法三：重命名单个列：`select`与`alias`一起使用**

```Scala
df.select($"_1".alias("x1"))
```

**方法四：重命名多列：`select`与`alias`一起使用**

```Scala
val lookup = Map("_1" -> "foo", "_3" -> "bar")

df.select(df.columns.map(c => col(c).as(lookup.getOrElse(c, c))): _*)
```

## 3.2 DataFrame更改列column的类型


```Scala
val spark = SparkSession.builder().master("local").appName("DataFrame API").getOrCreate()

//    读取spark项目中example中带的几个示例数据，创建DataFrame
val df = spark.read.format("json").load("data/people.json")
df.show()
df.printSchema()

// 方法1：
val p = df.selectExpr("cast(age as string) age_toString","name")
p.printSchema()

// 方法2：
import spark.implicits._ //导入这个为了隐式转换，或RDD转DataFrame之用
import org.apache.spark.sql.types.DataTypes
df.withColumn("age", $"age".cast(DataTypes.IntegerType))    //DataTypes下有若干数据类型，记住类的位置
df.printSchema()

// 方法3：
df.select($"_c0".cast(LongType), $"_c1".cast(DoubleType), $"_c2".cast(DoubleType), $"_c3")
```

==**时间戳转换**==


```scala
dataFrame.withColumn("timeStart2", from_unixtime(col("timeStart") / 1000))
      .withColumn("timeStop2",
        date_format(from_unixtime(col("timeStop") / 1000), "yyyy-MM-dd HH:mm"))
      .show()
```

时间戳转字符串
```scala
+---+-------------+-------------+-------------------+----------------+
| id|    timeStart|     timeStop|         timeStart2|       timeStop2|
+---+-------------+-------------+-------------------+----------------+
|  0|1623543770074|1623543770074|2021-06-13 08:22:50|2021-06-13 08:22|
|  1|1623545038554|1623548118499|2021-06-13 08:43:58|2021-06-13 09:35|
|  2|1623571165020|1623571721129|2021-06-13 15:59:25|2021-06-13 16:08|

```

timeStart、timeStop为Long类型时间戳数据（例如：1625879922051），精确到毫秒,timeStop2按指定格式“yyyy-MM-dd HH:mm”进行了转换

字符串转时间戳
```
// 方法1：
df.withColumn("timeStart1", unix_timestamp(col("timeStart"), "yyyy-MM-dd HH:mm:ss") * 1000).show
// 方法2：
df.select($"userId".cast(LongType),unix_timestamp($"timeStart")*1000 as "timeStart1").show
+--------------------+-------------------+-------------+
|              userId|          timeStart|   timeStart1|
+--------------------+-------------------+-------------+
| 2954724055252731433|2023-01-01 16:21:03|1672561263000|
|-2094623826264211492|2023-01-01 10:25:54|1672539954000|
|-5013564583055136933|2023-01-01 19:51:34|1672573894000|
|-7676582467924041098|2023-01-01 16:24:27|1672561467000|
```



- `from_unixtime` 可以转换秒精度的时间，输入数据为可转换为 long 类型的数字（例如字符串或整数），输出数据为“yyyy-MM-dd HH:mm:ss”格式的字符串
- `unix_timestamp` 返回当前时间的unix时间戳（当前时间到1970-01-01 00:00:00），输入为“yyyy-MM-dd HH:mm:ss”格式的字符串或者timestamp类型，输出为long类型的秒数
- `date_format` 将日期/时间戳/字符串转换为字符串值，其格式由第二个参数给出的日期格式指定

## 3.3 DataFrame新增一列

参考：https://blog.csdn.net/li3xiao3jie2/article/details/81317249<br>
https://blog.csdn.net/Android_xue/article/details/90114567

- 方法一：利用createDataFrame方法，新增列的过程包含在构建rdd和schema中
- 方法二：利用withColumn方法，新增列的过程包含在udf函数中
- 方法三：利用SQL代码，新增列的过程直接写入SQL代码中
- 方法四：以上三种是增加一个有判断的列，如果**想要增加一列唯一序号，可以使用`monotonically_increasing_id`**

```scala
使用任意的值(可以是df中存在的列值，也可以是不存在的)增加一列
 
   val newDF = df.withColumn("last_update_time", lit(DateFormatUtils.format(new Date(), "yyyy-MM-dd HH:mm:ss")))
   val newDF = df.withColumn("t_start", col = concat(frame_result("dt"), lit(" "), frame_result("dh"), lit(":00:00")))
```

**对DataFrame增加一列索引列(自增id列)**

**方法1：利用RDD的 zipWithIndex算子**
```Scala
// 在原Schema信息的基础上添加一列 “id”信息
val schema: StructType = dataframe.schema.add(StructField("id", LongType))

// DataFrame转RDD 然后调用 zipWithIndex
val dfRDD: RDD[(Row, Long)] = dataframe.rdd.zipWithIndex()

val rowRDD: RDD[Row] = dfRDD.map(tp => Row.merge(tp._1, Row(tp._2)))

// 将添加了索引的RDD 转化为DataFrame
val df2 = spark.createDataFrame(rowRDD, schema)

df2.show()
```


**方法2：使用SparkSQL的function**
```Scala
import org.apache.spark.sql.functions._
val inputDF = inputDF.withColumn("id", monotonically_increasing_id)
inputDF.show
```


> ```Scala
> import org.apache.spark.sql.functions.typedLit
> df.withColumn("newcol", typedLit(("sample", 10, .044)))
> 
> +---+----+-----------------+
> | id|col1|           newcol|
> +---+----+-----------------+
> |  0|   a|[sample,10,0.044]|
> |  1|   b|[sample,10,0.044]|
> |  2|   c|[sample,10,0.044]|
> +---+----+-----------------+
> ```


# 4. 删除DataFrame

删除一列

```Scala
val data = df.drop("Customers");
```

删除列名带点的一列

```Scala
val new = df.drop(df.col("old.column"))
```


# 5. 查询DataFrame

## 5.1 DataFrame使用抽取一列或几列

**生成DataFrame**
```Scala
import spark.implicits._

var df = Seq(
  ("0", "ming", "tj","2019-09-06 17:15:15", "2002", "192.196", "win7", "bai"),
  ("1", "ming", "tj","2019-09-07 16:15:15", "4004", "192.194", "win7", "wang"),
  ("0", "ming", "tj","2019-09-08 05:15:15", "7007", "192.195", "ios", "wang"),
  ("0", "ming", "ln","2019-09-08 05:15:15", "7007", "192.195", "ios", "wang"),
  ("0", "li", "hlj","2019-09-06 17:15:15", "2002", "192.196", "win7", "bai"),
  ("1", "li", "hlj","2019-09-06 17:15:15", "2002", "192.196", "win7", "bai"),
  ("0", "li", "hlj","2019-09-07 16:15:15", "4004", "192.194", "win7", "wang"),
  ("0", "li", "ln","2019-09-08 05:15:15", "7007", "192.195", "ios", "wang"),
  ("1", "tian", "hlj","2019-09-08 13:15:15", "8008", "192.194", "win7", "zhu"),
  ("0", "tian", "hlj","2019-09-08 19:15:15", "9009", "192.196", "mac", "bai"),
  ("0", "xixi", "ln","2019-09-08 19:15:15", "9009", "192.196", "mac", "bai"),
  ("1", "xixi", "jl","2019-09-08 19:15:15", "9009", "192.196", "mac", "bai"),
  ("0", "haha", "hegang","2019-09-08 15:15:15", "10010", "192.192", "ios", "wei")
).toDF("label", "name", "live","START_TIME", "AMOUNT", "CLIENT_IP", "CLIENT_MAC", "PAYER_CODE")

+-----+----+------+-------------------+------+---------+----------+----------+
|label|name|  live|         START_TIME|AMOUNT|CLIENT_IP|CLIENT_MAC|PAYER_CODE|
+-----+----+------+-------------------+------+---------+----------+----------+
|    0|ming|    tj|2019-09-06 17:15:15|  2002|  192.196|      win7|       bai|
|    1|ming|    tj|2019-09-07 16:15:15|  4004|  192.194|      win7|      wang|
|    0|ming|    tj|2019-09-08 05:15:15|  7007|  192.195|       ios|      wang|
|    0|ming|    ln|2019-09-08 05:15:15|  7007|  192.195|       ios|      wang|
|    0|  li|   hlj|2019-09-06 17:15:15|  2002|  192.196|      win7|       bai|
|    1|  li|   hlj|2019-09-06 17:15:15|  2002|  192.196|      win7|       bai|
|    0|  li|   hlj|2019-09-07 16:15:15|  4004|  192.194|      win7|      wang|
|    0|  li|    ln|2019-09-08 05:15:15|  7007|  192.195|       ios|      wang|
|    1|tian|   hlj|2019-09-08 13:15:15|  8008|  192.194|      win7|       zhu|
|    0|tian|   hlj|2019-09-08 19:15:15|  9009|  192.196|       mac|       bai|
|    0|xixi|    ln|2019-09-08 19:15:15|  9009|  192.196|       mac|       bai|
|    1|xixi|    jl|2019-09-08 19:15:15|  9009|  192.196|       mac|       bai|
|    0|haha|hegang|2019-09-08 15:15:15| 10010|  192.192|       ios|       wei|
+-----+----+------+-------------------+------+---------+----------+----------+

```

### 5.1.1 选择一列

```scala
val da1 = data1.select("label")
da1.show()

结果：

da1: org.apache.spark.sql.DataFrame = [label: string]
+-----+
|label|
+-----+
|    0|
|    1|
|    0|
|    0|
|    0|
|    1|
|    0|
|    0|
|    1|
|    0|
|    0|
|    1|
|    0|
+-----+
```

### 5.1.2 选择多列

val sef = Seq("label", "AMOUNT")<br>
(也可以用Array,ArrayBuffer)

- 方法1:<br>
  `select(sef.head, sef.tail: _*)`
- 方法2:<br>
  `select(sef.map(data1.col()): _*)`

```scala
val sef = Seq("label", "AMOUNT")
val da3 = data1.select(sef.head, sef.tail: _*)
da3.show()

结果：
da3: org.apache.spark.sql.DataFrame = [label: string, AMOUNT: string]
+-----+------+
|label|AMOUNT|
+-----+------+
|    0|  2002|
|    1|  4004|
|    0|  7007|
|    0|  7007|
|    0|  2002|
|    1|  2002|
|    0|  4004|
|    0|  7007|
|    1|  8008|
|    0|  9009|
|    0|  9009|
|    1|  9009|
|    0| 10010|
+-----+------+

或者：
data1.select(sef.map(data1.col(_)): _*).show
```

> ==**补充1**==：<br>
> **select方法**
>
> ```Scala
> def select(cols : org.apache.spark.sql.Column*) : org.apache.spark.sql.DataFrame = { /* compiled code */ }
> ```
> 针对传入的是字符，必须是 (col=“字符”, cols=Seq(“字符列表”): _*)
>
> 参考：https://n3xtchen.github.io/n3xtchen/scala/2018/01/22/scala-spark-dataframe-dataframeselect-multiple-columns-given-a-sequence-of-column-names

> ==**补充2**==：<br>
> `_*`的作用：不定参数列表/变量声明中的模式
>
> **1) 变长参数**
>
> 定义一个变长参数的方法sum，然后计算1-5的和，可以写为
> ```scala
> scala> def sum(args: Int*) = {
>      | var result = 0
>      | for (arg <- args) result += arg
>      | result
>      | }
> sum: (args: Int*)Int
> 
> scala> val s = sum(1,2,3,4,5)
> s: Int = 15
> ```
>
> 但是如果使用这种方式就会报错
>
> ```scala
> scala> val s = sum(1 to 5)
> <console>:12: error: type mismatch;
>  found   : scala.collection.immutable.Range.Inclusive
>  required: Int
>        val s = sum(1 to 5)
>                      ^
> ```
>
> 这种情况必须在后面写上`: _*`将1 to 5转化为参数序列
>
> ```scala
> scala> val s = sum(1 to 5: _*)
> s: Int = 15
> ```
>
> **2) 变量声明中的模式**
>
> 例如，下面代码分别将arr中的第一个和第二个值赋给first和second
>
> ```scala
> scala> val arr = Array(1,2,3,4,5)
> arr: Array[Int] = Array(1, 2, 3, 4, 5)
> 
> scala> val Array(1, 2, _*) = arr
> 
> scala> val Array(first, second, _*) = arr
> first: Int = 1
> second: Int = 2
> ```

## 5.2 过滤的filter和where
Where部分可以用filter函数和where函数。这俩函数的用法是一样的，官网文档里说where是filter的别名。

参考：https://spark.apache.org/docs/latest/api/sql/#_11

```show
spark.createDataFrame(Seq(("aaa",1,2),("bbb",3,4),("ccc",3,5),("bbb",4, 6))).toDF("key1","key2","key3")

scala> df.show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
| bbb|   3|   4|
| ccc|   3|   5|
| bbb|   4|   6|
+----+----+----+
```

### 5.2.1 filter
从Spark官网的文档中看到，filter函数有下面几种形式：

```scala
def filter(func: (T) ⇒ Boolean): Dataset[T]
def filter(conditionExpr: String): Dataset[T]
def filter(condition: Column): Dataset[T]
```

以下几种写法都是可以的：

```scala
scala> df.filter($"key1">"aaa").show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| bbb|   3|   4|
| ccc|   3|   5|
| bbb|   4|   6|
+----+----+----+


scala> df.filter($"key1"==="aaa").show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
+----+----+----+

scala> df.filter("key1='aaa'").show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
+----+----+----+

scala> df.filter("key2=1").show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
+----+----+----+

scala> df.filter("key2<>1").show

scala> df.filter($"key2"===3).show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| bbb|   3|   4|
| ccc|   3|   5|
+----+----+----+

scala> df.filter($"key2"===$"key3"-1).show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
| bbb|   3|   4|
+----+----+----+
```

==**其中, `===`是在Column类中定义的函数，对应的不等于`是=!=`**。==<br>
==**`$"列名"`这个是语法糖，返回Column对象**==



**多条件过滤**：
注意导入隐式转换`import sparksql.implicits._`
```Scala
df.filter($"s"===1 || $"ss"=!=2)

df.where("id = 10 and age = 5").show
```

### 5.2.2 where
```scala
scala> df.where("key1 = 'bbb'").show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| bbb|   3|   4|
| bbb|   4|   6|
+----+----+----+


scala> df.where($"key2"=!= 3).show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
| bbb|   4|   6|
+----+----+----+


scala> df.where($"key3">col("key2")).show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| aaa|   1|   2|
| bbb|   3|   4|
| ccc|   3|   5|
| bbb|   4|   6|
+----+----+----+


scala> df.where($"key3">col("key2")+1).show
+----+----+----+
|key1|key2|key3|
+----+----+----+
| ccc|   3|   5|
| bbb|   4|   6|
+----+----+----+
```

动态传递过滤条件
```scala
// 方法1：
val cityCode = "V0370100"
df.where(s"cityCode = '$cityCode'").show

// 方法2：
import spark.implicits._
df.where($"cityCode" === cityCode).show
```


集合类常用过滤条件
```scala
df.where("coreUser10 is null" ).show
df.where($"coreUser10".isNull).show

df.where("zoneId like '3701%'").show

// count 是数组或者map集合
df.where("size(counts) >2").show(false)
// userId 是字符串
df.where("length(userId) >1").show

//去掉空字符串
df.where("sentence <> ''").show()

//去除null和NaN
df.na.drop().show()
```

### 5.2.3 isin

查询DataFrame某列在某些值里面的内容，等于SQL IN ,如 `where year in('2017','2018')`

isin 方法**只能传集合类型，不能直接传DataFame或Column**


```Scala
val data = Array(("001", "张三", 21, "2018"), ("002", "李四", 18, "2017"),
      ("003", "sam", 18, "2019"), ("004", "abby", 23, "20117")
    )

val df = spark.createDataFrame(data).toDF("id", "name", "age", "year")
df.show()

val yearArray = Array("2017", "2018")
df.where(col("year").isin(yearArray: _*)).show()


#### 结果 #####
+---+----+---+-----+
| id|name|age| year|
+---+----+---+-----+
|001|张三| 21| 2018|
|002|李四| 18| 2017|
|003| sam| 18| 2019|
|004|abby| 23|20117|
+---+----+---+-----+

+---+----+---+----+
| id|name|age|year|
+---+----+---+----+
|001|张三| 21|2018|
|002|李四| 18|2017|
+---+----+---+----+
```

### 5.2.4 array_contains
用于判定包含（array_contains）或不包含（!array_contains）关系

array_contains(数组，值)，返回布尔类型值

```Scala
val data = Array(("001", Array("Jim", "Tom", "Juily"), "2018"),
      ("002", Array("Jim", "Tom", "Lucy"), "2017"),
      ("003", Array("Sam", "Ivy", "Jim"), "2019"),
      ("004", Array("Abby", "Angela", "Anny"), "2017")
    )

val df = spark.createDataFrame(data).toDF("id", "names", "year")

df.filter(array_contains($"names", "Jim")).show

#### 结果 #####
+---+-----------------+----+
| id|            names|year|
+---+-----------------+----+
|001|[Jim, Tom, Juily]|2018|
|002| [Jim, Tom, Lucy]|2017|
|003|  [Sam, Ivy, Jim]|2019|
+---+-----------------+----+

df.filter(array_contains($"names", "Jim") and array_contains($"names", "Tom")).show

#### 结果 #####
+---+-----------------+----+
| id|            names|year|
+---+-----------------+----+
|001|[Jim, Tom, Juily]|2018|
|002| [Jim, Tom, Lucy]|2017|
+---+-----------------+----+
```

### 5.2.5 when otherwise

```scala
样例1：
spark.sql("select * from mydemo")

+---+-----+------+
|id |name |gender|
+---+-----+------+
|1  |Jack |M     |
|2  |Judy |F     |
|3  |Robot|N     |
+---+-----+------+

var ds = spark.table("mydemo")

ds.withColumn("flag",
      when($"gender" === "M", "男").
        when($"gender" === "F", "女").
        otherwise("未知"))

+---+-----+------+----+
| id| name|gender|flag|
+---+-----+------+----+
|  1| Jack|     M|  男|
|  2| Judy|     F|  女|
|  3|Robot|     N|未知|
+---+-----+------+----+

样例2：
var seq = Seq(
     | (1, "S"),
     | (2, "M"),
     | (3, "Unknown"))
val df= spark.createDataFrame(seq).toDF("id", "size")

df.withColumn("flag",when($"size".isin(Array[String]("S","M","L"):_*),"exist").otherwise($"size")).show

+---+-------+-------+
| id|   size|   flag|
+---+-------+-------+
|  1|      S|  exist|
|  2|      M|  exist|
|  3|Unknown|Unknown|
+---+-------+-------+
```


# 6. 解析DataFrame
## 6.1 解析dataframe各列
parquetFile是读取parquet文件生成的DataFrame，格式与样例数据如下：

样例
```scala
(0,0)
(1,-9221709592858595828)
(2,1621065364280)
(3,1621065364280)
(4,WrappedArray(011-63CE71D0A8C573A1D9D22C791AB24AC5-84EB13CFED01764D9C401219FAA56D53))
(5,WrappedArray(3409011))
(6,WrappedArray([3409011,1]))
(7,wx4f3gu)
(8,)
(9,)
(10,)
(11,3409011)
(12,011-63CE71D0A8C573A1D9D22C791AB24AC5-84EB13CFED01764D9C401219FAA56D53)
(13,0)
(14,1)
(15,0.0)
(16,0)
(17,)
(18,)
(19,Macro)
```

格式
```Scala
(0,id,IntegerType)
(1,userId,LongType)
(2,timeStart,LongType)
(3,timeStop,LongType)
(4,cells,ArrayType(StringType,true))
(5,universalCells,ArrayType(LongType,true))
(6,cellsFrequency,ArrayType(StructType(StructField(_1,LongType,true), StructField(_2,IntegerType,true)),true))
(7,geoHash7,StringType)
(8,boundId,StringType)
(9,userPoi,StringType)
(10,userPoiLocationId,StringType)
(11,mainCellid,LongType)
(12,mainSourceCellId,StringType)
(13,lastDwell,IntegerType)
(14,hiddenDwell,IntegerType)
(15,visitProb,FloatType)
(16,timeDur,IntegerType)
(17,visitMatrics,StringType)
(18,visitProbMatrics,StringType)
(19,mainCellCoverType,StringType)
```

```Scala
 // [0,-9218125298347495924,1621008083143,1621065203132,WrappedArray(011-D5DDF4101AFDB6575BF7FCC79937C606-84EB13CFED01764D9C401219FAA56D53),WrappedArray(2850811),WrappedArray([2850811,236]),wx4d59d,,,,2850811,011-D5DDF4101AFDB6575BF7FCC79937C606-84EB13CFED01764D9C401219FAA56D53,0,0,0.0,57119,,,Micro]

  val rows = parquetFile.rdd.map(r => {
      val t1 = r.getAs[Int](0)
      val t2 = r.getAs[Long](1) //.toString
      val fdf = FastDateFormat.getInstance("yyyy-MM-dd HH:mm:ss")
      val t3 = fdf.format(new Date(r.getAs[Long](2))).split(" ").toList
      val t4 = fdf.format(new Date(r.getAs[Long](3))).split(" ").toList
      val t5 =  r.getAs[Seq[String]](4).toList
      // val t5 = r.getAs[mutable.WrappedArray[String]](4).toList
      // val t5: List[String] = r(4).asInstanceOf[Seq[String]].toList
      val arrs = r.getAs[mutable.WrappedArray[Row]](6).toList
      val t6: List[(Long, Int)] = arrs.map(r => {
        (r.getAs[Long](0), r.getAs[Int](1))
      })

      (t1, t2, t3, t4, t5, t6)

    })
```

**备注：**<br>

**注意事项1：**

第5列数据（索引为4），格式为`ArrayType(StringType,true)`,转换为rdd的时候，不能直接用array或list接收，不然会产生`scala.collection.mutable.WrappedArray$ofRef cannot be cast to [Ljava.lang.Object;`或`scala.collection.mutable.WrappedArray$ofRef cannot be cast to scala.collection.immutable.List`错误

==**正确转换方式**==

```Scala
// 错误写法
val t5 =  r.getAs[List[String]](4)
val t5 =  r.getAs[Array[String]](4).toList


// 正确写法
// 方法1
val t5 =  r.getAs[Seq[String]](4).toList

// 方法2
val t5 = r.getAs[mutable.WrappedArray[String]](4).toList

// 方法3
val t5: List[String] = r(4).asInstanceOf[Seq[String]].toList
```

**注意事项1：**

df的schema如下：

```
root
 |-- cityId: long (nullable = true)
 |-- countryId: long (nullable = true)
 |-- outline: array (nullable = true)
 |    |-- element: struct (containsNull = true)
 |    |    |-- _1: double (nullable = true)
 |    |    |-- _2: double (nullable = true)
 |-- provinceId: long (nullable = true)
 |-- townId: long (nullable = true)

```
使用如下方法在提取outline 字段时，报错


```scala
val outline = df.select("outline").collect().map(row => row.getAs[Seq[(Double,Double)]]("outline"))
```

解决方法，我已知两种：
第一种，在生成数据时，定义case class对象，不存储tuple，就我个人而言，不适用，数据已经生成完毕，再次生成数据需要2天的时间
第二种方法：

```scala
val lines = json.filter(row => !row.isNullAt(2)).select("outline").rdd.map(r => {
      val row:Seq[(Double,Double)] = r.getAs[Seq[Row]](0).map(x =>{(x.getDouble(0),x.getDouble(1))})
      row
    }).collect()
```

至于报错的原因，google了一下，我觉得有一种说法可信：
在row 中的这些列取得时候，要根据类型取，简单的像String，Seq[Double] 这种类型就可以直接取出来，但是像 Seq[(Double,Double)] 这种类型直接取得花就会丢失schema信息，虽然值能取到，但是schema信息丢了，在dataFrame中操作的时候就会抛错


## 6.2 DataFrame上的Action操作

### 6.2.1 `show`：展示数据

以表格的形式在输出中展示`jdbcDF`中的数据，类似于`select * from spark_sql_test`的功能。

`show`方法有四种调用方式，分别为：<br>
**（1）`show`**，只显示前20条记录。

示例：
```scala
jdbcDF.show
```

结果：<br>
![](https://img-blog.csdn.net/20161012231509220)

**（2）`show(numRows: Int)`**，显示`numRows`条

示例：
```scala
jdbcDF.show(3)
```

结果：<br>
![](https://img-blog.csdn.net/20161012231534486)

**（3）`show(truncate: Boolean)`**，是否最多只显示20个字符，默认为`true`。

示例：
```scala
jdbcDF.show(true)
jdbcDF.show(false)
```

结果：<br>
![](https://img-blog.csdn.net/20161012231559073)

**（4）`show(numRows: Int, truncate: Boolean)`**，综合前面的显示记录条数，以及对过长字符串的显示格式。  
示例：
```scala
jdbcDF.show(3, false)
```

结果：<br>
![](https://img-blog.csdn.net/20161012231622120)

### 6.2.2 `collect`：获取所有数据到数组

不同于前面的`show`方法，这里的`collect`方法会将`jdbcDF`中的所有数据都获取到，并返回一个`Array`对象。

```scala
jdbcDF.collect()
```

结果如下，结果数组包含了`jdbcDF`的每一条记录，每一条记录由一个`GenericRowWithSchema`对象来表示，可以存储字段名及字段值。

![](https://img-blog.csdn.net/20161012231645167)

### 6.2.3 `collectAsList`：获取所有数据到List

功能和`collect`类似，只不过将返回结构变成了`List`对象，使用方法如下

```scala
jdbcDF.collectAsList()
```
> 相似的还有`jdbcDF.collectAsMap()`

结果如下:<br>
![](https://img-blog.csdn.net/20161012231714902)

### 6.2.4 `describe(cols: String*)`：获取指定字段的统计信息

这个方法可以动态的传入一个或多个`String`类型的字段名，结果仍然为`DataFrame`对象，用于统计数值类型字段的统计值，比如`count, mean, stddev, min, max`等。

使用方法如下，其中`c1`字段为字符类型，`c2`字段为整型，`c4`字段为浮点型

```scala
jdbcDF .describe("c1" , "c2", "c4" ).show()
```

结果如下:<br>
![](https://img-blog.csdn.net/20161012231742058)

### 6.2.5 `first, head, take, takeAsList`：获取若干行记录

这里列出的四个方法比较类似，其中  
　　（1）`first`获取第一行记录  
　　（2）`head`获取第一行记录，`head(n: Int)`获取前n行记录  
　　（3）`take(n: Int)`获取前n行数据  
　　（4）`takeAsList(n: Int)`获取前n行数据，并以`List`的形式展现

以`Row`或者`Array[Row]`的形式返回一行或多行数据。`first`和`head`功能相同。

`take`和`takeAsList`方法会将获得到的数据返回到Driver端，所以，使用这两个方法时需要注意数据量，以免Driver发生`OutOfMemoryError`

# 7. DataSet API
来源：https://blog.csdn.net/u013560925/article/details/80398081

参考：https://www.cnblogs.com/xp-thebest/archive/2021/01/19/14300479.html

参考：https://www.cnblogs.com/honey01/p/8065232.html

Spark版本：2.2

DataSet API网址：http://spark.apache.org/docs/2.2.1/api/java/org/apache/spark/sql/Dataset.html

## 7.1 groupBy()

1) **使用方法**

按照某几列元素进行分组

```scala
dataset.groupBy("columnName","columnName")
dataset.groupBy(dataset("columnName"))
```

2) **注意事项**

运算完成之后，返回的不是普通的DataSet数据类型，而是`org.apache.spark.sql.RelationalGroupedDataset`

`org.apache.spark.sql.RelationalGroupedDataset`可是用的方法有max、min、agg和sum等少量函数


```bash
max(colNames: String*)方法，获取分组中指定字段或者所有的数字类型字段的最大值，只能作用于数字型字段
min(colNames: String*)方法，获取分组中指定字段或者所有的数字类型字段的最小值，只能作用于数字型字段
mean(colNames: String*)方法，获取分组中指定字段或者所有的数字类型字段的平均值，只能作用于数字型字段
sum(colNames: String*)方法，获取分组中指定字段或者所有的数字类型字段的和值，只能作用于数字型字段
count()方法，获取分组中的元素个数
```


3) **使用举例**

```scala
logDS.groupBy("userID").max()

data.groupBy("name").avg("score").show
```

[![](https://img-blog.csdn.net/20180521215120379)](https://z3.ax1x.com/2021/06/28/RNHGOf.png)

## 7.2 agg()

1) **使用方法**

dataset 聚合函数api，有多种列名的传入方式，使用as，可是重新命名，所有计算的列sum、avg或者distinct都会成为新的列，默认列名sum(columnName)，常常跟在groupBy算子后使用

```scala
 dataset.agg(sum(dataset("columnsName")),sum(dataset("")).as("newName"))
 dataset.agg(sum("columnsName"),avg("columnsName").as("newName"))
 ```

2) **注意事项**

如果想要使用更多的内置函数，请引入：`import org.apache.spark.sql.functions._`

计算完会生成新的列，所以一般放了方便后期使用，会进行重命名

3) **使用举例**

**==经常和groupBy后使用，对同一group内的数据操作==**

```scala
logDs.groupBy("userID").agg(max("consumed"),max("logID"))
```

[![](https://img-blog.csdn.net/20180521214445587)](https://z3.ax1x.com/2021/06/28/RNH86P.png)


```Scala
data.groupBy("name").agg("score" -> "sum", "lesson" -> "count").show()

data.groupBy("name").agg(Map("score" -> "sum", "lesson" -> "count")).show()
```

[![image](https://img-blog.csdnimg.cn/20200505105816929.png)](https://pic.imgdb.cn/item/610a56525132923bf88c6578.png)

## 7.3 sortWithinPartition()

1) **使用方法**

在分区内部进行排序，执行完成后，分区内部有序

```scala
dataset.sortWithinPartitions("columnName1","columnName2")
```

2) **注意事项**

一般在调用改方法之前，会按照规则将数据重新分区，否则没有意义


```Scala
 val result = result_tmp.repartition(3, $"user_id", $"imei_tac").sortWithinPartitions("user_id")
 result.show()
```

调用之后可以接foreachPartition()遍历每一个分区

但是奇怪的是，我没有查到dataset相关分区的方法，只有rdd(key-value)才有分区的能力，所以，这个方法的使用意义和环境暂时还不清楚，使用的频率也较低

3) **使用举例**

```scala
personsDS.sortWithinPartitions("name").mapPartitions(it=>{
      it.toList.reverse.iterator
})
```

## 7.4 sort()

1) **使用方法**

按照某几列进行排序，**默认为升序**，加个`-`表示降序排序。sort和orderBy使用方法相同

```scala
dataset.sort(dataset("name"),dataset("age").desc)
```

```scala
dataset.sort(dataset("age").desc)
```

```scala
dataset.sort("columnName")
```

2) **注意事项**

传入字符串为列名的时候，无法指定降序排序

```
// 错误代码
result_tmp.groupBy("brand").count().sort("count".desc).show()
```

==**可以传入带列名的Column参数，指定降序排序**==

方法一：隐士转换+`$"columnName"`

```Scala
import spark.implicits._
result_tmp.groupBy("brand").count().sort($"count".desc).show()
```

方法二：col("columnName")

```Scala
result_tmp.groupBy("brand").count().sort(col("count").desc).show()
```

方法三：df("columnName")
```Scala
val result_count = result_tmp.groupBy("brand").count()
result_count.sort(result_count("count").desc).show()
```


3) **使用举例**

```scala
 personsDS.sort(personsDS("age").desc)
```

## 7.5 orderBy()

可以看到源代码中，orderBy调用可sort

```scala
def orderBy(sortCol: String, sortCols: String*): Dataset[T] = sort(sortCol, sortCols : _*)
```

例子：
```scala
jdbcDF.orderBy(- jdbcDF("c4")).show(false)
// 或者
jdbcDF.orderBy(jdbcDF("c4").desc).show(false)
// 或者
需要导入 import spark.implicits._
jdbcDF.orderBy($"c4".desc,$"c2").show(false)
```
通过列表排序
```scala
val strings = Array("userId", "geoHashn")
dataSet.orderBy(strings.head, strings.tail: _*).show()
```

## 7.6 hint


```scala
//spark版本 >= 2.2 广播提示
largeDF.join(smallDF.hint("broadcast"), Seq("foo"))
```


## 7.7 select

1) **使用说明**

选取某几列，但是在代码中就已经固定了选取的列的数量，如果想动态选取列，请使用spark sql

可以在select中直接对列进行修改数据

2) **使用举例**

```scala
dataset.select("columnName1","columnName2")
dataset.select(dataset("columnName"))
dataset.select(dataset("columnName")+1)
```

## 7.8 selectExpr

1) **使用说明**

同select，但是，可以填写表达式

2) **使用举例**

```scala
ds.selectExpr("colA", "colB as newName", "abs(colC)")
ds.select(expr("colA"), expr("colB as newName"), expr("abs(colC)"))

// 取出数组中的第一个元素
transitionSS.selectExpr("cells[0]").show(false)
```

sql中的四则运算
```Scala
// 求速度，除法
df.select(col("distance") / col("travelTime") as("speed")).show()

// 保留2位小数
df.selectExpr("round(distance / travelTime,2) as speed").show
df.selectExpr("cast(distance / travelTime as decimal(18,4)) as speed").show
df.select(bround(col("distance") / col("travelTime"),scale=4).alias("speed")).show

```

> decimal(18,4)中的“18”指的是整数部分加小数部分的总长度，也即插入的数字整数部分不能超过“18-4”即14位，否则不能成功插入，会报超出范围的错误。
>
> “4”表示小数部分的位数，如果插入的值未指定小数部分或者小数部分不足两位则会自动补到4位小数，若插入的值小数部分超过了4位则会发生截断，截取前4位小数。
## 7.9 rollup、cube

1) **使用说明**

rollup和cube的实现方式都是使用了RelationalGroupedDataset来实现，跟groupBy有点相似

cube: 选中的每个列的每种元素，进行组合

rollup:组合的种类较少，并不是全组合

使用场合比较少

2) **使用举例**

方便理解，附上一些解释文章：

https://www.cnblogs.com/wwxbi/p/6102646.html

https://blog.csdn.net/shenqingkeji/article/details/52860843

## 7.10 drop

1) **使用说明**

返回删除某一列的新数据集

```scala
dataset.drop("name","age")
dataset.drop(personsDS("name"))
```

2) **注意事项**

如果想删除多列，请使用string传递列名，column的方式只能删除一列

## 7.11 duplicates、distinct

1) **使用说明**

删除包含重复元素的行，重复元素可以是多个列的元素结合

```scala
 val cols = Array("name","userID")
dataset.dropDuplicates(cols)
dataset.dropDuplicates("name")
 ```

`def dropDuplicates(colNames: Seq[String])`
传入的参数是一个序列。你可以在序列中指定你要根据哪些列的重复元素对数据表进行去重，然后也是<font color="red">**返回每一行数据出现的第一条**</font>

当 `dropDuplicates` 中没有传入列名的时候, 其含义是根据所有列去重, `dropDuplicates()` 方法还有一个别名, 叫做 **`distinct`**

2) **使用举例**

```scala
val row1 = List("wq",22,15)
val row2 = List("wq",12,15)
val row3 = List("tom",12,15)
import spark.implicits._
val dataRDD =  spark.sparkContext.parallelize(List(row1,row2,row3))
case class User(name:String,age:Int,length:Int)
val df = dataRDD.map(x=>User(x(0).toString,x(1).toString.toInt,x(2).toString.toInt))
val ds = df.toDS()
val cols = Array("age","length")
ds.dropDuplicates(cols).show
println("-------")
ds.distinct().show()
```

[![](https://img-blog.csdn.net/20180524095356971)](https://z3.ax1x.com/2021/06/28/RNH3lt.png)

3) **==保留最后一条==**

| id | carNum | orgId | capTime  |
|----|--------|-------|----------|
| 1  | 粤A321 | 0002  | 10:20:10 |
| 2  | 云A321 | 0001  | 10:20:10 |
| 3  | 粤A321 | 0001  | 10:30:10 |
| 4  | 云A321 | 0002  | 10:30:10 |
| 5  | 粤A321 | 0003  | 11:40:10 |
| 6  | 京A321 | 0003  | 11:50:10 |

各字段含义分别为记录id，车牌号，抓拍卡口，抓拍时间。现在需要筛选出所有车辆最后出现的一条记录，得到每辆车最后经过的抓拍点信息

**思路**：

首先使用 `withColumn()` 添加 num 字段，num 字段是由 `row_number() + Window() + orderBy()` 实现的：开窗函数中进行去重，先对车牌carNum 进行分组，倒序排序，然后取窗口内排在第一位的则为最后的行车记录，使用 where 做过滤，最后drop掉不再使用的 num 字段；


```scala
import org.apache.spark.sql.functions._
import org.apache.spark.sql.expressions.Window
import spark.implicits._

val lastPassCar = carDF.withColumn("num",
  row_number().over(
    Window.partitionBy($"carNum")
            .orderBy($"capTime" desc)
  )
).where($"num" === 1).drop($"num")
lastPassCar.explain()
lastPassCar.show()
```

```scala
    +---+------+-----+--------+
| id|carNum|orgId| capTime|
+---+------+-----+--------+
|  5|粤A321|    3|11:40:10|
        |  6|京A321|    3|11:50:10|
        |  4|云A321|    2|10:30:10|
        +---+------+-----+--------+
```


## 7.12 describe

1) **使用说明**

对所有数据，按照列计算：最大值、最小值、均值、方差和总数

2) **使用举例**

还是使用上面的数据

```scala
ds.describe().show
```

[![](https://img-blog.csdn.net/20180524095643825)](https://z3.ax1x.com/2021/06/28/RNH4pR.png)

## 7.13 repartition、coalesce

1) **使用说明**：

重新分区，但是repartirion不能够使用自定义的partitioner，这里只是分区数目的变化

当分区数目由大变小的时候--->coalesce 避免shuffler

当分区数目由小变大的时候--->repartirion

- coalesce：减少分区, 此算子和 RDD 中的 coalesce 不同, Dataset 中的 coalesce 只能减少分区数, coalesce 会直接创建一个逻辑操作, 并且设置 Shuffle 为 false
- repartitions：有两个作用, 一个是重分区到特定的分区数, 另一个是按照某一列来分区, 类似于 SQL 中的 DISTRIBUTE BY

> **补充**:<br>
> **==DataFrame的repartition、partitionBy、coalesce区别==**
>
> - (1) `repartition(numPartitions, *cols)`<br> 重新分区(内存中，用于并行度)，用于分区数变多，设置变小一般也只是action算子时才开始shuffing;而且当参数numPartitions小于当前分区个数时会保持当前分区个数等于失效。
>
> ```Scala
> df.reparition(3, 'user_id, 'imei_tac)
> 或
> import spark.implicits._
> df.reparition(3, $"user_id", $"imei_tac")
> ```
>
> - (2) `partitionBy(*cols)`<br> 根据指定列进行分区（主要磁盘存储，影响生成磁盘文件数量，后续再从存储中载入影响并行度），相似的在一个区，并没有参数来指定多少个分区，而且仅用于PairRdd
>
> ```scala
> df.toDF("month","day","value").write.partitionBy("month","day")
> ```
>
> - (3) `coalesce(numPartitions)`<br> 联合分区，用于将分区变少。不能指定按某些列联合分区
>
> ```scala
> df.coalesce(1).rdd.getNumPartitions()
> ```


2) **使用举例**

coalesce适合用在filterBy之后，避免数据倾斜，下面给出相关的使用文章：

https://blog.csdn.net/u013514928/article/details/52789169

## 7.14 filter

1) **使用说明**

filter 参数传递有三种方式：spark方式，sql方式，判断方法

2) **使用举例**

```scala
dss.filter(dss("age")>15)
dss.filter("age > 15")
dss.filter(func (User)=> boolean) 因为我的数据每一行是一个User对象，所以方法输入是一个User对象，返回为boolean
```

filter还能查询空值，其实还是sql的方式比如：

```scala
dataset.filter("colName is not null").select("colName").limit(10).show
```


## 7.15 dtypes、columns

1) **使用说明**

dtypes 返回一个数组，每个列的列名和类型

columns 返回一个数组，每个列列名

## 7.16 checkpoint

1) **使用说明**

checkpoint，rdd或者dataset磁盘存储，用于比较关键的部分的数据，persist是内存缓存，而checkpoint是真正落盘的存储，不会丢失，当第二次使用时，在检查完cache之后，如果没有缓存数据，就是使用checkpoint数据，而避免重新计算

2) **使用举例**

[spark中checkpoint的使用](https://blog.csdn.net/qq_20641565/article/details/76223002)

## 7.17 Na

1) **使用说明**

检查ds中的残缺元素，返回值为DataFrameNaFunctions

DataFrameNaFunctions之后有三种方法比较常见：  
drop、fill、replace

2) **使用举例**

```scala
dataset.na.drop() 过滤包含空值的行,去除null和NaN
dataset.na.drop(Array("colName","colName"))  删除某些列是空值的行
        dataset.na.drop(10，Array("colName"))  删除某一列的值低于10的行
```

```scala
dataset.na.fill("value") 填充所有空值
        dataset.na.fill(value="fillValue",cols=Array("colName1","colName2") ) 填充某几列空值
```

```scala
dataset.na.fill(Map("colName1"->"value1","colName2"->"value2") ) 不同列空值不同处理
```

> ps:去掉空字符串
> ```scala
> // 去掉空字符串
> dataset.where("sentence <> ''").show()
> ```


## 7.18 join

1) **使用说明**

重点来了。在`SQL`语言中用得很多的就是`join`操作，DataFrame中同样也提供了`join`的功能。

接下来隆重介绍`join`方法。在DataFrame中提供了六个重载的`join`方法。

2) **使用举例**

我平时一般倾向于第2种join方式，也就是Seq

```scala
// Joining df1 and df2 using the column "user_id"
df1.join(df2, "user_id")

// Joining df1 and df2 using the columns "user_id" and "user_name"
df1.join(df2, Seq("user_id", "user_name"))

// The following two are equivalent:
df1.join(df2, $"df1Key" === $"df2Key")
df1.join(df2).where($"df1Key" === $"df2Key")

// Scala:
import org.apache.spark.sql.functions._
df1.join(df2, $"df1Key" === $"df2Key", "outer")
 ```


**（1）笛卡尔积**

```scala
joinDF1.join(joinDF2)
```

**（2）`using`一个字段形式**  
下面这种join类似于`a join b using column1`的形式，需要两个DataFrame中有相同的一个列名，

```scala
joinDF1.join(joinDF2, "id")
```

`joinDF1`和`joinDF2`根据字段`id`进行`join`操作，结果如下，`using`字段只显示一次。

[![](https://img-blog.csdn.net/20161012232652485)](https://www.z4a.net/images/2021/12/08/1a14cd2639ba77fdc.jpg)

**（3）、`using`多个字段形式**  
除了上面这种`using`一个字段的情况外，还可以`using`多个字段，如下

```scala
joinDF1.join(joinDF2, Seq("id", "name"))
```


**（4）、指定`join`类型**  
两个DataFrame的`join`操作有`inner, outer, left_outer, right_outer, leftsemi`类型。在上面的`using`多个字段的join情况下，可以写第三个`String`类型参数，指定`join`的类型，如下所示

```scala
joinDF1.join(joinDF2, Seq("id", "name"), "inner")
```

**（5）、使用`Column`类型来`join`**  
如果不用`using`模式，灵活指定`join`字段的话，可以使用如下形式

```scala
joinDF1.join(joinDF2 , joinDF1("id" ) === joinDF2( "t1_id"))

joinDF1.join(joinDF2 , joinDF1("id" ) === joinDF2( "t1_id") && joinDF1("userId") === joinDF2("userId"))

joinDF1.join(joinDF2 , joinDF1("id" ) === joinDF2( "t1_id") and joinDF1("userId") === joinDF2("userId"))
```

结果如下，  
[![](https://img-blog.csdn.net/20161012232748251)](https://www.z4a.net/images/2021/12/08/2047ae491e8f49f4b.jpg)

**（6）、在指定`join`字段同时指定`join`类型**  
如下所示

```scala
joinDF1.join(joinDF2 , joinDF1("id" ) === joinDF2( "t1_id"), "inner")
```


## 7.19 joinWith

1) **使用说明**

跟join 使用方式相似，但是joinWith之后的结果不会合并为一个，如下**使用举例**所示

2) **使用举例**

[![](https://img-blog.csdn.net/20180525105218748?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTM1NjA5MjU=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)](https://z3.ax1x.com/2021/06/28/RNHfh9.png)


## 7.20 apply、col

1) **使用说明**

跟select相同，输入列名，返回列

2) **使用举例**

```scala
public Column apply(String colName)
```

```scala
public Column col(String colName)
```



## 7.21 groupByKey

1) **使用说明**

从Dataset<Class> 转换成KeyValueGroupedDataset<K,T> 类型

不是按照key 分组，而是转换成key,value结构

2) **使用方法**

KeyValueGroupedDataset后面可以接agg等api

```scala
personsDS.groupByKey(per=>(per.name,per.age))
```

## 7.22 union

1) **使用说明**

union方法：对两个DataFrame进行组合

类似于SQL中的UNION ALL操作。

将两个dataset所有数据，将一个dataset的所有行接在另外一个后面

2) **使用举例**

```scala
personsDS.union(personsDS)
```

## 7.23 sample

1) **使用说明**

随机取样：


```Java
public Dataset<T> sample(boolean withReplacement,
        double fraction,
        long seed)
```


withReplacement是建立不同的采样器；fraction为采样比例；seed为随机生成器的种子

2) **使用举例**

```scala
dataset.sample(false,0.2,100);
```

## 7.24 randomSplit

1) **使用说明**

随机划分dataset


```Java
public Dataset<T>[] randomSplit(double[] weights,
        long seed)
```

2) **使用举例**

```scala
double [] weights =  {0.1,0.2,0.7};
//依据所提供的权重对该RDD进行随机划分
Dataset<Class> [] randomSplitDs = dataset.randomSplit(weights,100);
```

## 7.25 mapPartitions

1) **使用说明**

按照分区遍历每一个元素

2) **使用举例**

```scala
def doubleFunc(iter: Iterator[Int]) : Iterator[(Int,Int)] = {
  var res = List[(Int,Int)]()
  while (iter.hasNext)
  {
    val cur = iter.next;
    res .::= (cur,cur*2)
  }
  res.iterator
}
val result = a.mapPartitions(doubleFunc)
```

## 7.26 foreach

1) **使用说明**

对每一行的元素进行遍历

2) **使用举例**

```scala
dataset.foreach(p=>p.age)
```

## 7.27 foreachPartitoins

1) **使用说明**

按照分区遍历元素

当需要开关、访问数据库或者其他占用资源的操作的时候，可是在foreachPartitions里进行

2) **使用举例**

foreachPartitions和mapPartions的区别：https://blog.csdn.net/u010454030/article/details/78897150

foreachPartitions开关数据库举例：https://blog.csdn.net/lsshlsw/article/details/49789373

## 7.28 explode


```scala
df.select("userId", "poiId", "poiType", "geoHashCounter","coreUser", "coreUser10").where("userId= -9222835108189541158").show(false)

+--------------------+-----+-------+----------------------------+--------+----------+
|userId              |poiId|poiType|geoHashCounter              |coreUser|coreUser10|
+--------------------+-----+-------+----------------------------+--------+----------+
|-9222835108189541158|0    |0      |{wx4eh2n -> 1}              |true    |true      |
        |-9222835108189541158|1    |0      |{wx4sxw8 -> 1}              |true    |true      |
        |-9222835108189541158|2    |0      |{wx4uhpw -> 1}              |true    |true      |
        |-9222835108189541158|3    |0      |{wx4svfx -> 1, wx4svfq -> 3}|true    |true      |
```


```scala
import org.apache.spark.sql.functions.explode

df.select($"*", explode($"geoHashCounter").as(Seq("geohash","count"))).where("userId= -9223263032473946623").show()

+--------------------+-----+-------+------+------------+--------------------+--------+----------+------------+-------+-----+
|              userId|poiId|poiType|  zone|      weight|      geoHashCounter|coreUser|coreUser10|    Weight10|geohash|count|
+--------------------+-----+-------+------+------------+--------------------+--------+----------+------------+-------+-----+
|-9223263032473946623|    6|      0|110113|[1.76987324]|      [wx4um81 -> 1]|    true|      true|[1.79549226]|wx4um81|    1|
        |-9223263032473946623|    7|      0|110113|[1.76987324]|      [wx4uq33 -> 1]|    true|      true|[1.79549226]|wx4uq33|    1|
        |-9223263032473946623|    8|      0|110113|[1.76987324]|      [wx4umcr -> 1]|    true|      true|[1.79549226]|wx4umcr|    1|
        |-9223263032473946623|    9|      0|110113|[1.76987324]|[wx4umfs -> 1, wx...|    true|      true|[1.79549226]|wx4umfs|    1|
        |-9223263032473946623|    9|      0|110113|[1.76987324]|[wx4umfs -> 1, wx...|    true|      true|[1.79549226]|wx4umfd|    1|


df.select($"userId", $"poiId", $"poiType", $"geoHashCounter",$"coreUser", $"coreUser10", explode($"geoHashCounter")).where("userId= -9222835108189541158").show()

+--------------------+-----+-------+--------------------+--------+----------+-------+-----+
|              userId|poiId|poiType|      geoHashCounter|coreUser|coreUser10|    key|value|
+--------------------+-----+-------+--------------------+--------+----------+-------+-----+
|-9222835108189541158|    0|      0|      {wx4eh2n -> 1}|    true|      true|wx4eh2n|    1|
        |-9222835108189541158|    1|      0|      {wx4sxw8 -> 1}|    true|      true|wx4sxw8|    1|
        |-9222835108189541158|    2|      0|      {wx4uhpw -> 1}|    true|      true|wx4uhpw|    1|
        |-9222835108189541158|    3|      0|{wx4svfx -> 1, wx...|    true|      true|wx4svfx|    1|
        |-9222835108189541158|    3|      0|{wx4svfx -> 1, wx...|    true|      true|wx4svfq|    3|
```

## 7.29 map_keys

参考：https://www.it1352.com/1933860.html

**Spark >= 2.3**

```scala
import org.apache.spark.sql.functions.map_keys

df.select($"userId", $"poiId", $"poiType", $"geoHashn", $"geoHashCounter",$"coreUser", $"coreUser10", map_keys($"geoHashCounter")).where("userId= -9222835108189541158").show()

+--------------------+-----+-------+--------+--------------------+--------+----------+------------------------+
|              userId|poiId|poiType|geoHashn|      geoHashCounter|coreUser|coreUser10|map_keys(geoHashCounter)|
        +--------------------+-----+-------+--------+--------------------+--------+----------+------------------------+
|-9222835108189541158|    0|      0| wx4eh2n|      {wx4eh2n -> 1}|    true|      true|               [wx4eh2n]|
|-9222835108189541158|    1|      0| wx4sxw8|      {wx4sxw8 -> 1}|    true|      true|               [wx4sxw8]|
|-9222835108189541158|    2|      0| wx4uhpw|      {wx4uhpw -> 1}|    true|      true|               [wx4uhpw]|
|-9222835108189541158|    3|      0| wx4svfq|{wx4svfx -> 1, wx...|    true|      true|      [wx4svfx, wx4svfq]|
```

## 7.30 size

统计数组、集合长度

```scala
//counts是map集合
fk.where("size(counts) >2").show(false)

+---------------+-------+----------------------------------------------------------------------------------------------+
|universalCellId|numDays|counts                                                                                        |
+---------------+-------+----------------------------------------------------------------------------------------------+
|6203511_1      |1      |[6203511_1 -> 3, 4585211_1 -> 1, 6323911_1 -> 2, 7089911_1 -> 10]                             |
|2756811_1      |1      |[2885211_1 -> 1, 2756811_1 -> 138, 5549111_1 -> 1]                                            |
|5076411_1      |1      |[5076411_1 -> 8, 6920311_1 -> 1, 710611_1 -> 1]                                               |
|748911_1       |1      |[7246511_1 -> 3, 2129411_1 -> 2, 748911_1 -> 25]                                              |
|1080111_1      |1      |[1948911_1 -> 1, 1422711_0 -> 1, 1080111_1 -> 7]                                              |
```

## 7.31 lit、typeLit
**增加常量列**
- 使用lit函数来添加简单类型常量列<br>
  可以通过函数：`org.apache.spark.sql.functions.lit` 来添加简单类型(string,int,float,long等)的常量列
- 使用typedLit函数添加复合类型常量列<br>
  通过函数：`org.apache.spark.sql.functions.typedLit`，可以添加List，Seq和Map类型的常量列

```scala
scala> val df1 = sc.parallelize(Seq("Hello", "world")).toDF()

// lit
df1.withColumn("data_map", lit("teststring")).show(false)
+-----+----------+
|value|data_map  |
+-----+----------+
|Hello|teststring|
|world|teststring|
+-----+----------+

//  typedlit内置函数在spark2.2.0版本开始出现
df1.withColumn("some_array", typedLit(Seq(7, 8, 9))).show()
+-----+----------+
|value|some_array|
+-----+----------+
|Hello| [7, 8, 9]|
        |world| [7, 8, 9]|
        +-----+----------+

df1.withColumn("some_struct", typedLit(("teststring", 1, 0.3))).show(false)
+-----+--------------------+
|value|some_struct         |
+-----+--------------------+
|Hello|[teststring, 1, 0.3]|
        |world|[teststring, 1, 0.3]|
        +-----+--------------------+

df1.withColumn("data_map", typedLit(Map("k1" -> 1, "k2" -> 2))).show(false)
+-----+------------------+
|value|data_map          |
+-----+------------------+
|Hello|[k1 -> 1, k2 -> 2]|
        |world|[k1 -> 1, k2 -> 2]|
        +-----+------------------+

```

> **遇到的问题**<br>
> `Caused by: java.lang.ClassCastException: scala.collection.immutable.Map$Map3 cannot be cast to scala.collection.mutable.HashMap`
>
> 问题分析：
>
> 无论外部使用`mutable.HashMap`类型还是`immutable.HashMap`类型，传入列中都会统一变为`immutable.Map`类型，故在自定义函数接收时参数也一定要使用`immutable.Map`类型。


**udf函数传入额外参数**

1. 定义带额外参数的udf函数

```scala
 val extract = udf{(params:String, field_name:String)=>
  val obj = JSON.parseObject(params)
  obj.getString(field_name)
}
```
该函数中，params参数是常规的 spark dataframe 中的列，而 field_name 参数是需要额外向函数传入的非列参数，我们需要借助它完成我们的函数逻辑。
2. 使用带额外参数的 udf函数

```scala
    es_data
        .withColumn("dms1", extract(col("params"),lit("dms1")))
        .withColumn("dms2", extract(col("params"),lit("dms2")))
```
在这段代码中，params字段列是一个json字符串

样例值

`{"dms1":"v1","dms2":"v2"}`

我们实现了从params列中解析我们需要的dms1值和dms2值,并形成我们的dms1,dms2新列。我们知道在自定义udf函数时，每个参数一般都是dataframe中真实存在的列

## 7.32集合类型的操作(交、并、差)

集合类型的操作主要包含：**except、intersect、union、limit**

**（1）except**

方法描述：`except` 和 `SQL` 语句中的 `except` 一个意思, 是求得 `ds1` 中存在， `ds2` 中不存在的数据, 其实就是**差集**

```scala
@Test
def collection(): Unit = {
  val ds1 = spark.range(10)
  val ds2 = spark.range(5,15)

  ds1.except(ds2).show()
}
```

**（2）intersect**

方法描述：求得两个集合的交集

```scala
@Test
def collection(): Unit = {
  val ds1 = spark.range(10)
  val ds2 = spark.range(5,15)


  ds1.intersect(ds2).show()
}
```

**（3）union**

方法描述：求得两个集合的并集

```scala
@Test
def collection(): Unit = {
  val ds1 = spark.range(10)
  val ds2 = spark.range(5,15)

  ds1.union(ds2).show()
}
```

**（4）limit**

方法描述：限制结果集数量

```scala
@Test
def collection(): Unit = {
  val ds1 = spark.range(10)
  val ds2 = spark.range(5,15)

  ds1.limit(5).show()
}
```

## 7.33 字符串函数

链接：https://blog.csdn.net/weixin_42223090/article/details/102843007


**1.concat对于字符串进行拼接**  
concat(str1, str2, …, strN) - Returns the concatenation of str1, str2, …, strN.

Examples: `SELECT concat('Spark', 'SQL'); --- SparkSQL`

**2.`concat_ws`在拼接的字符串中间添加某种格式**  
`concat_ws(sep, [str | array(str)]+)` - Returns the concatenation of the strings separated by sep.

Examples:`SELECT concat_ws(' ', 'Spark', 'SQL'); --- Spark SQL`

**3.decode转码**  
decode(bin, charset) - Decodes the first argument using the second argument character set.

Examples: `SELECT decode(encode('abc', 'utf-8'), 'utf-8'); --- abc`

**4.encode设置编码格式**  
encode(str, charset) - Encodes the first argument using the second argument character set.

Examples: `SELECT encode('abc', 'utf-8'); --- abc`

**5.`format_string`/printf 格式化字符串**  
`format_string(strfmt, obj, …)` - Returns a formatted string from printf-style format strings.

Examples: `SELECT format_string("Hello World %d %s", 100, "days"); --- Hello World 100 days`

**6.initcap将每个单词的首字母变为大写，其他字母小写; lower全部转为小写，upper大写**  
initcap(str) - Returns str with the first letter of each word in uppercase. All other letters are in lowercase. Words are delimited by white space.

Examples: `SELECT initcap('sPark sql'); --- Spark Sql`

**7.length返回字符串的长度**  
Examples: `SELECT length('Spark SQL '); --- 10`

**8.levenshtein编辑距离（将一个字符串变为另一个字符串的距离）**  
levenshtein(str1, str2) - Returns the Levenshtein distance between the two given strings.

Examples:`SELECT levenshtein('kitten', 'sitting');　--- 3`

**9.lpad返回固定长度的字符串，如果长度不够，用某种字符补全，rpad右补全**  
lpad(str, len, pad) - Returns str, left-padded with pad to a length of len. If str is longer than len, the return value is shortened to len characters.

Examples: `SELECT lpad('hi', 5, '??');　--- ???hi`

**10.ltrim去除空格或去除开头的某些字符,rtrim右去除，trim两边同时去除**  
ltrim(str) - Removes the leading space characters from str.

ltrim(trimStr, str) - Removes the leading string contains the characters from the trim string

Examples:

```sql
SELECT ltrim('    SparkSQL   '); --- SparkSQL
SELECT ltrim('Sp', 'SSparkSQLS'); --- arkSQLS
```

**11.`regexp_extract` 正则提取某些字符串，`regexp_replace`正则替换**

Examples:

```sql
SELECT regexp_extract('100-200', '(\d+)-(\d+)', 1); --- 100
SELECT regexp_replace('100-200', '(\d+)', 'num'); ---num-num

```

**12.repeat复制给的字符串n次**

Examples:  `SELECT repeat('123', 2); ---　123123`

**13.instr返回截取字符串的位置/locate**  
instr(str, substr) - Returns the (1-based) index of the first occurrence of substr in str.

Examples: `SELECT instr('SparkSQL', 'SQL'); ---　6`

Examples: `SELECT locate('bar', 'foobarbar'); --- 4`

> **locate() 详解**<br>
> 语法 一：  
> LOCATE(substr,str)  
> 返回字符串substr中第一次出现子字符串的位置 str。
>
> 语法二：  
> LOCATE(substr,str,pos)  
> 返回字符串substr中第一个出现子 字符串的 str位置，从位置开始 pos。0 如果substr不在，则 返回str。返回 NULL如果substr 或者str是NULL。

```sql
SELECT LOCATE('bar', 'foobarbar');
#  -> 4

SELECT LOCATE('xbar', 'foobar');
# -> 0

SELECT LOCATE('bar', 'foobarbar', 5);
# -> 7
```

**14.space 在字符串前面加n个空格**  
space(n) - Returns a string consisting of n spaces.

Examples: `SELECT concat(space(2), '1'); ---　1`

**15.split以某些字符拆分字符串**  
split(str, regex) - Splits str around occurrences that match regex.

Examples: `SELECT split('oneAtwoBthreeC', '[ABC]'); ---　["one","two","three",""]`

**16.substr截取字符串，`substring_index`**  
Examples:

```sql
SELECT substr('Spark SQL', 5);--- k SQL
SELECT substr('Spark SQL', -3);--- SQL
SELECT substr('Spark SQL', 5, 1); --- k
SELECT substring_index('www.apache.org', '.', 2); --- www.apache

```

**17.translate 替换某些字符串为**  
Examples: `SELECT translate('AaBbCc', 'abc', '123'); --- A1B2C3`

**18.`get_json_object`**  
`get_json_object(json_txt, path)` - Extracts a json object from path.

Examples: `SELECT get_json_object(’{“a”:“b”}’, ‘$.a’); --- b`

**19.unhex**  
unhex(expr) - Converts hexadecimal expr to binary.

Examples: `SELECT decode(unhex(‘537061726B2053514C’), ‘UTF-8’); --- Spark SQL`

**20.`to_json`**  
`to_json(expr[, options])` - Returns a json string with a given struct value

Examples:

```SQL

SELECT to_json(named_struct('a', 1, 'b', 2));#　　 {"a":1,"b":2}

SELECT to_json(named_struct('time', to_timestamp('2015-08-26', 'yyyy-MM-dd')), map('timestampFormat', 'dd/MM/yyyy'));#　　 {"time":"26/08/2015"}

SELECT to_json(array(named_struct('a', 1, 'b', 2));#　　 [{"a":1,"b":2}]

SELECT to_json(map('a', named_struct('b', 1)));#　　{"a":{"b":1}}

SELECT to_json(map(named_struct('a', 1),named_struct('b', 2)));#　　 {"[1]":{"b":2}}

SELECT to_json(map('a', 1));#　　{"a":1}

SELECT to_json(array((map('a', 1))));#　　[{"a":1}]
Since: 2.2.0

```

**21.split**

字符串切割
```scala
df.withColumn("key_arr", split($"key_strs", "[|]")).withColumn("geoHash7", explode($"key_arr"))
        .select("userId", "poiId", "poiType", "key_strs","geoHash7").show

结果：
+--------------------+-----+-------+-------------------------------+--------+
|userId              |poiId|poiType|key_strs                       |geoHash7|
+--------------------+-----+-------+-------------------------------+--------+
|-1955534353007518298|0    |0      |wwe09xd|wwe09xg|wwe09xe|wwe09xu|wwe09xd |
        |-1955534353007518298|0    |0      |wwe09xd|wwe09xg|wwe09xe|wwe09xu|wwe09xg |
        |-1955534353007518298|0    |0      |wwe09xd|wwe09xg|wwe09xe|wwe09xu|wwe09xe |
        |-1955534353007518298|0    |0      |wwe09xd|wwe09xg|wwe09xe|wwe09xu|wwe09xu |
        |-1955534353007518298|2    |0      |wwe09x5|wwe09wf|wwe09x7        |wwe09x5 |
        |-1955534353007518298|2    |0      |wwe09x5|wwe09wf|wwe09x7        |wwe09wf |
        |-1955534353007518298|2    |0      |wwe09x5|wwe09wf|wwe09x7        |wwe09x7 |
        |-1955534353007518298|3    |0      |wwe09wd                        |wwe09wd |
        |-1955534353007518298|4    |0      |wwe09tc                        |wwe09tc |
        +--------------------+-----+-------+-------------------------------+--------+
```


# 8. Spark DataFrame 窗口函数

上面介绍了spark dataframe常用的一些操作。除此之外，spark还有一类操作比较特别的操作，窗口函数。窗口函数常多用于sql，spark sql也集成了，同样，spark dataframe也有这种函数，spark sql的窗口函数与spark dataframe的写法不太一样。

## 8.1 窗口函数写法

**spark sql 写法**

```sql
select pcode,event_date,sum(duration) over (partition by pcode order by event_date asc) as sum_duration from userlogs_date
```

**spark dataframe写法**

```scala
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions._

val first_2_now_window = Window.partitionBy("pcode").orderBy("event_date")
df_userlogs_date.select(
  $"pcode",
  $"event_date",
  sum($"duration").over(first_2_now_window).as("sum_duration")
).show

```

## 8.2 常见窗口函数
窗口函数形式为 over(partition by A order by B)，意为对A分组，对B排序，然后进行某项计算，比如求count，max等。

```sql
count(...) over(partition by ... order by ...)  --求分组后的总数。
sum(...) over(partition by ... order by ...)  --求分组后的和。
max(...) over(partition by ... order by ...)  --求分组后的最大值。
min(...) over(partition by ... order by ...)  --求分组后的最小值。
avg(...) over(partition by ... order by ...)  --求分组后的平均值。
row_number() over(partition by ... order by ...)  --排序，不会有重复的排序数值。对于相等的两个数字，排序序号(rank)不一致。
rank() over(partition by ... order by ...)  --rank值可能是不连续的，可有并列值。对于相等的两个数字，排序序号一致。
dense_rank() over(partition by ... order by ...)  --rank值是连续的，可有并列值。对于相等的两个数字，排序序号一致。
first_value(...) over(partition by ... order by ...)  --求分组内的第一个值。
last_value(...) over(partition by ... order by ...)  --求分组内的最后一个值。
lag() over(partition by ... order by ...)  --取出前n行数据，把数据从上向下推，上端出现空格。　　
lead() over(partition by ... order by ...)  --取出后n行数据，把数据从下向上推，下端出现空格。
ratio_to_report() over(partition by ... order by ...)  --Ratio_to_report() 括号中就是分子，over() 括号中就是分母。
percent_rank() over(partition by ... order by ...)  -- 计算当前行所在前百分位
```

DSL风格
```scala
import org.apache.spark.sql.expressions.Window

functions.max("...").over(Window.partitionBy("...")))
```

```scala
// lag 和lead 有三个参数，第一个参数是列名，第二个参数是偏移的offset，第三个参数是 超出记录窗口时的默认值
// 取上一条数据，不存在取默认值 9999999999999
df.withColumn("prevTimeStop", lag($"timeStop", 1, 9999999999999L).over(Window.partitionBy($"userId").orderBy($"id")))
```


## 8.3 使用样例
**窗口函数可以实现如下逻辑：  
1.求取聚合后个体占组的百分比  
2.求解历史数据累加**

### 8.3.1 求取聚合后个体占组的百分比

```scala
val data = spark.read.json(spark.createDataset(
  Seq(
    """{"name":"A","lesson":"Math","score":100}""",
    """{"name":"B","lesson":"Math","score":100}""",
    """{"name":"C","lesson":"Math","score":99}""",
    """{"name":"D","lesson":"Math","score":98}""",
    """{"name":"A","lesson":"English","score":100}""",
    """{"name":"B","lesson":"English","score":99}""",
    """{"name":"C","lesson":"English","score":99}""",
    """{"name":"D","lesson":"English","score":98}"""
  )))

data.show

spark.sql(
  s"""
     |select name, lesson, score, (score/sum(score) over()) as y1, (score/sum(score) over(partition by name)) as y2
     |from score
     |""".stripMargin).show

```

[![](https://img-blog.csdnimg.cn/20200505140551225.png)](https://z3.ax1x.com/2021/08/10/fYylmn.png)

### 8.3.2 求解历史数据累加

比如，有个需求，求取从2018年到2020年各年累加的物品总数。

```scala
val data1 = spark.read.json(spark.createDataset(
  Seq(
    """{"date":"2020-01-01","build":1}""",
    """{"date":"2020-01-01","build":1}""",
    """{"date":"2020-04-01","build":1}""",
    """{"date":"2020-04-01","build":1}""",
    """{"date":"2020-05-01","build":1}""",
    """{"date":"2020-09-01","build":1}""",
    """{"date":"2019-01-01","build":1}""",
    """{"date":"2019-01-01","build":1}""",
    """{"date":"2018-01-01","build":1}"""
  )))
data1.createOrReplaceTempView("data1")


spark.sql(
  s"""
     |select c.dd,sum(c.sum_build) over(partition by 1 order by dd asc) from
     |(select  substring(date,0,4) as dd, sum(build) as sum_build  from data1 group by dd) c
     |""".stripMargin).show

spark.sql(
  s"""
     |select c.dd,sum(c.sum_build) over (partition by 1) from
     |(select  substring(date,0,4) as dd, sum(build) as sum_build  from data1 group by dd) c
     |""".stripMargin).show

```

[![](https://img-blog.csdnimg.cn/20200505141231141.png)](https://z3.ax1x.com/2021/08/10/fYsoz4.png)
