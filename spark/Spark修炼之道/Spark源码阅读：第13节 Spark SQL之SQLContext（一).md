[TOC]

来源： https://blog.csdn.net/lovehuangjiaju/article/details/50427650

# 1. SQLContext的创建
SQLContext是Spark SQL进行结构化数据处理的入口，可以通过它进行DataFrame的创建及SQL的执行，其创建方式如下：

```scala
//sc为SparkContext
val sqlContext = new org.apache.spark.sql.SQLContext(sc)
```

其对应的源码为：

```scala
def this(sparkContext: SparkContext) = {
    this(sparkContext, new CacheManager, SQLContext.createListenerAndUI(sparkContext), true)
  }
```

其调用的是私有的主构造函数：

```scala
//1.主构造器中的参数CacheManager用于缓存查询结果
//在进行后续查询时会自动读取缓存中的数据
//2.SQLListener用于监听Spark scheduler事件，它继承自SparkListener
//3.isRootContext表示是否是根SQLContext
class SQLContext private[sql](
    @transient val sparkContext: SparkContext,
    @transient protected[sql] val cacheManager: CacheManager,
    @transient private[sql] val listener: SQLListener,
    val isRootContext: Boolean)
  extends org.apache.spark.Logging with Serializable {
```

当spark.sql.allowMultipleContexts设置为true时，则允许创建多个SQLContexts/HiveContexts，创建方法为newSession


```scala
def newSession(): SQLContext = {
    new SQLContext(
      sparkContext = sparkContext,
      cacheManager = cacheManager,
      listener = listener,
      isRootContext = false)
  }
```

其isRootContext 被设置为false，否则会抛出异常，因为root SQLContext只能有一个，其它SQLContext与root SQLContext共享SparkContext, CacheManager, SQLListener。如果spark.sql.allowMultipleContexts为false，则只允许一个SQLContext存在

# 2. 核心成员变量--catalog


```scala
protected[sql] lazy val catalog: Catalog = new SimpleCatalog(conf)
```

catalog用于注销表、注销表、判断表是否存在等，例如当DataFrame调用registerTempTable 方法时


```scala
val people = sc.textFile("examples/src/main/resources/people.txt").map(_.split(",")).map(p => Person(p(0), p(1).trim.toInt)).toDF()
people.registerTempTable("people")
```

会sqlContext的registerDataFrameAsTable方法


```scala
def registerTempTable(tableName: String): Unit = {
    sqlContext.registerDataFrameAsTable(this, tableName)
  }
```

sqlContext.registerDataFrameAsTable实质上调用的就是catalog的registerTable 方法：

```scala
private[sql] def registerDataFrameAsTable(df: DataFrame, tableName: String): Unit = {
    catalog.registerTable(TableIdentifier(tableName), df.logicalPlan)
  }
```

SimpleCatalog整体源码如下：

```scala
class SimpleCatalog(val conf: CatalystConf) extends Catalog {
  private[this] val tables = new ConcurrentHashMap[String, LogicalPlan]

  override def registerTable(tableIdent: TableIdentifier, plan: LogicalPlan): Unit = {
    tables.put(getTableName(tableIdent), plan)
  }

  override def unregisterTable(tableIdent: TableIdentifier): Unit = {
    tables.remove(getTableName(tableIdent))
  }

  override def unregisterAllTables(): Unit = {
    tables.clear()
  }

  override def tableExists(tableIdent: TableIdentifier): Boolean = {
    tables.containsKey(getTableName(tableIdent))
  }

  override def lookupRelation(
      tableIdent: TableIdentifier,
      alias: Option[String] = None): LogicalPlan = {
    val tableName = getTableName(tableIdent)
    val table = tables.get(tableName)
    if (table == null) {
      throw new NoSuchTableException
    }
    val tableWithQualifiers = Subquery(tableName, table)

    // If an alias was specified by the lookup, wrap the plan in a subquery so that attributes are
    // properly qualified with this alias.
    alias.map(a => Subquery(a, tableWithQualifiers)).getOrElse(tableWithQualifiers)
  }

  override def getTables(databaseName: Option[String]): Seq[(String, Boolean)] = {
    tables.keySet().asScala.map(_ -> true).toSeq
  }

  override def refreshTable(tableIdent: TableIdentifier): Unit = {
    throw new UnsupportedOperationException
  }
}
```

# 3. 核心成员变量--sqlParser
sqlParser在SQLContext的定义：

```scala
protected[sql] val sqlParser = new SparkSQLParser(getSQLDialect().parse(_))
```

SparkSQLParser为顶级的Spark SQL解析器，对Spark SQL支持的SQL语法进行解析，其定义如下：


```scala
private[sql] class SparkSQLParser(fallback: String => LogicalPlan) extends AbstractSparkSQLParser
```

fallback函数用于解析其它非Spark SQL Dialect的语法。 
Spark SQL Dialect支持的关键字包括：

```scala
  protected val AS = Keyword("AS")
  protected val CACHE = Keyword("CACHE")
  protected val CLEAR = Keyword("CLEAR")
  protected val DESCRIBE = Keyword("DESCRIBE")
  protected val EXTENDED = Keyword("EXTENDED")
  protected val FUNCTION = Keyword("FUNCTION")
  protected val FUNCTIONS = Keyword("FUNCTIONS")
  protected val IN = Keyword("IN")
  protected val LAZY = Keyword("LAZY")
  protected val SET = Keyword("SET")
  protected val SHOW = Keyword("SHOW")
  protected val TABLE = Keyword("TABLE")
  protected val TABLES = Keyword("TABLES")
  protected val UNCACHE = Keyword("UNCACHE")
```

# 4. 核心成员变量--ddlParser
用于解析DDL（Data Definition Language 数据定义语言）


```scala
protected[sql] val ddlParser = new DDLParser(sqlParser.parse(_))
```

其支持的关键字有：

```scala
  protected val CREATE = Keyword("CREATE")
  protected val TEMPORARY = Keyword("TEMPORARY")
  protected val TABLE = Keyword("TABLE")
  protected val IF = Keyword("IF")
  protected val NOT = Keyword("NOT")
  protected val EXISTS = Keyword("EXISTS")
  protected val USING = Keyword("USING")
  protected val OPTIONS = Keyword("OPTIONS")
  protected val DESCRIBE = Keyword("DESCRIBE")
  protected val EXTENDED = Keyword("EXTENDED")
  protected val AS = Keyword("AS")
  protected val COMMENT = Keyword("COMMENT")
  protected val REFRESH = Keyword("REFRESH")
```

主要做三件事，分别是创建表、描述表和更新表

```scala
protected lazy val ddl: Parser[LogicalPlan] = createTable | describeTable | refreshTable
```

createTable方法具有如下（具体功能参考注释说明）：

```scala
/**
   * `CREATE [TEMPORARY] TABLE avroTable [IF NOT EXISTS]
   * USING org.apache.spark.sql.avro
   * OPTIONS (path "../hive/src/test/resources/data/files/episodes.avro")`
   * or
   * `CREATE [TEMPORARY] TABLE avroTable(intField int, stringField string...) [IF NOT EXISTS]
   * USING org.apache.spark.sql.avro
   * OPTIONS (path "../hive/src/test/resources/data/files/episodes.avro")`
   * or
   * `CREATE [TEMPORARY] TABLE avroTable [IF NOT EXISTS]
   * USING org.apache.spark.sql.avro
   * OPTIONS (path "../hive/src/test/resources/data/files/episodes.avro")`
   * AS SELECT ...
   */
  protected lazy val createTable: Parser[LogicalPlan] = {
    // TODO: Support database.table.
    (CREATE ~> TEMPORARY.? <~ TABLE) ~ (IF ~> NOT <~ EXISTS).? ~ tableIdentifier ~
      tableCols.? ~ (USING ~> className) ~ (OPTIONS ~> options).? ~ (AS ~> restInput).? ^^ {
      case temp ~ allowExisting ~ tableIdent ~ columns ~ provider ~ opts ~ query =>
        if (temp.isDefined && allowExisting.isDefined) {
          throw new DDLException(
            "a CREATE TEMPORARY TABLE statement does not allow IF NOT EXISTS clause.")
        }

        val options = opts.getOrElse(Map.empty[String, String])
        if (query.isDefined) {
          if (columns.isDefined) {
            throw new DDLException(
              "a CREATE TABLE AS SELECT statement does not allow column definitions.")
          }
          // When IF NOT EXISTS clause appears in the query, the save mode will be ignore.
          val mode = if (allowExisting.isDefined) {
            SaveMode.Ignore
          } else if (temp.isDefined) {
            SaveMode.Overwrite
          } else {
            SaveMode.ErrorIfExists
          }

          val queryPlan = parseQuery(query.get)
          CreateTableUsingAsSelect(tableIdent,
            provider,
            temp.isDefined,
            Array.empty[String],
            mode,
            options,
            queryPlan)
        } else {
          val userSpecifiedSchema = columns.flatMap(fields => Some(StructType(fields)))
          CreateTableUsing(
            tableIdent,
            userSpecifiedSchema,
            provider,
            temp.isDefined,
            options,
            allowExisting.isDefined,
            managedIfNoPath = false)
        }
    }
  }
```

describeTable及refreshTable代码如下：

```scala
 /*
   * describe [extended] table avroTable
   * This will display all columns of table `avroTable` includes column_name,column_type,comment
   */
  protected lazy val describeTable: Parser[LogicalPlan] =
    (DESCRIBE ~> opt(EXTENDED)) ~ tableIdentifier ^^ {
      case e ~ tableIdent =>
        DescribeCommand(UnresolvedRelation(tableIdent, None), e.isDefined)
    }

  protected lazy val refreshTable: Parser[LogicalPlan] =
    REFRESH ~> TABLE ~> tableIdentifier ^^ {
      case tableIndet =>
        RefreshTable(tableIndet)
    }
```
