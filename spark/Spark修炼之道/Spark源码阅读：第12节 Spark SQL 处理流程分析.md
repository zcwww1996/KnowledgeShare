[TOC]

来源：https://blog.csdn.net/lovehuangjiaju/article/details/50375431

下面的代码演示了通过Case Class进行表Schema定义的例子：

```scala
// sc is an existing SparkContext.
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
// this is used to implicitly convert an RDD to a DataFrame.
import sqlContext.implicits._

// Define the schema using a case class.
// Note: Case classes in Scala 2.10 can support only up to 22 fields. To work around this limit,
// you can use custom classes that implement the Product interface.
case class Person(name: String, age: Int)

// Create an RDD of Person objects and register it as a table.
val people = sc.textFile("examples/src/main/resources/people.txt").map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt)).toDF()
people.registerTempTable("people")

// SQL statements can be run by using the sql methods provided by sqlContext.
val teenagers = sqlContext.sql("SELECT name, age FROM people WHERE age >= 13 AND age <= 19")

// The results of SQL queries are DataFrames and support all the normal RDD operations.
// The columns of a row in the result can be accessed by field index:
teenagers.map(t => "Name: " + t(0)).collect().foreach(println)

// or by field name:
teenagers.map(t => "Name: " + t.getAs[String]("name")).collect().foreach(println)

// row.getValuesMap[T] retrieves multiple columns at once into a Map[String, T]
teenagers.map(_.getValuesMap[Any](List("name", "age"))).collect().foreach(println)
// Map("name" -> "Justin", "age" -> 19)
```

(1) sql方法返回DataFrame

```scala
def sql(sqlText: String): DataFrame = {
  DataFrame(this, parseSql(sqlText))
  }
```

其中parseSql(sqlText)方法生成相应的LogicalPlan得到，该方法源码如下：

```scala
//根据传入的sql语句，生成LogicalPlan
protected[sql] def parseSql(sql: String): LogicalPlan = ddlParser.parse(sql, false)
```

ddlParser对象定义如下：

```scala
protected[sql] val sqlParser = new SparkSQLParser(getSQLDialect().parse(_))
protected[sql] val ddlParser = new DDLParser(sqlParser.parse(_))
```

(2) 然后调用DataFrame的apply方法

```scala
private[sql] object DataFrame {
  def apply(sqlContext: SQLContext, logicalPlan: LogicalPlan): DataFrame = {
    new DataFrame(sqlContext, logicalPlan)
  }
}
```

可以看到，apply方法参数有两个，分别是SQLContext和LogicalPlan，调用的是DataFrame的构造方法，具体源码如下：

```scala
//DataFrame构造方法，该构造方法会自动对LogicalPlan进行分析，然后返回QueryExecution对象
def this(sqlContext: SQLContext, logicalPlan: LogicalPlan) = {
    this(sqlContext, {
      val qe = sqlContext.executePlan(logicalPlan)
      //判断是否已经创建，如果是则抛异常
      if (sqlContext.conf.dataFrameEagerAnalysis) {
        qe.assertAnalyzed()  // This should force analysis and throw errors if there are any
      }
      qe
    })
  }
```

(3) val qe = sqlContext.executePlan(logicalPlan) 返回QueryExecution， sqlContext.executePlan方法源码如下：

```scala
protected[sql] def executePlan(plan: LogicalPlan) =
    new sparkexecution.QueryExecution(this, plan)
```

QueryExecution类中表达了Spark执行SQL的主要工作流程，具体如下


```scala
class QueryExecution(val sqlContext: SQLContext, val logical: LogicalPlan) {

  @VisibleForTesting
  def assertAnalyzed(): Unit = sqlContext.analyzer.checkAnalysis(analyzed)

  lazy val analyzed: LogicalPlan = sqlContext.analyzer.execute(logical)

  lazy val withCachedData: LogicalPlan = {
    assertAnalyzed()
    sqlContext.cacheManager.useCachedData(analyzed)
  }

  lazy val optimizedPlan: LogicalPlan = sqlContext.optimizer.execute(withCachedData)

  // TODO: Don't just pick the first one...
  lazy val sparkPlan: SparkPlan = {
    SparkPlan.currentContext.set(sqlContext)
    sqlContext.planner.plan(optimizedPlan).next()
  }

  // executedPlan should not be used to initialize any SparkPlan. It should be
  // only used for execution.
  lazy val executedPlan: SparkPlan = sqlContext.prepareForExecution.execute(sparkPlan)

  /** Internal version of the RDD. Avoids copies and has no schema */
  //调用toRDD方法执行任务将结果转换为RDD
  lazy val toRdd: RDD[InternalRow] = executedPlan.execute()

  protected def stringOrError[A](f: => A): String =
    try f.toString catch { case e: Throwable => e.toString }

  def simpleString: String = {
    s"""== Physical Plan ==
       |${stringOrError(executedPlan)}
      """.stripMargin.trim
  }

  override def toString: String = {
    def output =
      analyzed.output.map(o => s"${o.name}: ${o.dataType.simpleString}").mkString(", ")

    s"""== Parsed Logical Plan ==
       |${stringOrError(logical)}
       |== Analyzed Logical Plan ==
       |${stringOrError(output)}
       |${stringOrError(analyzed)}
       |== Optimized Logical Plan ==
       |${stringOrError(optimizedPlan)}
       |== Physical Plan ==
       |${stringOrError(executedPlan)}
       |Code Generation: ${stringOrError(executedPlan.codegenEnabled)}
    """.stripMargin.trim
  }
}
```

可以看到，SQL的执行流程为 
1. Parsed Logical Plan：LogicalPlan 
2. Analyzed Logical Plan： 
`lazy val analyzed: LogicalPlan = sqlContext.analyzer.execute(logical) `
3. Optimized Logical Plan：lazy val optimizedPlan: `LogicalPlan = sqlContext.optimizer.execute(withCachedData) `
4. Physical Plan：`lazy val executedPlan: SparkPlan = sqlContext.prepareForExecution.execute(sparkPlan)`

可以调用results.queryExecution方法查看，代码如下：

```scala
scala> results.queryExecution
res1: org.apache.spark.sql.SQLContext#QueryExecution =
== Parsed Logical Plan ==
'Project [unresolvedalias('name)]
 'UnresolvedRelation [people], None

== Analyzed Logical Plan ==
name: string
Project [name#0]
 Subquery people
  LogicalRDD [name#0,age#1], MapPartitionsRDD[4] at createDataFrame at <console>:47

== Optimized Logical Plan ==
Project [name#0]
 LogicalRDD [name#0,age#1], MapPartitionsRDD[4] at createDataFrame at <console>:47

== Physical Plan ==
TungstenProject [name#0]
 Scan PhysicalRDD[name#0,age#1]

Code Generation: true
```
(4) 然后调用DataFrame的主构造器完成DataFrame的构造

```scala
class DataFrame private[sql](
    @transient val sqlContext: SQLContext,
    @DeveloperApi @transient val queryExecution: QueryExecution) extends Serializable
```

(5) 
当调用DataFrame的collect等方法时，便会触发执行executedPlan

```scala
def collect(): Array[Row] = withNewExecutionId {
    queryExecution.executedPlan.executeCollect()
  }
```

例如：

```scala
scala> results.collect
res6: Array[org.apache.spark.sql.Row] = Array([Michael], [Andy], [Justin])
```

整体流程图如下：

[![3.jpg](https://www.helloimg.com/images/2020/08/20/352a714b3b218d239.png)](https://ae03.alicdn.com/kf/U667f4f9ba5cb447aaa90e5b737eda164U.jpg)