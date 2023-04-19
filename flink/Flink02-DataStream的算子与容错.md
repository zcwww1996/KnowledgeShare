[TOC]
# 1. Flink DataStream的算子
## 1.1 Flink Source

在Flink中，Source主要负责数据的读取

### 1.1.1 基于File的数据源

readTextFile，使用TextInputFormat方式诉求文本文件，并将结果作为String返回。


```scala
val env = StreamExecutionEnvironment.getExecutionEnvironment
val inputStream = env.readTextFile(args(0))
inputStream.print()
env.execute("hello-world")
```


### 1.1.2 基于Socket的数据源
从Socket中读取信息，元素可以用分隔符分开。
● **socketTextStream**

```
val inputStream = env.socketTextStream("localhost", 8888)
```

### 1.1.3 基于集合的数据源

从集合中创建一个数据流，集合中所有元素的类型是一致的。

● **fromCollection(seq)**

```
val list = List(1,2,3,4,5,6,7,8,9)
val inputStream = env.fromCollection(list)
```


● **fromCollection(Iterator)**

```
val iterator = Iterator(1,2,3,4)
val inputStream = env.fromCollection(iterator)
```


● **fromElements(elements:\_\*)**

从一个给定的对象序列中创建一个数据流，所有的对象必须是相同类型的。

```
val lst1 = List(1,2,3,4,5)
val lst2 = List(6,7,8,9,10)
val inputStream = env.fromElement(lst1, lst2)
```


● **generateSequence(from, to)**
从给定的间隔中并行地产生一个数字序列。

```
val inputStream = env.generateSequence(1,10)
```


## 1.2 Flink Transformation
在Flink中，Transformation主要负责对属于的转换操作，调用Transformation后会生成一个新的DataStream
### 1.2.1 map
### 1.2.2 flatMap

val flatMappedStream = inputStream.flatMap(\_.split(" "))
### 1.2.3 filter

```
val nums = env.generateSequence(1,10)
val filtered = nums.filter(\_ % 2 \== 0)

```

### 1.2.4 connect
DataStream,DataStream 转换成 ConnectedStreams：连接两个保持他们类型的数据流，两个数据流被Connect之后，只是被放在了一个同一个流中，内部依然保持各自的数据和形式不发生任何变化，两个流相互独立。

```
val stream1 = env.fromCollection(List("a","b","c","d"))
val stream2 = env.fromCollection(List(1,2,3,4,5,6))

//将两个流连接
val streamConnect: ConnectedStreams[String, Int] = stream1.connect(stream2)

//第一个函数对stream1进行处理
//第一个函数对stream2进行处理
val stream3: DataStream[Any] = streamConnect.map(w => w.toUpperCase , x => x * 10)
stream3.print()
```

### 1.2.5 split
DataStream 转换成 SplitStream：根据某些特征把一个DataStream拆分成两个或者多个DataStream。

```
val split = someDataStream.split(
(num: Int) =>
(num % 2) match {
case 0 => List("even")
case 1 => List("odd")
})
```


### 1.2.6 select
SplitStream 转换成 DataStream：从一个SplitStream中获取一个或者多个DataStream。
### 1.2.7 union
DataStream 转换成 DataStream：对两个或者两个以上的DataStream进行union操作，产生一个包含所有DataStream元素的新DataStream。
### 1.2.8 keyBy
DataStream 转换成 KeyedStream：输入必须是Tuple类型，逻辑地将一个流拆分成不相交的分区，每个分区包含具有相同key的元素，在内部以hash的形式实现的。
### 1.2.9 reduce
KeyedStream 转换成 DataStream：一个分组数据流的聚合操作，合并当前的元素和上次聚合的结果，产生一个新的值，返回的流中包含每一次聚合的结果，而不是只返回最后一次聚合的最终结果。
### 1.2.10 aggregations

KeyedStream转换成DataStream：分组数据流上的滚动聚合操作
- keyedStream.sum(0)
- keyedStream.sum("key")
- keyedStream.min(0)
- keyedStream.min("key")
- keyedStream.max(0)
- keyedStream.max("key")



## 1.3 Flink Sink
在Flink中，Sink负责最终数据的输出
### 1.3.1 print
打印每个元素的toString()方法的值到标准输出或者标准错误输出流中。或者也可以在输出流中添加一个前缀，这个可以帮助区分不同的打印调用，如果并行度大于1，那么输出也会有一个标识由哪个任务产生的标志。
### 1.3.2 writeAsText
将元素以字符串形式逐行写入（TextOutputFormat），这些字符串通过调用每个元素的toString()方法来获取。
### 1.3.3 writeAsCsv
将元组以逗号分隔写入文件中（CsvOutputFormat），行及字段之间的分隔是可配置的。每个字段的值来自对象的toString()方法。
### 1.3.4 writeUsingOutputFormat
自定义文件输出的方法和基类（FileOutputFormat），支持自定义对象到字节的转换。
### 1.3.5 writeToSocket
根据SerializationSchema 将元素写入到socket中。

# 2 Flink的容错
## 2.1 State状态
Flink实时计算程序为了保证计算过程中，出现异常可以容错，就要将中间的计算结果数据存储起来，这些中间数据就叫做State。

State可以是多种类型的，默认是保存在JobManager的内存中，也可以保存到TaskManager本地文件系统或HDFS这样的分布式文件系统
## 2.2 StateBackEnd
用来保存State的存储后端就叫做StateBackEnd，默认是保存在JobManager的内存中，也可以保存的本地文件系统或HDFS这样的分布式文件系统
## 2.3 CheckPointing
Flink实时计算为了容错，可以将中间数据定期保存到起来，这种定期触发保存中间结果的机制叫CheckPointing。CheckPointing是周期执行的。具体的过程是JobManager定期的向TaskManager中的SubTask发送RPC消息，SubTask将其计算的State保存到StateBackEnd中，并且向JobManager相应Checkpoint是否成功。如果程序出现异常或重启，TaskManager中的SubTask可以从上一次成功的CheckPointing的State恢复。


![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440279236-2622fd43-23af-4c84-8842-f54c33e53764.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0)


## 2.4 重启策略
Flink实时计算程序，为了容错，需要开启CheckPointing，一旦开启CheckPointing，如果没有重启策略，默认的重启策略是无限重启，可以也可以设置其他重启策略，如：重启固定次数且可以延迟执行的策略。

## 2.5 CheckPointingMode
- exactly-once 精确一次性语义，可以保证数据消费且消费一次，但是要结合对应的数据源，比如Kafka支持exactly-once
- at-least-once 至少消费一次，可能会重复消费，但是效率要比exactly-once高


# 3. Flink执行计划图

## 3.1 执行图
Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

● StreamGraph：是根据用户通过 Stream API 编写的代码生成的最初的图。用来表示程序的拓扑结构。

● JobGraph：StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。

● ExecutionGraph：JobManager 根据 JobGraph 生成ExecutionGraph。方便调度和监控和跟踪各个 tasks 的状态。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。

● 物理执行图：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
## 3.2 图演变过程
2个并发度（Source为1个并发度）的 SocketTextStreamWordCount 四层执行图的演变过程：

env.socketTextStream().flatMap(...).keyBy(0).sum(1).print();


![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440279864-4635111d-caef-449f-ad22-a496a5fa102d.png?x-oss-process=image%2Fresize%2Cw_1500%2Climit_0)

- **StreamGraph** 根据用户通过 Stream API 编写的代码生成的最初的图。
- **StreamNode** 用来代表 operator 的类，并具有所有相关的属性，如并发度、入边和出边等。
- **StreamEdge** 表示连接两个StreamNode的边。

- **JobGraph** StreamGraph经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。
- **JobVertex** 经过优化后符合条件的多个StreamNode可能会chain在一起生成一个JobVertex，即一个JobVertex包含一个或多个operator，JobVertex的输入是JobEdge，输出是IntermediateDataSet。
- **IntermediateDataSet** 表示JobVertex的输出，即经过operator处理产生的数据集。producer是JobVertex，consumer是JobEdge。

- **JobEdge** 代表了job graph中的一条数据传输通道。source 是 IntermediateDataSet，target 是 JobVertex。即数据通过JobEdge由IntermediateDataSet传递给目标JobVertex。

- **ExecutionGraph** JobManager 根据 JobGraph 生成ExecutionGraph。ExecutionGraph是JobGraph的并行化版本，是调度层最核心的数据结构。
- **ExecutionJobVertex** 和JobGraph中的JobVertex一一对应。每一个ExecutionJobVertex都有和并发度一样多的 ExecutionVertex。

- **ExecutionVertex** 表示ExecutionJobVertex的其中一个并发子任务，输入是ExecutionEdge，输出是IntermediateResultPartition。

- **IntermediateResult** 和JobGraph中的IntermediateDataSet一一对应。一个IntermediateResult包含多个IntermediateResultPartition，其个数等于该operator的并发度。

- **IntermediateResultPartition** 表示ExecutionVertex的一个输出分区，producer是ExecutionVertex，consumer是若干个ExecutionEdge。

- **ExecutionEdge** 表示ExecutionVertex的输入，source是IntermediateResultPartition，target是 ExecutionVertex。source和target都只能是一个。

- **Execution** 是执行一个 ExecutionVertex 的一次尝试。当发生故障或者数据需要重算的情况下 ExecutionVertex 可能会有多个 ExecutionAttemptID。一个 Execution 通过 ExecutionAttemptID 来唯一标识。JM和TM之间关于 task 的部署和 task status 的更新都是通过 ExecutionAttemptID 来确定消息接受者。

- **物理执行图** JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构。
- **Task** Execution被调度后在分配的 TaskManager 中启动对应的 Task。Task 包裹了具有用户执行逻辑的 operator。

- **ResultPartition** 代表由一个Task的生成的数据，和ExecutionGraph中的IntermediateResultPartition一一对应。

- **ResultSubpartition** 是ResultPartition的一个子分区。每个ResultPartition包含多个ResultSubpartition，其数目要由下游消费 Task 数和 DistributionPattern 来决定。

- **InputGate** 代表Task的输入封装，和JobGraph中JobEdge一一对应。每个InputGate消费了一个或多个的ResultPartition。

- **InputChannel** 每个InputGate会包含一个以上的InputChannel，和ExecutionGraph中的ExecutionEdge一一对应，也和ResultSubpartition一对一地相连，即一个InputChannel接收一个ResultSubpartition的输出。

# 4. Flink 数据交换和Redistribute详解

[https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks](https://cwiki.apache.org/confluence/display/FLINK/Data+exchange+between+tasks)


在Flink中，是TaskManager而不是task在网络上交换数据。比如，处于同一个TM内的task，他们之间的数据交换是在一个网络连接（TaskManager创建并维护）上基于多路复用的。


![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440280395-5745122a-5f22-48fb-920e-22d3a39addc5.png?x-oss-process=image%2Fresize%2Cw_1401%2Climit_0)

ExecutionGraph: 执行图是一个包含job计算的逻辑的数据结构。它包含节点（ExecutionVertex，表示计算任务），以及中间结果（IntermediateResultPartition，表示任务产生的数据）。节点通过ExecutionEdge（EE）来连接到它们要消费的中间结果：

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440280888-a476595c-7604-4754-9020-beeea2f7ae02.png?x-oss-process=image%2Fresize%2Cw_1331%2Climit_0)

这些都是存在与JobManager中的逻辑数据结构（描写信息）。它们在TaskManager中存在运行时等价的数据结构，用来应对最终的数据处理。在运行时，IntermediateResultPartition的等价数据结构被称为ResultPartition。

ResultPartition（RP）表示BufferWriter写入的data chunk。一个RP是ResultSubpartition（RS）的集合。这是为了区别被不同接收者定义的数据，例如针对一个reduce或一个join的分区shuffle的场景。

ResultSubpartition（RS）表示一个operator创建的数据的一个分区，跟要传输的数据逻辑一起传输给接收operator。RS的特定的实现决定了最终的数据传输逻辑，它被设计为插件化的机制来满足系统各种各样的数据传输需求。例如，PipelinedSubpartition就是一种支持流数据交换的pipeline的实现。而SpillableSubpartition是一个支持批处理的块数据实现。

InputGate: 在接收端，逻辑上等价于RP。它用于处理并收集来自上游的buffer中的数据。
InputChannel: 在接收端，逻辑上等价于RS。用于接收某个特定的分区的数据。

序列化器、反序列化器用于可靠得将类型化的数据转化为纯粹的二进制数据，处理跨buffer的数据。


## 4.1 数据交换的控制流

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440281389-7101c598-c38c-483b-9223-c68ef60eb081.png?x-oss-process=image%2Fresize%2Cw_1402%2Climit_0)




上图表示一个简单的map-reduce job并具有两个并行的task。我们有两个TaskManager，每个TaskManager都有两个task（一个map，一个reduce），这两个TaskManager运行在两个不同的节点上，有一个JobManager运行在第三方节点上。我们聚焦在task M1和R2之间的传输初始化。数据传输使用粗箭头表示，消息使用细箭头表示。首先，M1生产一个ResultPartition（RP1）（箭头1）。当RP对于消费端变得可访问（我们后面会讨论），它会通知JobManager（箭头2）。JobManager通知想要接收这个分区数据的接收者（task R1和R2）分区当前已经准备好了。如果接收者还没有被调度，这将会触发task的deployment（箭头3a,3b）。然后接收者将会向RP请求数据（箭头4a,4b）。这将会初始化任务之间的数据传输（5a,5b）,这个初始化要么是本地的(5a)，或者通过TaskManager的网络栈传输（5b）。这种机制给了RP在决定什么时候通知JobManager自己已经处于准备好状态的时机上拥有充分的自由度。例如，如果RP1希望在通知JM之前，等待数据完整地传输完（比如它将数据写到一个临时文件里），这种数据交换机制粗略来看等同于批处理数据交换，就像在Hadoop中实现的那样。而如果RP1一旦在其第一条记录准备好时就通知JobManager，那么我就拥有了一个流式的数据交换。

## 4.2 字节缓冲区在两个task之间的传输

![](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626440281957-cc74985d-31b6-4e87-8c2f-be141a9541f1.png?x-oss-process=image%2Fresize%2Cw_1402%2Climit_0)


上面这张图展示了一个更细节的过程，描述了数据从生产者传输到消费者的完整生命周期。最初，MapDriver生产数据记录（通过Collector收集），这些记录被传给RecordWriter对象。RecordWriter包含一组序列化器（RecordSerializer对象）。消费者task可能会消费这些数据。一个ChannelSelector选择一个或者多个序列化器来处理记录。如果记录在broadcast中，它们将被传递给每一个序列化器。如果记录是基于hash分区的，ChannelSelector将会计算记录的hash值，然后选择合适的序列化器。

序列化器将数据记录序列化成二进制的表示形式。然后将它们放到大小合适的buffer中（记录也可以被切割到多个buffer中）。这些buffer首先会被传递给BufferWriter，然后被写到一个ResulePartition（RP）中。一个RP包含多个subpartition（ResultSubpartition - RS），用于为特定的消费者收集buffer数据。在上图中的这个buffer是为TaskManager2中的reducer定义的，然后被放到RS2中。既然首个buffer进来了，RS2就对消费者变成可访问的状态了（注意，这个行为实现了一个streaming shuffle），然后它通知JobManager。

JobManager查找RS2的消费者，然后通知TaskManager 2一个数据块已经可以访问了。通知TM2的消息会被发送到InputChannel，该inputchannel被认为是接收这个buffer的，接着通知RS2可以初始化一个网络传输了。然后，RS2通过TM1的网络栈请求该buffer，然后双方基于netty准备进行数据传输。网络连接是在TaskManager（而非特定的task）之间长时间存在的。

一旦buffer被TM2接收，它会穿过一个类似的对象栈，起始于InputChannel（接收端 等价于IRPQ）,进入InputGate（它包含多个IC），最终进入一个RecordDeserializer，它用于从buffer中还原成类型化的记录，然后将其传递给接收task，这个例子中是ReduceDriver。