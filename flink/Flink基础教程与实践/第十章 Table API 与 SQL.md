[TOC]

Table API 是流处理和批处理通用的关系型 API，Table API 可以基于流输入或者批输入来运行而不需要进行任何修改。Table API 是 SQL 语言的超集并专门为 Apache Flink 设计的，Table API 是 Scala 和 Java 语言集成式的 API。与常规 SQL 语言中将查询指定为字符串不同，Table API 查询是以 Java 或 Scala 中的语言嵌入样式来定义的，具有 IDE 支持如:自动完成和语法检测。


# 1. 需要引入的 pom 依赖

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>flink-table_2.11</artifactId>
    <version>1.7.2</version>
</dependency>
```

# 2. 简单了解 TableAPI

```scala
def main(args: Array[String]): Unit = { 
    val env: StreamExecutionEnvironment =
  					StreamExecutionEnvironment.getExecutionEnvironment
	val myKafkaConsumer: FlinkKafkaConsumer011[String] = MyKafkaUtil.getConsumer("ECOMMERCE")
	val dstream: DataStream[String] = env.addSource(myKafkaConsumer)
  val tableEnv: StreamTableEnvironment = TableEnvironment.getTableEnvironment(env)
	val ecommerceLogDstream: DataStream[EcommerceLog] = dstream.map{
				jsonString => JSON.parseObject(jsonString,classOf[EcommerceLog]) 
  } 
  
  val ecommerceLogTable: Table = tableEnv.fromDataStream(ecommerceLogDstream)
	val table: Table = ecommerceLogTable.select("mid,ch").filter("ch='appstore'")
	val midchDataStream: DataStream[(String, String)] =
	table.toAppendStream[(String,String)]
  midchDataStream.print()
  env.execute()
}
```

## 2.1 动态表
如果流中的数据类型是 case class 可以直接根据 case class 的结构生成 table


```scala
tableEnv.fromDataStream(ecommerceLogDstream)
```

或者根据字段顺序单独命名

```scala
tableEnv.fromDataStream(ecommerceLogDstream,’mid,’uid .......)
```

最后的动态表可以转换为流进行输出

```scala
table.toAppendStream[(String,String)]
```

## 2.2 字段
用一个单引放到字段前面来标识字段名, 如 ‘name , ‘mid ,’amount 等


# 3. TableAPI 的窗口聚合操作

## 3.1 通过一个例子了解 TableAPI

```scala
	//每 10 秒中渠道为 appstore 的个数
def main(args: Array[String]): Unit = {
//sparkcontext
	val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment
	//时间特性改为 eventTime env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)
    val myKafkaConsumer: FlinkKafkaConsumer011[String] = MyKafkaUtil.getConsumer("ECOMMERCE")
		val dstream: DataStream[String] = env.addSource(myKafkaConsumer)
		val ecommerceLogDstream: DataStream[EcommerceLog] = 
  					dstream.map{ 
              jsonString =>JSON.parseObject(jsonString,classOf[EcommerceLog]) 
            }
  
	//告知 watermark 和 eventTime 如何􏰁取
	val ecommerceLogWithEventTimeDStream: DataStream[EcommerceLog] = 
  		ecommerceLogDstream.assignTimestampsAndWatermarks(new BoundedOutOfOrdernessTimestampExtractor[EcommerceLog](Time.seconds(0L)) {
				override def extractTimestamp(element: EcommerceLog): Long = {
						element.ts 
        }
			}).setParallelism(1)

  
  val tableEnv: StreamTableEnvironment =
					TableEnvironment.getTableEnvironment(env)
  
  
//把数据流转化成 Table
val ecommerceTable: Table = tableEnv.fromDataStream(ecommerceLogWithEventTimeDStream , 
                                                    'mid,'uid,'appid,'area,'os,'ch,'logType,'vs,
                                                    'logDate,'logHour,'logHourMinut e,'ts.rowtime)

  
  //通过 table api 进行操作
	// 每 10 秒 统计一次各个渠道的个数 table api 解决
	//1 groupby 2 要用 window 3 用 eventtime 来确定开窗时间
	val resultTable: Table = 
  	ecommerceTable.window(Tumble over 10000.millis on 'ts as 'tt).groupBy('ch,'tt ).select( 'ch, 'ch.count)

  //把 Table 转化成数据流
	val resultDstream: DataStream[(Boolean, (String, Long))] = 
  			resultSQLTable.toRetractStream[(String,Long)]
				resultDstream.filter(_._1).print()
  			env.execute()
   
}
```



## 3.2 关于 group by
1. 如果了使用 groupby，table 转换为流的时候只能用 toRetractDstream

```scala
val rDstream: DataStream[(Boolean, (String, Long))] = table 
	.toRetractStream[(String,Long)]
```

2. toRetractDstream 得到的第一个 boolean 型字段标识 true 就是最新的数据
(Insert)，false 表示过期老数据(Delete)

```scala
val rDstream: DataStream[(Boolean, (String, Long))] = table 
	.toRetractStream[(String,Long)]
rDstream.filter(_._1).print()
```

3. 如果使用的 api 包括时间窗口，那么窗口的字段必须出现在 groupBy 中。

```scala
val table: Table = ecommerceLogTable
.filter("ch ='appstore'")
.window(Tumble over 10000.millis on 'ts as 'tt) .groupBy('ch ,'tt)
.select("ch,ch.count ")
```

## 3.3 关于时间窗口
1. 用到时间窗口，必须􏰁前声明时间字段，如果是 processTime 直接在创建动态表时进行追加就可以。


```scala
val ecommerceLogTable: Table = tableEnv 
	.fromDataStream( ecommerceLogWithEtDstream,
	'mid,'uid,'appid,'area,'os,'ch,'logType,'vs,'logDate,'logHour,'lo gHourMinute,'ps.proctime)
```

2. 如果是 EventTime 要在创建动态表时声明

```scala
val ecommerceLogTable: Table = tableEnv 
	.fromDataStream(ecommerceLogWithEtDstream,
	'mid,'uid,'appid,'area,'os,'ch,'logType,'vs,'logDate,'logHour,'lo gHourMinute,'ts.rowtime)
```

3. 滚动窗口可以使用 Tumble over 10000.millis on 来表示

```scala
val table: Table = ecommerceLogTable.filter("ch ='appstore'") 
    	.window(Tumble over 10000.millis on 'ts as 'tt) 
    	.groupBy('ch ,'tt)
		.select("ch,ch.count ")
```

# 4. SQL 如何编写

```scala
def main(args: Array[String]): Unit = {

  //sparkcontext
	val env: StreamExecutionEnvironment = StreamExecutionEnvironment.getExecutionEnvironment

  //时间特性改为 eventTime env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

  val myKafkaConsumer: FlinkKafkaConsumer011[String] = MyKafkaUtil.getConsumer("ECOMMERCE")

  val dstream: DataStream[String] = env.addSource(myKafkaConsumer)

  val ecommerceLogDstream: DataStream[EcommerceLog] = dstream.map{ jsonString =>JSON.parseObject(jsonString,classOf[EcommerceLog]) }

  //告知 watermark 和 eventTime 如何􏰁取

  val ecommerceLogWithEventTimeDStream: DataStream[EcommerceLog] = 
  			ecommerceLogDstream.assignTimestampsAndWatermarks(
          new BoundedOutOfOrdernessTimestampExtractor[EcommerceLog](Time.seconds(0L)) {
    override def extractTimestamp(element: EcommerceLog): Long = { element.ts
    }
  }).setParallelism(1)
  //SparkSession
    val tableEnv: StreamTableEnvironment = TableEnvironment.getTableEnvironment(env)
	
  	//把数据流转化成 Table
		
  val ecommerceTable: Table = 
  tableEnv.fromDataStream(ecommerceLogWithEventTimeDStream ,
                          'mid,'uid,'appid,'area,'os,'ch,'logType,'vs,
                          'logDate,'logHour,'logHourMinu te,'ts.rowtime)
  
	
  //通过 table api 进行操作
	
  // 每 10 秒 统计一次各个渠道的个数 table api 解决

  //1 groupby 2 要用 window 3 用 eventtime 来确定开窗时间

  val resultTable: Table = ecommerceTable.window(Tumble over 10000.millis on 'ts as 'tt).groupBy('ch,'tt ).select( 'ch, 'ch.count)

  // 通过 sql 进行操作

  val resultSQLTable : Table = tableEnv.sqlQuery( "select ch ,count(ch) from"
                                                 +ecommerceTable
                                                 +" group by ch ,Tumble(ts,interval '10' SECOND )")

  //把 Table 转化成数据流

  //val appstoreDStream: DataStream[(String, String, Long)] = 
  appstoreTable.toAppendStream[(String,String,Long)]
  

  val resultDstream: DataStream[(Boolean, (String, Long))] = 
  resultSQLTable.toRetractStream[(String,Long)]
  resultDstream.filter(_._1).print()
  env.execute()
}
```
