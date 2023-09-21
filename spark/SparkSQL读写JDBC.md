[TOC]

参考文档：[Spark基础：读写JDBC](https://zhuanlan.zhihu.com/p/148556552)

本文会涉及一些Spark的源码分析，使用的版本是`org.apache.spark:spark-sql_2.12:3.2.1`


# 1. Spark SQL读写JDBC的基本操作
首先简单介绍一下Spark SQL读写JDBC的基本操作和参数配置。

Spark SQL支持通过JDBC直接读取数据库中的数据，这个特性是基于JdbcRDD实现，返回值作为DataFrame返回或者注册成Spark SQL的临时表，用户可以在数据源选项中配置JDBC相关的连接参数，user和password一般是必须提供的参数，常用的参数列表如下：

| 参数名称 | 参数说明 | 作用范围 |
| --- | --- | --- |
| url | 数据库连接jdbc url | read/write |
| user | 数据库登陆用户名 | read/write |
| password | 数据库登陆密码 | read/write |
| dbtable | 需要读取或者写入的JDBC表。注意不能同时配置dbtable和query。 | read/write |
| query | query用于指定from后面的子查询，拼接成的sql如下：SELECT FROM () spark\_gen\_alias 。 注意dbtable和query不能同时使用； 不允许同时使用partitionColumn和query。 | read/write |
| driver | jdbc驱动driver | read/write |
| partitionColumn, lowerBound, upperBound | 指定时这三项需要同时存在，描述了worker如何并行读取数据库。 其中partitionColumn必须是数字、date、timestamp。 lowerBound和upperBound只是决定了分区的步长，而不会过滤数据，因此表中所有的数据都会被分区返回 | read |
| numPartitions | 读写时的最大分区数。这也决定了连接JDBC的最大连接数，如果并行度超过该配置，将会使用coalesce(partition)来降低并行度。 | read/write |
| queryTimeout | driver执行statement的等待时间，0意味着没有限制。 写入的时候这个选项依赖于底层是如何实现setQueryTimeout的 | read/write |
| fetchsize | fetch的大小，决定了每一个fetch，拉取多少数据量。 | read |
| batchsize | batch大小，决定插入时的并发大小，默认1000。 | write |
| isolationLevel | 事务隔离的等级，作用于当前连接，默认是READ\_UNCOMMITTED。 可以配置成NONE, READ\_COMMITTED, READ\_UNCOMMITTED, REPEATABLE\_READ, SERIALIZABLE。 依赖于底层jdbc提供的事务隔离。 | write |
| truncate | 当使用SaveMode.Overwrite时，该配置才会生效。 默认是false，会先删除表再创建表，会改变原有的表结构。 如果是true，则会直接执行truncate table，但是由于各个DBMS的行为不同，使用它并不总是安全的，并不是所有的DBMS都支持truncate。 | write |

基础代码如下：

```scala


Dataset<Row> jdbcDF = spark.read()
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .load();

Properties connectionProperties = new Properties();
connectionProperties.put("user", "username");
connectionProperties.put("password", "password");
Dataset<Row> jdbcDF2 = spark.read()
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);


jdbcDF.write()
  .format("jdbc")
  .option("url", "jdbc:postgresql:dbserver")
  .option("dbtable", "schema.tablename")
  .option("user", "username")
  .option("password", "password")
  .save();

jdbcDF2.write()
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);


jdbcDF.write()
  .option("createTableColumnTypes", "name CHAR(64), comments VARCHAR(1024)")
  .jdbc("jdbc:postgresql:dbserver", "schema.tablename", connectionProperties);

```

简单介绍了一下SparkSQL读写JDBC的操作，接下来介绍一下我在项目上遇到的一些问题。

1.  我想利用数据库索引，能提高查询速度，能过滤掉一些数据，并不想全表查询生成DataFrame，然后通过DataFrame操作过滤数据。（虽说我也不知道性能能提高多少，有懂哥也可以帮忙解释一下）
2.  dbtable和query应该用什么？到底有什么样的区别？
3.  本来想指定`numPartitions`，用来提高并行度，但是并不管用，通过源码查看，发现还是需要跟`partitionColumn, lowerBound, upperBound`三个参数一起使用才会生效，但是并不是每个表都有符合分区的字段，比如查询的字段都是字符串类型，那就只能通过调用`DataFrame.repartition(numPartitions)`，但是需要先把数据全部都查出来再进行分区。
4.  使用`DataFrame.write()`方法很暴力，竟然会改变表的结构？

# 2. 常见问题
## 2.1 Q1: mysql内部查询与dataframe哪个查询性能更好？

看下面的方法。方式1是通过query中写sql语句带着查询的条件，并且字段是索引字段。方式2则是通过操作DataFrame来过滤的。这两种方式哪种性能更好呢？

```Scala
// 方式1
spark
  .read()
  .format("jdbc")
  .option(
  "url",
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true")
  .option("user", "root")
  .option("password", "root")
  .option("driver", "com.mysql.cj.jdbc.Driver")
  .option("query", "select * from user where id > 10")
  .load();

// 方式2
spark
  .read()
  .format("jdbc")
  .option(
  "url",
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true")
  .option("user", "root")
  .option("password", "root")
  .option("driver", "com.mysql.cj.jdbc.Driver")
  .option("query", "select * from user where id > 10")
  .load()
  .where("id > 10")

```

除了性能，还有另外一个话题也可以说道说道。一般来说传给SQL的查询参数是固定的，但是每次查询的参数值是不固定的。例如`select * from user where birthday >= ? and birthday < ?`，根据生日的区间来查看用户列表。

如果采用方式1，有个很尴尬的问题，参数值如何传进去呢？看Spark SQL关于JDBC DataSource的文档发现并没有关于这一块的配置，通过源码查看也并未发现可以传参数进去的方式。

目前我能想到的方式有两种。

第一种通过字符串替换的方式，生成对应的SQL语句。举例来说，传给main方法参数有

*   `sql=select * from user where birthday >= ${startTime} and birthday < ${endTime}`
*   `params=[startTime: 1995-02-01, endTime: 2000-02-01]`

然后通过字符串替换的方式，将sql替换为`select * from user where birthday >= '1995-02-01' and birthday < '2000-02-01'`

但上述的方式有个致命的缺陷，就是SQL注入，那么如何解决SQL注入的问题呢？

其实答案也很简单，通过`prepareStatement`来set参数，那么就需要params里带着参数的类型，例如`params=[{"name": "startTime", "value": '1995-02-01', "type": "Date"}, {"name": "endTime", "value": '2000-02-01', "type": "Date"}]`。那么可以通过`prepareStatement.setDate`方法给参数赋值即可，最后通过`prepareStatement.toString()`方法来获取到预处理之后的SQL，这样就能保证SQL不会被注入了。

虽说这个方法避免了SQL的注入，但是`prepareStatement.toString()`具体实现依赖各个数据库提供的驱动包，MySQL是会打印预处理之后的SQL，但是不能保证其他数据库（例如Oracle）也会有相同的行为，这个需要对其他数据库也要充分的调研。

第二种方法则是通过SparkSQL方式

```Scala
spark.sql("set startTime = 1995-02-01");
spark.sql("set endTime = 2000-02-01");
spark
  .read()
  .format("jdcb")
  .option(
  "url",
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true")
  .option("user", "root")
  .option("password", "root")
  .option("driver", "com.mysql.cj.jdbc.Driver")
  .option("query", "select * from user")
  .load().createOrReplaceTempView("t1");
spark.sql("select * from t1 where birthday >= ${startTime} and birthday < ${endTime}")

```

Spark SQL这种set params的方式，盲猜应该也是直接替换字符串的方式。

至于这两种方式孰优孰劣，个人觉得还是得测试一下性能才能确定，虽说看上去Spark SQL的方式更加通用一下，但是缺点还是需要获取到全表数据，一旦数据量很大，会很影响性能的，也会丧失数据库本身索引的优势。

当然这只是自己的一厢情愿，其实我还没具体测试过。。。

## 2.2 Q2: dbtable和query到底有什么区别

根据官方文档上的说明，dbtable和query不能同时使用。

dbtable可以填写表名，也可以是一个sql语句作为子查询。

query可以填写sql语句，但不可以填写表名。

```scala
option("dbtable", "user"); 
option("dbtable", "(select * from user) as subQuery"); 
option("dbtable", "select * from user"); 

option("query", "select * from user"); 
option("query", "user"); 
option("query", "(select * from user) as subQuery"); 

```

看正反示例代码也可以发现，dbtable和query是冲突的，所以不能同时使用。

再来看看源代码，也能充分说明dbtable和query的区别

```scala

val tableOrQuery = (parameters.get(JDBC_TABLE_NAME), parameters.get(JDBC_QUERY_STRING)) match {
    
    case (Some(name), Some(subquery)) =>
      throw QueryExecutionErrors.cannotSpecifyBothJdbcTableNameAndQueryError(
        JDBC_TABLE_NAME, JDBC_QUERY_STRING)
    
    case (None, None) =>
      throw QueryExecutionErrors.missingJdbcTableNameAndQueryError(
        JDBC_TABLE_NAME, JDBC_QUERY_STRING)
    
    case (Some(name), None) =>
      if (name.isEmpty) {
        throw QueryExecutionErrors.emptyOptionError(JDBC_TABLE_NAME)
      } else {
        name.trim
      }
    
    case (None, Some(subquery)) =>
      if (subquery.isEmpty) {
        throw QueryExecutionErrors.emptyOptionError(JDBC_QUERY_STRING)
      } else {
        s"(${subquery}) SPARK_GEN_SUBQ_${curId.getAndIncrement()}"
      }
  }




val sqlText = s"SELECT $columnList FROM ${options.tableOrQuery} $myWhereClause" +
      s" $getGroupByClause"

```

除了配置上的区别，还有一个区别就是如果指定了partitionColumn，lowerBound，upperBound，则必须使用dbtable，不能使用query。这一点在官方文档里有说明，源代码里也有体现。

```scala

require(!(parameters.get(JDBC_QUERY_STRING).isDefined && partitionColumn.isDefined),
  s"""
     |Options '$JDBC_QUERY_STRING' and '$JDBC_PARTITION_COLUMN' can not be specified together.
     |Please define the query using `$JDBC_TABLE_NAME` option instead and make sure to qualify
     |the partition columns using the supplied subquery alias to resolve any ambiguity.
     |Example :
     |spark.read.format("jdbc")
     |  .option("url", jdbcUrl)
     |  .option("dbtable", "(select c1, c2 from t1) as subq")
     |  .option("partitionColumn", "c1")
     |  .option("lowerBound", "1")
     |  .option("upperBound", "100")
     |  .option("numPartitions", "3")
     |  .load()
   """.stripMargin
)

```

所以如果指定了partitionColumn就必须使用dbtable。不过，我没整明白，dbtable和query的实际作用一样，最后都是select语句，为啥partitionColumn就必须使用dbtable？不知道是不是spark对这一块是否还有其他看法。

## 2.3 Q3: numPartitions, partitionColumn, lowerBound, upperBound四个参数之间的关系到底是什么？


1.  `numPartitions`：读、写的**最大**分区数，也决定了开启数据库连接的数目。注意**最大**两个字，也就是说你指定了32个分区，它也不一定就真的分32个分区了。比如：在读的时候，即便指定了`numPartitions`为任何大于1的值，如果没有指定分区规则，就只有一个`task`去执行查询。
2.  `partitionColumn, lowerBound, upperBound`：指定读数据时的分区规则。要使用这三个参数，必须定义`numPartitions`，而且这三个参数不能单独出现，要用就必须全部指定。而且`lowerBound, upperBound`不是过滤条件，只是用于决定分区跨度。

```scala
spark
  .read()
  .format("jdbc")
  .option(
  "url",
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true")
  .option("user", "root")
  .option("password", "root")
  .option("driver", "com.mysql.cj.jdbc.Driver")
  .option("dbtable", "(select * from user) as subQuery")
  .option("numPartitions", "10")
  .load().rdd().getNumPartitions(); // 结果为1

spark
  .read()
  .format("jdbc")
  .option(
  "url",
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true")
  .option("user", "root")
  .option("password", "root")
  .option("driver", "com.mysql.cj.jdbc.Driver")
  //            .option("query", "select * from user")
  .option("dbtable", "(select * from user) as subQuery")
  .option("partitionColumn", "id")
  .option("lowerBound", "1")
  .option("upperBound", "50")
  .option("numPartitions", "10")
  .load().rdd().getNumPartitions(); // 结果为10

```

对于`numPartitions`的使用有我感到疑惑的地方，官方文档也没有说明的很清楚，如果说指定了`numPartitions`但是不指定分区规则，这个参数相当于没用，如果需要指定分区规则就需要用到`partitionColumn, lowerBound, upperBound`这三个字段，官网在介绍`numPartitions`并没有说明一定要这三个字段才生效，不知道有没有懂哥知道其他指定分区的方法。

所以我扒了一下源码，浅析了一下。首先先看看JDBCOptions里的描述

```scala

require((partitionColumn.isEmpty && lowerBound.isEmpty && upperBound.isEmpty) ||
  (partitionColumn.isDefined && lowerBound.isDefined && upperBound.isDefined &&
    numPartitions.isDefined),
  s"When reading JDBC data sources, users need to specify all or none for the following " +
    s"options: '$JDBC_PARTITION_COLUMN', '$JDBC_LOWER_BOUND', '$JDBC_UPPER_BOUND', " +
    s"and '$JDBC_NUM_PARTITIONS'")


```

具体使用的逻辑在`org.apache.spark.sql.execution.datasources.jdbc.JDBCRelation#columnPartition`

```scala
def columnPartition(
    schema: StructType,
    resolver: Resolver,
    timeZoneId: String,
    jdbcOptions: JDBCOptions): Array[Partition] = {
  val partitioning = {
    import JDBCOptions._

    val partitionColumn = jdbcOptions.partitionColumn
    val lowerBound = jdbcOptions.lowerBound
    val upperBound = jdbcOptions.upperBound
    val numPartitions = jdbcOptions.numPartitions

    
    if (partitionColumn.isEmpty) {
      assert(lowerBound.isEmpty && upperBound.isEmpty, "When 'partitionColumn' is not " +
        s"specified, '$JDBC_LOWER_BOUND' and '$JDBC_UPPER_BOUND' are expected to be empty")
      null
    } else {
      
      assert(lowerBound.nonEmpty && upperBound.nonEmpty && numPartitions.nonEmpty,
        s"When 'partitionColumn' is specified, '$JDBC_LOWER_BOUND', '$JDBC_UPPER_BOUND', and " +
          s"'$JDBC_NUM_PARTITIONS' are also required")

      
      val (column, columnType) = verifyAndGetNormalizedPartitionColumn(
        schema, partitionColumn.get, resolver, jdbcOptions)
      
      val lowerBoundValue = toInternalBoundValue(lowerBound.get, columnType, timeZoneId)
      val upperBoundValue = toInternalBoundValue(upperBound.get, columnType, timeZoneId)
      JDBCPartitioningInfo(
        column, columnType, lowerBoundValue, upperBoundValue, numPartitions.get)
    }
  }

  
  
  if (partitioning == null || partitioning.numPartitions <= 1 ||
    partitioning.lowerBound == partitioning.upperBound) {
    return Array[Partition](JDBCPartition(null, 0))
  }

  val lowerBound = partitioning.lowerBound
  val upperBound = partitioning.upperBound
  require (lowerBound <= upperBound,
    "Operation not allowed: the lower bound of partitioning column is larger than the upper " +
    s"bound. Lower bound: $lowerBound; Upper bound: $upperBound")

  val boundValueToString: Long => String =
    toBoundValueInWhereClause(_, partitioning.columnType, timeZoneId)
  
  
  val numPartitions =
    if ((upperBound - lowerBound) >= partitioning.numPartitions || 
        (upperBound - lowerBound) < 0) {
      partitioning.numPartitions
    } else {
      
      upperBound - lowerBound
    }

  val upperStride = (upperBound / BigDecimal(numPartitions))
    .setScale(18, RoundingMode.HALF_EVEN)
  val lowerStride = (lowerBound / BigDecimal(numPartitions))
    .setScale(18, RoundingMode.HALF_EVEN)

  val preciseStride = upperStride - lowerStride
  
  val stride = preciseStride.toLong
  
  val lostNumOfStrides = (preciseStride - stride) * numPartitions / stride
  val lowerBoundWithStrideAlignment = lowerBound +
    ((lostNumOfStrides / 2) * stride).setScale(0, RoundingMode.HALF_UP).toLong

  var i: Int = 0
  val column = partitioning.column
  var currentValue = lowerBoundWithStrideAlignment
  
  val ans = new ArrayBuffer[Partition]()
  
  
  while (i < numPartitions) {
    val lBoundValue = boundValueToString(currentValue)
    
    val lBound = if (i != 0) s"$column >= $lBoundValue" else null
    currentValue += stride
    val uBoundValue = boundValueToString(currentValue)
    
    val uBound = if (i != numPartitions - 1) s"$column < $uBoundValue" else null
    val whereClause =
      if (uBound == null) {
        lBound
      } else if (lBound == null) {
        s"$uBound or $column is null"
      } else {
        s"$lBound AND $uBound"
      }
    ans += JDBCPartition(whereClause, i)
    i = i + 1
  }
  val partitions = ans.toArray
  logInfo(s"Number of partitions: $numPartitions, WHERE clauses of these partitions: " +
    partitions.map(_.asInstanceOf[JDBCPartition].whereClause).mkString(", "))
  partitions
}

```

小总结一下，实际上`numPartitions, partitionColumn, lowerBound, upperBound`这四个参数，如果设置了，则4个都需要设置，如果只设置了`numPartitions`是无效的，原因源码里有说明。

那如果dbtable里并没有满足可以分区的字段，比如都是String类型的字段，那该如何分区呢？其实，当时刚看到`numPartitions`我的第一想法是如果没有指定`partitionColumn`，Spark会根据数据库分页的方式来做分区，虽说最后调研的结果看起来是我想多了，但这确实也给我这个问题的答案提供了思路，大致思路如下：

```scala
// 需要先执行select count(*) from <dbtable>
int count = 1000
int numPartitions = 10
int stride = count / numPartitions
SparkSession spark =
  SparkSession.builder().appName("local-test").master("local[2]").getOrCreate()
String[] predicates = new String[numPartitions]
// 拼接where条件，这里除了分页，也可以是其他可以分区的条件。
for (int i = 0
  predicates[i] = String.format("1 = 1 limit %d, %d", i * stride, stride);
}
Properties properties = new Properties();
properties.put("user", "root");
properties.put("password", "root");
properties.put("driver", "com.mysql.cj.jdbc.Driver");
Dataset<Row> df =
  spark
  .read()
  .jdbc(
  "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true",
  "user",
  predicates,
  properties);
System.out.println(df.rdd().getNumPartitions()); // 10


```

## 2.4 Q4: Spark SQL对于jdbc write方法很暴力，竟然会改变表的结构？


```scala
@Test
public void testJDBCWriter() {
  SparkSession spark =
    SparkSession.builder().appName("local-test").master("local[2]").getOrCreate();
  StructType structType =
    new StructType(
    new StructField[] {
      new StructField("id", DataTypes.IntegerType, false, Metadata.empty()),
      new StructField("name", DataTypes.StringType, false, Metadata.empty()),
      new StructField("createTime", DataTypes.TimestampType, false, Metadata.empty())
    });
  Dataset<Row> df =
    spark
    .read()
    .option("header", "true")
    .schema(structType)
    .format("csv")
    .load(SparkSQLDataSourceDemo.class.getClassLoader().getResource("user.csv").getPath());
  Properties properties = new Properties();
  properties.put("user", "root");
  properties.put("password", "root");
  properties.put("driver", "com.mysql.cj.jdbc.Driver");
  df.write()
    .jdbc(
    "jdbc:mysql://localhost:3306/test?createDatabaseIfNotExist=true&serverTimezone=UTC&useSSL=true",
    "user",
    properties);
}

```

如果user表没有创建，dataframe jdbc write就会自己根据schema和表名自己在jdbc里创建表。

如果user表已经存在，默认的`SaveMode.ErrorIfExists`如果表已经存在，会报`Table or view 'user' already exists.`的错误。

如果user表已经存在，`SaveMode`选择`Append`，则会追加到表里，但是如果配置了主键或者唯一约束，相同的数据会报错。

如果user表已经存在，`SaveMode`选择`Overwrite`，并且`truncate`为`false`。会先删除表再重建表，会改变原先表结构，例如原表的表结构里有主键，索引，或者某字段类型（比如varchar），经过先删除表再重建表的操作，原来的主键，索引已经没有了，字段的类型也被改变了（比如varchar变成了text）。

源代码如下：

```scala

override def createRelation(
      sqlContext: SQLContext,
      mode: SaveMode,
      parameters: Map[String, String],
      df: DataFrame): BaseRelation = {
    val options = new JdbcOptionsInWrite(parameters)
    val isCaseSensitive = sqlContext.conf.caseSensitiveAnalysis

    val conn = JdbcUtils.createConnectionFactory(options)()
    try {
      val tableExists = JdbcUtils.tableExists(conn, options)
      if (tableExists) {
        mode match {
          case SaveMode.Overwrite =>
            if (options.isTruncate && isCascadingTruncateTable(options.url) == Some(false)) {
              
              truncateTable(conn, options)
              val tableSchema = JdbcUtils.getSchemaOption(conn, options)
              saveTable(df, tableSchema, isCaseSensitive, options)
            } else {
              
              dropTable(conn, options.table, options)
              createTable(conn, options.table, df.schema, isCaseSensitive, options)
              saveTable(df, Some(df.schema), isCaseSensitive, options)
            }

          case SaveMode.Append =>
            val tableSchema = JdbcUtils.getSchemaOption(conn, options)
            saveTable(df, tableSchema, isCaseSensitive, options)

          case SaveMode.ErrorIfExists =>
            throw QueryCompilationErrors.tableOrViewAlreadyExistsError(options.table)

          case SaveMode.Ignore =>
            
            
            
        }
      } else {
        createTable(conn, options.table, df.schema, isCaseSensitive, options)
        saveTable(df, Some(df.schema), isCaseSensitive, options)
      }
    } finally {
      conn.close()
    }

    createRelation(sqlContext, parameters)
  }

```

那如果我们想更好的实现Overwrite，那应该怎么实现呢？我的解决方案是使用`Dataframe.foreachPartition()`方法实现，实现思路如下：

```scala
df.foreachPartition(
        partition -> {
          Connection connection = null
          PreparedStatement ps = null
          try {
            Class.forName(driverClassName)
            connection = DriverManager.getConnection(url, username, password)
            // sql可以根据是Append还是Overwrite来提供
            ps = connection.prepareStatement(sql)
            connection.setAutoCommit(false)
            connection.setTransactionIsolation(transactionLevel)
            int count = 0
            while (partition.hasNext()) {
              Row row = partition.next()
              // 根据Column类型来setParamete。
              setParameter(ps, row)
              count++
              if (count < batchSize) {
                ps.addBatch()
              } else {
                ps.executeBatch()
                connection.commit()
                count = 0
              }
            }
            if (count != 0) {
              ps.executeBatch()
              connection.commit()
            }
          } catch (Exception e) {
            if (connection != null) {
              connection.rollback()
            }
          } finally {
            if (ps != null) {
              ps.close()
            }
            if (connection != null) {
              connection.close()
            }
          }
        })

```

对上述代码（1）处的说明如下：

1.  sql可以根据写入模式是Append还是Overwrite来生成

2.  如果是Append，则可以提供insert语句

3.  如果是Overwrite，如果是MySQL，可以提供Replace语句也可以提供insert on duplicate key update语句，但是其他的DBMS，需要看看有没有其他语句能够达成这样的功效。

4.  对于第三点，是依赖主键或者其他唯一约束的，如果表里没有主键或者其他数据库没有类似MySQL那种insert on duplicate key update语句，也可以自定义唯一的条件语句，通过update来实现，如果update返回0，则执行insert。`PreparedStatement.executeUpdate()`会返回执行成功的条数，根据这个来判断即可。

# 3. 总结
本篇文章主要讲述了Spark SQL读写JDBC一些细节，通过一些案例来讲述，总结如下：

1.  Spark SQL读写JDBC，一些参数是必须的，例如`url, driver, user, password`。
2.  讨论了一下如何替换sql参数，避免sql注入。主要还是需要通过`PreparedStatement.setXXX`方式实现，缺点就是`PreparedStatement.toString()`方法依赖各个驱动包的实现。也可以通过Spark SQL set语法实现，但主要问题是这么做就需要全表查询，封装成DataFrame再操作，如果数据量很大，会很消耗内存，如果能够利用数据库查询语句，不仅能够过滤出符合条件的数据，也能利用数据库索引的优势提高查询效率。虽说这是我一厢情愿的想法，还有待验证。
3.  dbtable和query的区别就是dbtable可以填写表名例如user，也可以是一个sql语句作为子查询，例如(select * from user) as tmp，query仅能填写填写sql语句，例如select * from user。如果配置了partitionColumn，那就只能使用dbtable。
4.  `numPartitions, partitionColumn, lowerBound, upperBound`这四个参数，如果设置了，则4个都需要设置，如果只设置了`numPartitions`是无效的。
5.  如果只想指定`numPartitions`，又想分区，怎么办？可以调用`spark.read().jdbc("url", "tablename",predicates,properties)`方法，自己实现一个作为分区的predicates即可。
6.  对于write jdbc而言，原生spark sql的方式还是比较暴力，且不安全，如果不指定truncate，则会删除表再重建，这样会改变原来的表结构。truncate还依赖各个数据库的行为，不一定所有数据库都支持truncate。
7.  个人觉得write jdbc的最佳实践还是通过`DataFrame.foreachPartition()`方法实现，不管写入模式是Append还是Overwrite，都可以自己控制逻辑。