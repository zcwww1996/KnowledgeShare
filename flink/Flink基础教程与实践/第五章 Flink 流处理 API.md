[TOC]

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581778371889-0921ca8a-eab1-4786-a4c0-4226c0ac615d.png)

# 1. Environment
## 1.1 getExecutionEnvironment
创建一个执行环境，表示当前执行程序的上下文。 如果程序是独立调用的，则此方法返回本地执行环境;如果从命令行客户端调用程序以提交到集群，则此方法返回此集群的执行环境，也就是说，getExecutionEnvironment 会根据查询运行的方式决定返回什么样的运行环境，是最常用的一种创建执行环境的方式。


```scala
val env: ExecutionEnvironment = ExecutionEnvironment.getExecutionEnvironment
```



如果没有设置并行度，会以 flink-conf.yaml 中的配置为准，默认是 1

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581778606591-72d52667-f3f3-4096-889b-55bfb58caac8.png)


## 1.2 createLocalEnvironment
返回本地执行环境，需要在调用时指定默认的并行度。

```scala
val env = StreamExecutionEnvironment.createLocalEnvironment(1)
```

## 1.3 createRemoteEnvironment
返回集群执行环境，将 Jar 提交到远程服务器。需要在调用时指定 JobManager
的 IP 和端口号，并指定要在集群中运行的 Jar 包。

```scala
val env = ExecutionEnvironment.createRemoteEnvironment("jobmanage-hostname", 6123,"YOURPATH//wordcount.jar")
```

# 2. Source

## 2.1 从集合读取数据
```scala
// 定义样例类，传感器 id，时间戳，温度
case class SensorReading(id: String, timestamp: Long, temperature: Double)
object Sensor {
def main(args: Array[String]): Unit = {
	val env = StreamExecutionEnvironment.getExecutionEnvironment
	val stream1 = env.fromCollection(List(
		SensorReading("sensor_1", 1547718199, 35.80018327300259), SensorReading("sensor_6", 1547718201, 15.402984393403084), SensorReading("sensor_7", 1547718202, 6.720945201171228), SensorReading("sensor_10", 1547718205, 38.101067604893444)
))
stream1.print("stream1:").setParallelism(1)
    env.execute()
  }
}
```


## 2.2 从文件读取数据
```scala
val stream2 = env.readTextFile("YOUR_FILE_PATH")
```

## 2.3 以 kafka 消息队列的数据作为来源
需要引入 kafka 连接器的依赖:

pom.xml

```xml
<!-- https://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.11 -->
<dependency>
	<groupId>org.apache.flink</groupId>
    <artifactId>flink-connector-kafka-0.11_2.11</artifactId> 
    <version>1.7.2</version>
</dependency>
```

具体代码如下:

```scala
val properties = new Properties()
properties.setProperty("bootstrap.servers", "localhost:9092") 
properties.setProperty("group.id", "consumer-group")
properties.setProperty("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer") 
properties.setProperty("value.deserializer","org.apache.kafka.common.serialization.StringDeserializer")
properties.setProperty("auto.offset.reset", "latest")

val stream3 = env.addSource(new FlinkKafkaConsumer011[String]("sensor",
                                          new SimpleStringSchema(), properties))
```


## 2.4 自定义 Source
除了以上的 source 数据来源，我们还可以自定义 source。需要做的，只是传入
一个 SourceFunction 就可以。具体调用如下:
```scala
val stream4 = env.addSource( new MySensorSource() )
```

我们希望可以随机生成传感器数据，MySensorSource 具体的代码实现如下

```scala
class MySensorSource extends SourceFunction[SensorReading]{
    
// flag: 表示数据源是否还在正常运行 var running: Boolean = true
override def cancel(): Unit = { running = false
}
    
override def run(ctx: SourceFunction.SourceContext[SensorReading]): Unit={
    // 初始化一个随机数发生器 val rand = new Random()
    var curTemp = 1.to(10).map(
    	i => ( "sensor_" + i, 65 + rand.nextGaussian() * 20 )
    )
        
    while(running){ // 更新温度值
       curTemp = curTemp.map(
           t => (t._1, t._2 + rand.nextGaussian() )
        )
        // 获取当前时间戳
        val curTime = System.currentTimeMillis()
        curTemp.foreach(
        t => ctx.collect(SensorReading(t._1, curTime, t._2))
        )

        Thread.sleep(100) 
    	}
	}
}
```

# 3. Transform

## 3.1 map
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581781944902-6490a2d9-bb45-4775-a487-218d0f97d7b0.png)

```scala
val streamMap = stream.map { x => x * 2 }
```

## 3.2 flatMap

```scala
flatMap 的函数签名:
def flatMap[A,B](as: List[A])(f: A ⇒ List[B]): List[B]
例如: flatMap(List(1,2,3))(i ⇒ List(i,i))
结果是 List(1,1,2,2,3,3),
而 
List("a b", "c d").flatMap(line ⇒ line.split(" "))

结果是 List(a, b, c, d)。
```

```scala
 val streamFlatMap = stream.flatMap{
      x => x.split(" ")
}
```


## 3.3 Filter


```scala
val streamFilter = stream.filter{
    x => x == 1
}
```

## 3.4 KeyBy
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581782088722-27a274c3-f36b-4f19-8e6f-0573ca5909e1.png)

**DataStream**→ **KeyedStream**：逻辑地将一个流拆分成不相交的分区，每个分
区包含具有相同 key 的元素，在内部以 hash 的形式实现的。


## 3.5 滚动聚合算子(Rolling Aggregation)


```scala
这些算子可以针对 KeyedStream 的每一个支流做聚合。
- sum()
- min()
- max()
- minBy() 
- maxBy()
```

## 3.6 Reduce
**KeyedStream**→ **DataStream**：一个分组数据流的聚合操作，合并当前的元素
和上次聚合的结果，产生一个新的值，返回的流中包含每一次聚合的结果，而不是
只返回最后一次聚合的最终结果。

```scala
val stream2 = env.readTextFile("YOUR_PATH\\sensor.txt")
.map( data => {
	val dataArray = data.split(",") 
    SensorReading(dataArray(0).trim, dataArray(1).trim.toLong,
					dataArray(2).trim.toDouble) 
})
.keyBy("id")
.reduce( (x, y) => SensorReading(x.id, x.timestamp + 1, y.temperature) )
```

## 3.7 Split 和 Select
Split

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581862749315-a5cab789-c7ee-400c-86b6-b6b3f0484496.png)

select

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581862774541-77be665d-3797-4dc4-8e3d-79545022ffe6.png)



```scala
val splitStream = stream2.split( sensorData => {
	if (sensorData.temperature > 30) Seq("high") else Seq("low")
})
val high = splitStream.select("high")
val low = splitStream.select("low")
val all = splitStream.select("high", "low")
```

## 3.8 Connect 和 CoMap
connect

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581862897654-09c3fb8f-c540-45e8-8047-8eb92245e5e9.png)

coflatmap

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581862930044-590fa009-6f87-4ac3-8a26-c3e555aa45f3.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581862958397-bd90527b-54e6-47a6-8b89-a5081f1365d8.png)

```scala
val warning = high.map( sensorData => (sensorData.id,
sensorData.temperature) )
val connected = warning.connect(low)
val coMap = connected.map(
	warningData => (warningData._1, warningData._2, "warning"), 
  lowData => (lowData.id, "healthy")
)
```


## 3.9 Union
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581863032010-fa605257-65fb-485e-b392-d9b381424929.png)



```scala
//合并以后打印
val unionStream: DataStream[StartUpLog] = appStoreStream.union(otherStream) 
unionStream.print("union:::")
```

**Connect 与 Union 区别:**

```scala
1. Union 之前两个流的类型必须是一样，Connect 可以不一样，在之后的 coMap 中再去调整成为一样的。
2. Connect 只能操作两个流，Union 可以操作多个。
```

# 4. 支持的数据类型
Flink 流应用程序处理的是以数据对象表示的事件流。所以在 Flink 内部，我们需要能够处理这些对象。它们需要被序列化和反序列化，以便通过网络传送它们;或者从状态后端、检查点和保存点读取它们。

为了有效地做到这一点，Flink 需要明
确知道应用程序所处理的数据类型。Flink 使用类型信息的概念来表示数据类型，并为每个数据类型生成特定的序列化器、反序列化器和比较器。


Flink 还具有一个类型提取系统，该系统分析函数的输入和返回类型，以自动获取类型信息，从而获得序列化器和反序列化器。但是，在􏰂些情况下，例如 lamb函数或泛型类型，需要显式地提供类型信息，才能使应用程序正常工作或提高其性能。


Flink 支持 Java 和 Scala 中所有常见数据类型。使用最广泛的类型有以下几种。

## 4.1 基础数据类型
Flink 支持所有的 Java 和 Scala 基础数据类型，Int, Double, Long, String, ...

```scala
val numbers: DataStream[Long] = env.fromElements(1L, 2L, 3L, 4L) 
numbers.map( n => n + 1 )
```


## 4.2 Java 和 Scala 元组(Tuples)


```scala
val persons: DataStream[(String, Integer)] = env.fromElements( ("Adam", 17),
                                                              ("Sarah", 23) ) 
persons.filter(p => p._2 > 18)
```

## 4.3 Scala 样例类(case classes)


```scala
case class Person(name: String, age: Int)
val persons: DataStream[Person] = env.fromElements( Person("Adam", 17),
Person("Sarah", 23) )
persons.filter(p => p.age > 18)
```

## 4.4 Java 简单对象(POJOs)


```scala
public class Person {
public String name;
public int age;
public Person() {}
public Person(String name, int age) {
    this.name = name;
    this.age = age;
  }
}
DataStream<Person> persons = env.fromElements(
	new Person("Alex", 42),
  new Person("Wendy", 23));
```

## 4.5 其它(Arrays, Lists, Maps, Enums, 等等)
Flink 对 Java 和 Scala 中的一些特殊目的的类型也都是支持的，比如 Java的ArrayList，HashMap，Enum 等等。


# 5. 实现 UDF 函数——更细粒度的控制流

## 5.1 函数类(Function Classes)
Flink 暴露了所有 udf 函数的接口(实现方式为接口或者抽象类)。例如
MapFunction, FilterFunction, ProcessFunction 等等。

下面例子实现了 FilterFunction 接口:
										
```scala
class FilterFilter extends FilterFunction[String] {
  override def filter(value: String): Boolean = { value.contains("flink")
} }
val flinkTweets = tweets.filter(new FlinkFilter)
```

还可以将函数实现成匿名类

```scala
val flinkTweets = tweets.filter(
new RichFilterFunction[String] {
override def filter(value: String): Boolean = {
           value.contains("flink")
       }
} )
```

我们 filter 的字符串"flink"还可以当作参数传进去。


```scala
val tweets: DataStream[String] = ...
val flinkTweets = tweets.filter(new KeywordFilter("flink"))
class KeywordFilter(keyWord: String) extends FilterFunction[String] { override def filter(value: String): Boolean = {
       value.contains(keyWord)
    }
}
```

## 5.2 匿名函数(Lambda Functions)


```scala
val tweets: DataStream[String] = ...
val flinkTweets = tweets.filter(_.contains("flink"))
```

## 5.3 富函数(Rich Functions)
“富函数”是 DataStream API 提供的一个函数类的接口，所有 Flink 函数类都有其 Rich 版本。它与常规函数的不同在于，可以获取运行环境的上下文，并拥有一些生命周期方法，所以可以实现更复杂的功能。

- RichMapFunction
- RichFlatMapFunction
- RichFilterFunction
- ...




Rich Function 有一个生命周期的概念。典型的生命周期方法有:

- open()方法是 rich function 的初始化方法，当一个算子例如 map 或者 filter，被调用之前 open()会被调用。
- close()方法是生命周期中的最后一个调用的方法，做一些清理工作。
- getRuntimeContext()方法提供了函数的 RuntimeContext 的一些信息，例如函数执行的并行度，任务的名字，以及 state 状态


```scala
class MyFlatMap extends RichFlatMapFunction[Int, (Int, Int)] {
  var subTaskIndex = 0
	override def open(configuration: Configuration): Unit = {
    subTaskIndex = getRuntimeContext.getIndexOfThisSubtask // 以下可以做一些初始化工作，例如建立一个和 HDFS 的连接
}
override def flatMap(in: Int, out: Collector[(Int, Int)]): Unit = {
  if (in % 2 == subTaskIndex) {
       out.collect((subTaskIndex, in))
    }
}
override def close(): Unit = {
  // 以下做一些清理工作，例如断开和 HDFS 的连接。 }
}
```
# 6. Sink
Flink 没有类似于 spark 中 foreach 方法，让用户进行迭代的操作。虽有对外的输出操作都要利用 Sink 完成。最后通过类似如下方式完成整个任务最终输出操作。

```scala
stream.addSink(new MySink(xxxx))
```

官方提供了一部分的框架的 sink。除此以外，需要用户自定义实现 sink。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581863778019-f94527bb-0175-4d8d-98ad-d5d8ea07f225.png)


## 6.1 Kafka

pom.xml

```xml
<!-- https://mvnrepository.com/artifact/org.apache.flink/flink-connector-kafka-0.11 -->
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-kafka-0.11_2.11</artifactId>
	<version>1.7.2</version> 
</dependency>
```

主函数中添加 sink:

```scala
val union = high.union(low).map(_.temperature.toString)
union.addSink(new FlinkKafkaProducer011[String]("localhost:9092", 
                                               "test", new SimpleStringSchema()))
```

## 6.2 Redis

pom.xml

```xml
<!-- https://mvnrepository.com/artifact/org.apache.bahir/flink-connector-redis -->
<dependency>
  <groupId>org.apache.bahir</groupId>
  <artifactId>flink-connector-redis_2.11</artifactId>
  <version>1.0</version>
</dependency>
```

定义一个 redis 的 mapper 类，用于定义保存到 redis 时调用的命令:

```scala
class MyRedisMapper extends RedisMapper[SensorReading]{
override def getCommandDescription: RedisCommandDescription = {
new RedisCommandDescription(RedisCommand.HSET, "sensor_temperature") }
override def getValueFromData(t: SensorReading): String = t.temperature.toString
override def getKeyFromData(t: SensorReading): String = t.id }
```

在主函数中调用:

```scala
val conf = new 
FlinkJedisPoolConfig.Builder().setHost("localhost").setPort(6379).build() 
dataStream.addSink( new RedisSink[SensorReading](conf, new MyRedisMapper) )
```

## 6.3 Elasticsearch

pom.xml

```xml
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-elasticsearch6_2.11</artifactId>
  <version>1.7.2</version> 
</dependency>
```

在主函数中调用:

```scala
val httpHosts = new util.ArrayList[HttpHost]()
httpHosts.add(new HttpHost("localhost", 9200))
val esSinkBuilder = new ElasticsearchSink.Builder[SensorReading](
 		 	httpHosts, 
      new ElasticsearchSinkFunction[SensorReading] {
	
     override def process(t: SensorReading, runtimeContext: RuntimeContext, requestIndexer: RequestIndexer): Unit = {
     println("saving data: " + t)

       val json = new util.HashMap[String, String]() json.put("data", t.toString)
       val indexRequest =
			Requests.indexRequest().index("sensor").`type`("readingData").source(json) requestIndexer.add(indexRequest)
			println("saved successfully")
} })
dataStream.addSink( esSinkBuilder.build()）
```

## 6.4 JDBC 自定义 sink										
```xml
<!-- https://mvnrepository.com/artifact/mysql/mysql-connector-java --> 
<dependency>
  <groupId>mysql</groupId> 
  <artifactId>mysql-connector-java</artifactId>
  <version>5.1.44</version>
</dependency>
```

添加 MyJdbcSink

```scala
class MyJdbcSink() extends RichSinkFunction[SensorReading]{
  var conn: Connection = _
  var insertStmt: PreparedStatement = _
  var updateStmt: PreparedStatement = _
  
  // open 主要是创建连接
  override def open(parameters: Configuration): Unit = {
  	super.open(parameters)
    conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/test", "root", "123456")
    insertStmt = conn.prepareStatement("INSERT INTO temperatures (sensor, temp) VALUES (?, ?)")
    updateStmt = conn.prepareStatement("UPDATE temperatures SET temp = ? WHERE
    sensor = ?") }
  
		// 调用连接，执行 sql
    override def invoke(value: SensorReading, context: SinkFunction.Context[_]): Unit = {
    updateStmt.setDouble(1, value.temperature)	
      updateStmt.setString(2, value.id) updateStmt.execute()
    if (updateStmt.getUpdateCount == 0) { insertStmt.setString(1, value.id) insertStmt.setDouble(2, value.temperature) insertStmt.execute()
    } }
  
		override def close(): Unit = { insertStmt.close() updateStmt.close() conn.close()
} }
```

在 main 方法中增加，把明细保存到 mysql 中

```scala
dataStream.addSink(new MyJdbcSink())
```