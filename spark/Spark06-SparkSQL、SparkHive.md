[TOC]

# 1. SparkSql

Spark SQL处理结构化的数据的（类似传统的数据库中的二维表，有行有列有表头（schema））

Spark SQL中有一个基本的的抽象，即DataFrame（1.3）、Dataset（1.6），跟RDD比，DataFrame有更多的描述信息，有了这些额外的描述信息，Spark SQL执行时效率会提升

**DataFrame的额外的描述信息是什么？**

相比RDD就是多了schema（描述了表有哪些列，每一列叫什么名字，是什么类型，能不能为空）

**DataFrame = RDD + schema**

Spark在1.3提出了DataFrame，Spark1.6推出了Datasets，spark2.x将DataFrame和Dataset的API进行了统一，在Spark2.x中，
DataFrame = Dataset[Row]（Row有哪些列，什么类型）有了这些额外的描述信息，执行就会被优化

## 1.1 Spark SQL特点
Spark SQL是Spark的五大核心模块之一，用于在Spark平台之上处理结构化数据，利用Spark SQL可以构建大数据平台上的数据仓库，它具有如下特点：
1. 能够无缝地将SQL语句集成到Spark应用程序当中
2. 统一的数据访问方式 统一的数据访问方式
3. 兼容Hive 兼容Hive
4. 可采用JDBC or ODBC连接 可采用JDBC or ODBC连接

## <font color="red">1.2 Spark SQL和Hive On Spark（HiveSQL）的区别</font>

1. spark是标准的sql，默认情况下，spark使用的是标准的sql
2. hiveSql有很多特殊的语法，如果spark想使用hivesql，就必须开启对hiveSQL的支持
SparkSQL（SQL、DataFrame、Dataset、Hive on Spark）

---
## 1.3 SparkSQL API
SparkSQL的API有新旧两种形式：
    Spark1.x SQLContext
    Spark2.x SparkSession

从API的使用方式也有两种：SQL、DSL



### 1.3.1 Spark2.0 SparkSession
```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.sql.types._
import org.apache.spark.{SparkConf, SparkContext}
import org.apache.spark.sql.{DataFrame, Row, SQLContext, SparkSession}

object SQLDemo2x1 {

  def main(args: Array[String]): Unit = {

    //在spark2.x SQL的编程API的入口是SparkSession

    val session = SparkSession.builder()
      .appName("SQLDemo2x1")
      .master("local[*]")
      .getOrCreate()

    // RDD + schema = DataFrame
    //将RDD添加额外的信息，编程DF
    val lines:RDD[String] = session.sparkContext.parallelize(List("1,laoyang,99,29", "2,laozhao,9999,18", "3,laoduan,99,34"))

    //整理数据，将RDD管理schema信息
    val rowRDD: RDD[Row] = lines.map(line => {
      val fields = line.split(",")
      val id = fields(0).toLong
      val username = fields(1)
      val faceV = fields(2).toDouble
      val age = fields(3).toInt
      Row(id, username, faceV, age)
    })

    //有RDD了，但是没有schema信息
    val schema = StructType(
      List(
        StructField("id", LongType),
        StructField("name", StringType),
        StructField("fv", DoubleType),
        StructField("age", IntegerType)
      )
    )

    val df: DataFrame = session.createDataFrame(rowRDD, schema)

    //对df进行处理,注册表（视图）
    df.createTempView("v_boy")

    val result:DataFrame = session.sql("SELECT id,name, fv, age FROM v_boy ORDER BY fv DESC, age ASC")

    //执行Action

    result.foreachPartition(it => {
      //获取数据库连接
      it.foreach(row => {
        val id = row.getLong(0)
        val name = row.getString(1)
        //保存到数据库
      })
      //关闭连接
    })
  }
}
```
### 1.3.2 WordCountBySql
```scala
import org.apache.spark.sql.{DataFrame, Dataset, SparkSession}


object WordCountBySql {
  def main(args: Array[String]): Unit = {
    val session = SparkSession.builder().appName("WordCount").master("local[*]").getOrCreate()
    val line: Dataset[String] = session.read.textFile("d:/Test/word.txt")
    //切分压平
    //Dataset调用RDD上的方法，必须导入隐式转换
    import session.implicits._
    val words: Dataset[String] = line.flatMap(_.split(" "))
    //注册成表

    words.createTempView("t_word")
    //sql查询
    val result: DataFrame = session.sql("select value word,count(*) counts from t_word group by word order by counts desc")
    result.show(100)
    session.stop()
  }
}
```
### 1.3.3 WordCountByDSL
```scala
import org.apache.spark.sql.{Dataset, Row, SparkSession}

object WordCountByDSL {
  def main(args: Array[String]): Unit = {
    val session = SparkSession.builder().appName("WordCount").master("local[*]").getOrCreate()
    val line: Dataset[String] = session.read.textFile("d:/Test/word.txt")
    //切分压平
    //Dataset调用RDD上的方法，必须导入隐式转换
    import session.implicits._
    val words: Dataset[String] = line.flatMap(_.split(" "))
    //分组计算
    val result: Dataset[Row] = words.groupBy($"value" as "word").count().withColumnRenamed("count", "counts").orderBy($"counts" desc)
    result.show(50)
    session.close()
  }
}
```
## 1.4 DataSet、DataFrame区别

1. 关于类型方面：
    DataSet是带有类型的（typed），例：```DataSet<Persono>```。取得每条数据某个值时，使用类似person.getName()这样的API，可以保证类型安全。
    而DataFrame是无类型的，是以列名来作处理的，所以它的定义为```DataSet<Row>```。取得每条数据某个值时，可能要使用row.getString(0)或col("department")这样的方式来取得，无法知道某个值的具体的数据类型。
    ```DataFrame <==> DataSet[Row]```
2. 关于schema：
DataFrame带有schema，而DataSet没有schema。schema定义了每行数据的“数据结构”，就像关系型数据库中的“列”，schema指定了某个DataFrame有多少列。

## 1.5 自定义函数

介绍如何在Spark Sql和DataFrame中使用UDF，如何利用UDF给一个表或者一个DataFrame根据需求添加几列，并给出了旧版（Spark1.x）和新版（Spark2.x）完整的代码示例。

*   关于UDF：UDF：User Defined Function，用户自定义函数。

### 1.5.1 创建测试用DataFrame

下面以Spark2.x为例给出代码，关于Spark1.x创建DataFrame可在最后的完整代码里查看。

```scala

val userData = Array(("Leo", 16), ("Marry", 21), ("Jack", 14), ("Tom", 18))


val userDF = spark.createDataFrame(userData).toDF("name", "age")
userDF.show
```

```bash
+-----+---+
|  Leo| 16|
|Marry| 21|
| Jack| 14|
```

```scala
userDF.createOrReplaceTempView("user")
```

### 1.5.2 Spark Sql用法

#### 1.5.2.1 通过匿名函数注册UDF

下面的UDF的功能是计算某列的长度，该列的类型为String

##### 1.5.2.1.1 注册

*   Spark2.x:

```scala
spark.udf.register("strLen", (str: String) => str.length())
```

*   Spark1.x:

```scala
sqlContext.udf.register("strLen", (str: String) => str.length())
```

##### 1.5.2.2.2 使用

仅以Spark2.x为例

```scala
spark.sql("select name,strLen(name) as name_len from user").show
```

```bash
+-----+--------+
|  Leo|       3|
|Marry|       5|
| Jack|       4|
```

#### 1.5.2.2 通过实名函数注册UDF

实名函数的注册有点不同，要在后面加 \_(注意前面有个空格)  
定义一个实名函数

```scala
/**
 * 根据年龄大小返回是否成年 成年：true,未成年：false
*/
def isAdult(age: Int) = {
  if (age < 18) {
    false
  } else {
    true
  }

}
```

注册（仅以Spark2.x为例）

```scala
spark.udf.register("isAdult", isAdult _)
```

至于使用都是一样的

#### 1.5.2.3 关于spark.udf和sqlContext.udf

在Spark2.x里，两者实际最终都是调用的spark.udf  
sqlContext.udf源码

```bash
def udf: UDFRegistration = sparkSession.udf
```

可以看到调用的是sparkSession的udf，即spark.udf

### 1.5.3 DataFrame用法

DataFrame的udf方法虽然和Spark Sql的名字一样，但是属于不同的类，它在org.apache.spark.sql.functions里，下面是它的用法

#### 1.5.3.1注册

```scala
import org.apache.spark.sql.functions._

val strLen = udf((str: String) => str.length())

val udf_isAdult = udf(isAdult _)
```

#### 1.5.3.2 使用

可通过withColumn和select使用,下面的代码已经实现了给user表添加两列的功能  
\* 通过看源码，下面的withColumn和select方法Spark2.0.0之后才有的，关于spark1.xDataFrame怎么使用注册好的UDF没有研究

```scala

userDF.withColumn("name_len", strLen(col("name"))).withColumn("isAdult", udf_isAdult(col("age"))).show

userDF.select(col("*"), strLen(col("name")) as "name_len", udf_isAdult(col("age")) as "isAdult").show
```

结果均为

```bash
+-----+---+--------+-------+

|  Leo| 16|       3|  false|
|Marry| 21|       5|   true|
| Jack| 14|       4|  false|
```

#### 1.5.3.3 withColumn和select的区别

可通过withColumn的源码看出withColumn的功能是实现增加一列，或者替换一个已存在的列，他会先判断DataFrame里有没有这个列名，如果有的话就会替换掉原来的列，没有的话就用调用select方法增加一列，所以如果我们的需求是增加一列的话，两者实现的功能一样，且最终都是调用select方法，但是withColumn会提前做一些判断处理，所以withColumn的性能不如select好。

**注：select方法和sql 里的select一样，如果新增的列名在表里已经存在，那么结果里允许出现两列列名相同但数据不一样。**

```Scala
/**
 * Returns a new Dataset by adding a column or replacing the existing column that has
 * the same name.
 *
 * @group untypedrel
 * @since 2.0.0
*/
def withColumn(colName: String, col: Column): DataFrame = {
  val resolver = sparkSession.sessionState.analyzer.resolver
  val output = queryExecution.analyzed.output
  val shouldReplace = output.exists(f => resolver(f.name, colName))
  if (shouldReplace) {
    val columns = output.map { field =>
      if (resolver(field.name, colName)) {
        col.as(colName)
      } else {
        Column(field)
      }
    }
    select(columns : _*)
  } else {
    select(Column("*"), col.as(colName))
  }
}
```

### 1.5.4 完整代码


下面的代码的功能是使用UDF给user表添加两列:name\_len、isAdult，每个输出结果都是一样的

```bash
+-----+---+--------+-------+

|  Leo| 16|       3|  false|
|Marry| 21|       5|   true|
| Jack| 14|       4|  false|
```

代码：

```scala
package com.dkl.leanring.spark.sql

import org.apache.spark.SparkContext
import org.apache.spark.sql.SQLContext
import org.apache.spark.SparkConf
import org.apache.spark.sql.SparkSession

/**
 * Spark Sql 用户自定义函数示例
 */
object UdfDemo {

  def main(args: Array[String]): Unit = {
    oldUdf
    newUdf
    newDfUdf
    oldDfUdf
  }

  /**
   * 根据年龄大小返回是否成年 成年：true,未成年：false
   */
  def isAdult(age: Int) = {
    if (age < 18) {
      false
    } else {
      true
    }

  }

  /**
   * 旧版本(Spark1.x)Spark Sql udf示例
   */
  def oldUdf() {

    
    val conf = new SparkConf()
      .setMaster("local")
      .setAppName("oldUdf")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)
    import sqlContext.implicits._

    
    val userData = Array(("Leo", 16), ("Marry", 21), ("Jack", 14), ("Tom", 18))
    
    val userDF = sc.parallelize(userData).toDF("name", "age")
    
    userDF.registerTempTable("user")

    
    sqlContext.udf.register("strLen", (str: String) => str.length())

    sqlContext.udf.register("isAdult", isAdult _)
    
    sqlContext.sql("select *,strLen(name)as name_len,isAdult(age) as isAdult from user").show
    
    sc.stop()

  }

  /**
   * 新版本(Spark2.x)Spark Sql udf示例
   */
  def newUdf() {
    
    val spark = SparkSession.builder().appName("newUdf").master("local").getOrCreate()

    
    val userData = Array(("Leo", 16), ("Marry", 21), ("Jack", 14), ("Tom", 18))

    
    val userDF = spark.createDataFrame(userData).toDF("name", "age")

    
    userDF.createOrReplaceTempView("user")

    
    spark.udf.register("strLen", (str: String) => str.length())
    
    spark.udf.register("isAdult", isAdult _)
    spark.sql("select *,strLen(name) as name_len,isAdult(age) as isAdult from user").show

    
    spark.stop()

  }

  /**
   * 新版本(Spark2.x)DataFrame udf示例
   */
  def newDfUdf() {
    val spark = SparkSession.builder().appName("newDfUdf").master("local").getOrCreate()

    
    val userData = Array(("Leo", 16), ("Marry", 21), ("Jack", 14), ("Tom", 18))

    
    val userDF = spark.createDataFrame(userData).toDF("name", "age")
    import org.apache.spark.sql.functions._
    
    val strLen = udf((str: String) => str.length())
    
    val udf_isAdult = udf(isAdult _)

    
    userDF.withColumn("name_len", strLen(col("name"))).withColumn("isAdult", udf_isAdult(col("age"))).show
    
    userDF.select(col("*"), strLen(col("name")) as "name_len", udf_isAdult(col("age")) as "isAdult").show

    
    spark.stop()
  }
  /**
   * 旧版本(Spark1.x)DataFrame udf示例
   * 注意，这里只是用的Spark1.x创建sc的和df的语法，其中注册udf在Spark1.x也是可以使用的的
   * 但是withColumn和select方法Spark2.0.0之后才有的，关于spark1.xDataFrame怎么使用注册好的UDF没有研究
   */
  def oldDfUdf() {
    
    val conf = new SparkConf()
      .setMaster("local")
      .setAppName("oldDfUdf")
    val sc = new SparkContext(conf)
    val sqlContext = new SQLContext(sc)
    import sqlContext.implicits._

    
    val userData = Array(("Leo", 16), ("Marry", 21), ("Jack", 14), ("Tom", 18))
    
    val userDF = sc.parallelize(userData).toDF("name", "age")
    import org.apache.spark.sql.functions._
    
    val strLen = udf((str: String) => str.length())
    
    val udf_isAdult = udf(isAdult _)

    
    userDF.withColumn("name_len", strLen(col("name"))).withColumn("isAdult", udf_isAdult(col("age"))).show
    
    userDF.select(col("*"), strLen(col("name")) as "name_len", udf_isAdult(col("age")) as "isAdult").show

    
    sc.stop()
  }

}
```

[![](https://wx2.sinaimg.cn/large/e44344dcly1ftvr5q8aftj215y0pkwgl.jpg)](https://z3.ax1x.com/2021/08/10/fY4dXT.jpg)

## 1.6 Spark SQL 解析流程

参考：https://zhuanlan.zhihu.com/p/58428916

SQL和DataFrame 进来会先解析成逻辑执行计划，从hive metastore或SessionCatalog里面拿一些表、字段的元数据信息，生成一个解析过的执行计划。经过spark的优化器，改变逻辑执行计划的逻辑结构，通过planner生成物理的执行计划。


**Spark SQL核心——Catalyst查询编译器**

[![Catalyst框架](https://img-blog.csdnimg.cn/20190918141632834.png "Catalyst框架")](https://img.imgdb.cn/item/6066e49e8322e6675ceb54e8.png)


由上图看出，Spark SQL 的解析流程为：

**1) 使用 SessionCatalog 保存元数据**

在解析SQL语句之前，会创建 SparkSession，或者如果是2.0之前的版本初始化 SQLContext

SparkSession 只是封装了 SparkContext 和 SQLContext 的创建而已，会把元数据保存在SessionCatalog中，涉及到表名，字段名称和字段类型

创建临时表或者视图，其实就会往Session 目录（Catalog）注册

**2) 解析SQL使用 ANTLR 生成未绑定的逻辑计划**

当调用 SparkSession 的SQL或者 SQLContext 的SQL方法，我们以2.0为准，就会使用 SparkSql Parser 进行解析SQL

使用的 ANTLR 进行词法解析和语法解析，它分为2个步骤来生成Unresolved LogicalPlan:

    词法分析：Lexical Analysis, 负责将token分组成符号类
    语法分析：构建一个分析树或者语法树AST

**3) 使用分析器Analyzer绑定逻辑计划**

在该阶段，Analyzer 会使用 Analyzer规则（Rule），并结合 SessionCatalog，对未绑定的逻辑计划进行解析，生成已绑定的逻辑计划

**4) 使用优化器Optimizer优化逻辑计划**

优化器也是会定义一套 Rules，利用这些Rule对逻辑计划和 Exepression 进行迭代处理，从而使得树的节点进行合并和优化。

> 常用的规则如谓词下推（Predicate Pushdown）、列裁剪（Column Pruning）、连接重排序（Join Reordering），基于成本的优化器（Cost-based Optmizer）等

**5) 使用SparkPlanner生成物理计划**

Spark Spanner 使用 Planning Strategies，对优化后的逻辑计划进行转换，生成可以执行的物理执行计划（Physical Plan）

**6) 使用 QueryExecution 执行物理计划**

此时调用 SparkPlan 的 execute 方法，底层其实已经再触发JOB了，然后返回RDD

经过上述的一整个流程，就完成了从用户编写的SQL语句（或DataFrame/Dataset），到Spark内部RDD的具体操作逻辑的转化。

## 1.7 sparkSession 读取和处理zip、gzip、excel等各种文件

### 1.7.1 当后缀名为zip、gzip，spark可以自动处理和读取

1. spark非常智能，如果一批压缩的zip和gzip文件，并且里面为一堆text文件时，可以用如下方式读取或者获取读取后的schema

```scala
spark.read.text("xxxxxxxx/xxxx.zip")
spark.read.text("xxxxxxxx/xxxx.zip").schema
spark.read.text("xxxxxxxx/xxxx.gz")
spark.read.text("xxxxxxxx/xxxx.gz").schema
```

2. 当压缩的一批text文件里面的内容为json时，还可以通过read.json读取，并且自动解析为json数据返回

```scala
spark.read.json("xxxxxxxx/xxxx.zip")
spark.read.json("xxxxxxxx/xxxx.zip").schema
spark.read.json("xxxxxxxx/xxxx.gz")
spark.read.json("xxxxxxxx/xxxx.gz").schema
```

备注：spark在读取text、zip、gzip等各种文件时，支持直接传入类似这样的通配符匹配路径

```scala
spark.read.text("xxxxxxxx/*.zip")
spark.read.text("xxxxxxxx/*")
```

spark读取文件内容时是按行处理的，如果需要将文件里面**多行处理为一行数据**，可以通过设置**multiLine=true（默认为false）**

```scala
spark.read.option("multiLine","true").json("xxxxxxxx/xxxx.zip")
```

3. 当zip或者gzip的文件没有任何后缀名或者后缀名不对时，那spark就无法自动读取了，但是此时可以通过类似如下的方式来读取

spark.read.format("binaryFile").load("dbfs:/xxx/data/xxxx/xxxx/2021/07/01/*")
读取到后，自己在代码中来解析处理读取的二进制文件数据

```scala
spark.read.format("binaryFile").load("dbfs:/xxx/data/xxxx/xxxx/2021/07/01/*").foreach(data=>{
     // data解析
    })
```

而且在读取到binaryFile文件后，还可以通过注册udf函数来进行处理

spark在读取数据转换为dataframe时，是通过DataFrameReader.scala来处理的。从中可以看到option选项除了支持multiLine外，还支持了很多，详情可以阅读[DataFrameReader.scala源码注释](https://github.com/apache/spark/blob/v3.1.2/sql/core/src/main/scala/org/apache/spark/sql/DataFrameReader.scala)

### 1.7.2 spark输出 csv / parquet / json文件

```scala
df.write.mode("overwrite").option("delimiter", "|").option("compression", "snappy").csv(outputPath)
```
- csv文件默认压缩格式为`none`, 其他可选格式`bzip2`, `gzip`, `lz4`, `snappy` and `deflate`
- parquet文件一般默认压缩格式为 `snappy`,其他可选格式`none`, `uncompressed`, `gzip`, `lzo`, `brotli`, `lz4`, and `zstd`
- json文件默认压缩格式为`none`, 其他可选格式`none`, `bzip2`, `gzip`, `lz4`, `snappy` and `deflate`
- text文件不支持多列，保存为多列的时候，可以用concat_ws来连接多列，或者df.rdd.saveAsTextFile

### 1.7.3 spark读写excel文件
引入如下依赖


```
Scala 2.12
groupId: com.crealytics
artifactId: spark-excel_2.12
version: <spark-version>_0.14.0
```


**1、从Excel文件创建DataFrame**
```
import org.apache.spark.sql._

 val spark: SparkSession = SparkSession.builder()
      .appName("Excel_Test")
      .getOrCreate()
      
val df = spark.read
    .format("com.crealytics.spark.excel")
    .option("dataAddress", "'My Sheet'!B3:C35") // Optional, default: "A1"
    .option("header", "true") // Required
    .option("treatEmptyValuesAsNulls", "false") // Optional, default: true
    .option("setErrorCellsToFallbackValues", "true") // Optional, default: false, where errors will be converted to null. If true, any ERROR cell values (e.g. #N/A) will be converted to the zero values of the column's data type.
    .option("usePlainNumberFormat", "false") // Optional, default: false, If true, format the cells without rounding and scientific notations
    .option("inferSchema", "false") // Optional, default: false
    .option("addColorColumns", "true") // Optional, default: false
    .option("timestampFormat", "MM-dd-yyyy HH:mm:ss") // Optional, default: yyyy-mm-dd hh:mm:ss[.fffffffff]
    .option("maxRowsInMemory", 20) // Optional, default None. If set, uses a streaming reader which can help with big files (will fail if used with xls format files)
    .option("excerptSize", 10) // Optional, default: 10. If set and if schema inferred, number of rows to infer schema from
    .option("workbookPassword", "pass") // Optional, default None. Requires unlimited strength JCE for older JVMs
    .schema(myCustomSchema) // Optional, default: Either inferred schema, or all columns are Strings
    .load("Worktime.xlsx")
```

**2、DataFrame输出为Excel文件**

```
df.write
  .format("com.crealytics.spark.excel")
  .option("dataAddress", "'My Sheet'!B3:C35")
  .option("header", "true")
  .option("dateFormat", "yy-mmm-d") // Optional, default: yy-m-d h:mm
  .option("timestampFormat", "mm-dd-yyyy hh:mm:ss") // Optional, default: yyyy-mm-dd hh:mm:ss.000
  .mode("append") // Optional, default: overwrite.
  .save("Worktime2.xlsx")
```


# 2. SparkHive
## 2.1 SparkSql整合Hive

### 2.1.1 安装MySQL并创建一个普通用户，并且授权

```sql
CREATE USER 'bigdata'@'%' IDENTIFIED BY '123568'; 
GRANT ALL PRIVILEGES ON hivedb.* TO 'bigdata'@'%' IDENTIFIED BY '123568' WITH GRANT OPTION;
FLUSH PRIVILEGES;
```


### 2.1.2 添加一个hive-site.xml

```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
   Licensed to the Apache Software Foundation (ASF) under one or more
   contributor license agreements.  See the NOTICE file distributed with
   this work for additional information regarding copyright ownership.
   The ASF licenses this file to You under the Apache License, Version 2.0
   (the "License"); you may not use this file except in compliance with
   the License.  You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

   Unless required by applicable law or agreed to in writing, software
   distributed under the License is distributed on an "AS IS" BASIS,
   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   See the License for the specific language governing permissions and
   limitations under the License.
-->
<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://linux03:3306/hivedb?createDatabaseIfNotExist=true</value>
    <description>JDBC connect string for a JDBC metastore</description>
  </property>

   <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
    <description>Driver class name for a JDBC metastore</description>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
    <description>username to use against metastore database</description>
  </property>

  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>123456</value>
    <description>password to use against metastore database</description>
  </property>

</configuration>
```

### 2.1.3 上传一个mysql连接驱动
```bash
bin/spark-sql --master spark://linux01:7077 --driver-class-path /root/mysql-connector-java-5.1.44.jar
```

### 2.1.4 修改DBS表中的DB_LOCATION_UIR

sparkSQL会在mysql上创建一个database，需要手动改一下DBS表中的DB_LOCATION_UIR改成hdfs的地址

### 2.1.5 重新启动SparkSQL的命令行

## 2.2 spark连接hive参看资料

spark连接hive（spark-shell和eclipse两种方式） 
https://blog.csdn.net/dkl12/article/details/80248716

Spark记录-Spark-Shell客户端操作读取Hive数据

https://www.cnblogs.com/xinfang520/p/7985939.html

https://www.jianshu.com/p/983ecc768b55

SparkSql实现Mysql到hive的数据流动

https://www.cnblogs.com/WinseterCheng/p/10340375.html

SparkSQL操作Hive Table（enableHiveSupport()） 
https://blog.csdn.net/zhao897426182/article/details/78435234/