[TOC]
# 1．Flink简介

## 1.1什么是Flink

 Apache Flink 是一个分布式大数据处理弓擎，可对有限数据流和无限数据流进行有状态计算。可部署在各种集群环境，对各种大小的数据规模进行快速计算。

[![flink_home](https://z3.ax1x.com/2021/11/05/InN4v8.png)](https://pic.downk.cc/item/5f28ffe614195aa5944ee72d.png)

## 1.2 Flink的历史

早在2008年，Flink的前身已经是柏林理工大学一个研究性项目，在2014被Apache孵化器所接受，然后迅速地成为了ASF（Apache Software Foundation）的顶级项目之一品易教育

Flink的商业公司Data Artisans，位于柏林的，公司成立于2014年，共获得两轮融资共计650万欧。

该公司旨在为企业提供大规模数据处理解决方案，使企业可以管理和部署实时数据，实时反馈数据，做更快、更精准的商业决策。目前，ING，Netflix和Uber等企业都通过Data Artisans的Apache Flink平台部署大规模分布式应用，如实时数据分析、机器学习、搜索、排序推荐和欺诈风险等。

2019年1月8日，阿里巴巴以9000r万欧元收购该公司

## 1.3 Flink的特点

- 批流统一
- 支持高吞吐、低延迟、高性能的流处
- 支持带有事件时间的窗口（Window）操作
- 支持有状态计算的Exactly-once语义
- 支持高度灵活的窗口（Window）操作，支持基于time、count、session窗口操作支持具有Backpressure功能的持续流模型
- 支持基于轻量级分布式快照（Snapshot）实现的容错支持选代计算
- Flink在JVM内部实现了自己的内存管理
- 支持程序自动优化：避免特定情况下Shuffle、排序等昂贵操作，中间结果有必要进行缓存

### 1.3.1 Flink与其他框架的对比


| 框架          | 优点                                                | 缺点                                          |
| ------------- | --------------------------------------------------- | :-------------------------------------------- |
| Storm         | 低延迟                                              | 吞吐量低、不能保证exactly-once、编程API不丰富 |
| Spak Stcaning | 吞吐量高、可以保证exactly-once、编程API丰富          | 延迟较高                                      |
| Flink         | 低延迟、吞吐量高、可以保证exactly-once、编程API丰富 | 快速迭代中，API变化比较快                     |

[![包含](https://pic.imgdb.cn/item/6184a9f92ab3f51d91daa0d2.jpg)](https://shop.io.mi-img.com/app/shop/img?id=shop_84b6e741cba5753be3f21d6e949859d7.jpeg)

Spark就是为离线计算而设计的，在Spark生态体系中，不论是流处理和批处理都是底层引擎都是Spark Core，**Spark Streaming将微批次小任务不停的提交到Spark引擎，从而实现准实时计算，SparkStreaming只不过是一种特殊的批处理而已。**

[![Spark全局组成](https://www.helloimg.com/images/2021/11/05/CYu3Nq.jpg)](https://pic.downk.cc/item/5f2909a214195aa594540bb9.jpg)

Spark  Spark MLlib GraphX SQL ||Streaming|（machine（graph）
learning）
Apache Spark Flink 就是为实时计算而设计的，Flink可以同时实现批处理和流处理，**Flink 将批处理（即有有界数据）视作一种特殊的流处理。**

[![Flink全局组成结构图](https://z3.ax1x.com/2021/11/05/InUeKO.jpg)](https://pic.downk.cc/item/5f2909a214195aa594540bbc.jpg)

# 2. Flink架构体系简介

参考：https://blog.csdn.net/weixin_33828101/article/details/89694651<br>
参考:https://ci.apache.org/projects/flink/flink-docs-release-1.11/concepts/flink-architecture.html

[![Flink架构体系](https://shop.io.mi-img.com/app/shop/img?id=shop_5ff3fde7ff0f471a394dcca8cb423617.jpeg)](https://pic.downk.cc/item/5f290e7414195aa5945608cf.png)

- **JobManager**：<br>
也称之为Master，用于协调分布式执行，它用来调度task，协调检查点，协调失败时恢复等。Flink运行时至少存在一个master，如果配置高可用模式则会存在多个master，它们其中有一个是leader，而其他的都是standby。

- **TaskManager**：<br>
也称之为Worker，用于执行一个dataflow的task、数据缓冲和Data Streams的数据交换，Flink运行时至少会存在一个TaskManager.JobManager 和TaskManager可以直接运行在物理机上，或者运行YARN这样的资源调度框架，TaskManager 通过网络连接到JobManager，通过RPC通信告知自身的可用性进而获得任务分配。

- **Task(任务)**:<br>
Task 是一个阶段多个功能相同 subTask 的集合，类似于 Spark 中的 TaskSet。

- **subTask(子任务)**：<br>
subTask 是 Flink 中任务最小执行单元，是一个 Java 类的实例，这个 Java 类中有属性和方法，完成具体的计算逻辑。

- **Operator Chains(算子链)**：<br>
没有 shuffle 的多个算子合并在一个 subTask 中，就形成了 Operator Chains，类似于 Spark 中的 Pipeline。

- **Slot(插槽)**：<br>
Flink 中计算资源进行隔离的单元，一个 Slot 中可以运行多个 subTask，但是这些 subTask 必须是来自同一个 application 的不同阶段的 subTask。


==**如何划分Task**==：

1. 并行度发生变化时;
2. keyBy() /window()/apply() 等发生 Rebalance 重新分配;
3. 调用 startNewChain() 方法，开启一个新的算子链；
4. 调用 diableChaining()方法，即：告诉当前算子操作不使用 算子链 操作。将其单独划分，形成一个task，和其他算子不再有Operator Chains


SparkStreaming和Flink的角色对比

| Spark Streaming | Flink          |
| :-------------- | :------------- |
| DStream         | DataStream     |
| Trasnformation  | Trasnformation |
| Action          | sink           |
| TaskSub         | Task           |
| Pipeline        | Opratorchains  |
| DAGDataFlow     | Graph          |
| Master+Driver   | JobManager     |
| Worker+Executor | TaskManager    |

# 3. Flink环境搭建
## 3.1 架构说明(standalone模式)
standalone模式是Flink自带的分布式集群模式，不依赖其他的资源调度框架

```
graph TD
  A((node-1)) -.-> B((node-2))
  A((node-1)) -.-> C((node-3))
  A((node-1)) -.-> D((node-4))

  F[standalone模式]
```

## 3.2 搭建步骤

- ①下载fink安装包，下载地址：https:/flink.apache.org/downloads.html<br>
- ②上传flink 安装包到Linux服务器上<br>
- ③解压flink 安装包<br>
 `tar -xvf flink-1.9.1-bin-scala_2.11.tgz -C /bigdata/`<br>
- ④修改conf目录下的flink-conf.yaml配置文件<br>

```bash
#指定jobmanager的地址
jobmanager.rpc.address:node-1
#指定taskmanager的可用槽位的数量
taskmanager.numberOfTaskSlots：2
```

- ⑤修改conf目录下的slaves配置文件，指定taskmanager的所在节点

```bash
node-2
node-3
```

- ⑥将配置好的Flink拷贝到其他节点

```bash
for i in{2..3};do scp-r flink-1.9.1/node-Si:$PWD;done
```

## 3.3 启动flink集群和检测
1) 执布启动脚本

```bash
bin/start-cluster.sh
```

2) 执行jps命令查看Java进程

```bash
在ndoe-1上可用看见StandaloneSessionClusterEntrypoint进程即JobManager，

在其他的节点上可用看见到TaskManagerRunner 即TaskManager
```
3) 访问JobManager的web管理界面，端口8081

[![flink web页面](https://files.catbox.moe/k1vh5y.jpg)](http://graph.baidu.com/resource/126fdb8412540ee35f36c01596608099.jpg)

## 3.4 Flink提交任务
### 3.4.1 第一种方式：通过web页面提交

[![web提交任务](https://s.pc.qq.com/tousu/img/20210112/2942589_1610432645.jpg)](https://files.catbox.moe/u6c38x.jpg)

[![参数设置](https://s.pc.qq.com/tousu/img/20210112/3428103_1610432645.jpg)](https://files.catbox.moe/i2sxfa.jpg)

停止任务：<br>
Jobs -> Running Jobs -> 正在执行的任务 -> 右上角Cancel Job

### 3.4.2 第二种方式：使用命令行提交

```bash
bin/flink run m node-1:8081 -p 4 -c org.apache.flink.streaming.examples.socket.SocketWindowWordCount examples/streaming/SocketwindowWordCount.jar --hostname node-1 --port 8888
```
参数说明：
- -m 指定主机名后面的端口为 **==JobManager的 REST的端口==**，而不是RPC的端口，RPC通信端口是6123
- -p 指定是并行度
- -c 指定main方法的全类名

# 4 Flink 编程入门
## 4.1 初始化Flink 项目模板
### 4.1.1 准备工作
要求安装Maven3.0.4及以上版本和JDK8
### 4.1.2 使用maven命令创建java项目模板
1) 执行maven命令，如果maven本地仓库没有依赖的jar，需要有网络

```bash
mvn archetype:generate
-DarchetypeGroupld-org.apache.flink
-DarchetypeArtifactld=flink-quickstart-java
-DarchetypeVersion=1.9.1
-Dgroupld-cn.51doit.flink
-Dartifactld=flink-java
-Dversion=1.0
-Dpackage=n._51doit.flink
-DinteractiveMode=false
```

2) 或者在命令行中执行下面的命令，需要有网络

```bash
curl https://flink.apache.org/q/quickstart.sh | bash -s 1.9.1
```

### 4.1.3 使用maven命令创建scala项目模板
1) 执行maven命令，如果maven本地仓库没有依赖的jar，需要有网络

```
mvn archetype:generate
-DarchetypleGroupld=org.apache.flink
-DarchetypeArtifactld=flink-quickstart-scala
-DarchetypeVersion=1.9.1
-Dgroupld=cn._51doit.flink
-Dartifactld=flink-scala
-Dversion=1.0
-Dpackage=cn._51doit.flink
-DinteractiveMode=false
```

2) 或者在命令行中执行下面的命令，需要有网络

```bash
curl htps:/flink.apache.org/q/quickstart-scala.sh | bash -s 1.9.1
```

## 4.2 DataFlow编程模型


```
graph LR
DataSource(Data Source) --> Transformation(Transformation)
Transformation --> DataSink(Data Sink)
```


Flink 提供了不同级别的编程抽象，通过调用抽象的数据集调用算子构建 DataFlow就可以实现对分布式的数据进行流式计算和离线计算，**DataSet是批处理的抽象数据集，DataStream是流式计算的抽象数据集**，他们的方法都分别为 **==Source、Transformation、Sink==**

- **Source** 主要负责数据的读取
- **Transformation** 主要负责对数据的转换操作
- **Sink** 负责最终计算好的结果数据输出

## 4.3 Flink 第一个入门程序
### 4.3.1 实时WordCount
从一个Socket 端口中实时的读取数据，然后实时统计相同单词出现的次数，该程序会一直运行，启动程序前先使用`nc -l 8888`启动一个socket用来发送数据

**java版**
```java
public class StreamWordCount {

    public static void main(String[] args) throws Exception {

        //创建一个flink stream 程序的执行环境
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        //使用StreamExecutionEnvironment创建DataStream
        //Source
        DataStream<String> lines = env.socketTextStream(args[0], Integer.parseInt(args[1]));

        //Transformation开始

        //调用DataStream上的方法Transformation（s）
        SingleOutputStreamOperator<String> words = lines.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public void flatMap(String line, Collector<String> out) throws Exception {
                //切分
                String[] words = line.split(" ");
                for (String word : words) {
                    //输出
                    out.collect(word);
                }
            }
        });

        //将单词和一组合
        SingleOutputStreamOperator<Tuple2<String, Integer>> wordAndOne = words.map(new MapFunction<String, Tuple2<String, Integer>>() {
            @Override
            public Tuple2<String, Integer> map(String word) throws Exception {
                return Tuple2.of(word, 1);
            }
        });


        SingleOutputStreamOperator<Tuple2<String, Integer>> summed = wordAndOne.keyBy(0).sum(1);

        //Transformation结束

        // 调用Sink (Sink必须调用)
        summed.print();

        //启动
        env.execute("StreamWordCount");

    }
}
```

**Scala版**


```Scala

object StreamWorldCount {
def main(args: Array[String]): Unit = {
    val env = StreamExecutionEnvironment.getExecutionEnvironment

    //Source
    val lines: DataStream[String] = env.socketTextStream("222.180.220.157", 8888) //nc -lk 8888

    //Transformation开始
    // 隐式转换，导入org.apache.flink.streaming.api.scala下所有包
    import org.apache.flink.streaming.api.scala._
    val words = lines.flatMap(line => {
      val str = line.replaceAll("[\\pP+~$`^=|<>～｀＄＾＋＝｜＜＞￥×]", "") //替换特殊字符
      str.split("\\s+") //按空白字符切分， \\s表示 空格,回车,换行等空白符, +号表示一个或多个的意思
    })

    val wordAndOne: DataStream[(String, Int)] = words.map((_, 1))

    val grouped = wordAndOne.keyBy(0)

    val summed = grouped.sum(1)

    // 调用sink将结果打印
    summed.print()

    env.execute("scala StreamWorldCount")
  }
}
 
```


### 4.3.2 离线WordCount
**Java版**

```java
public class BatchWordCount {

    public static void main(String[] args) throws Exception {

        //离线批处理使用的执行换行是ExecutionEnvironment，少了Stream
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();

        //使用ExecutionEnvironment创建DataSet
        DataSet<String> lines = env.readTextFile(args[0]);

        //去除标点符号
        MapOperator<String, String> etlLines = lines.map(new MapFunction<String, String>() {
            @Override
            public String map(String line) throws Exception {
                String str = line.replaceAll("[\\pP+~$`^=|<>～｀＄＾＋＝｜＜＞￥×]", "");
                return str;
            }
        });

        //切分压平
        FlatMapOperator<String, Tuple2<String, Integer>> wordAndOne = etlLines.flatMap(new FlatMapFunction<String, Tuple2<String, Integer>>() {
            @Override
            public void flatMap(String line, Collector<Tuple2<String, Integer>> out) throws Exception {
                String[] words = line.split(" ");
                for (String word : words) {
                    out.collect(Tuple2.of(word, 1));
                }
            }
        });

        //离线计算实现的是分组聚合，调用的是groupBy
        AggregateOperator<Tuple2<String, Integer>> summed = wordAndOne.groupBy(0).sum(1);

        //每个分区进行排序
        SortPartitionOperator<Tuple2<String, Integer>> sortPartition = summed.sortPartition(1, Order.DESCENDING).sortPartition(0, Order.ASCENDING);

        //将结果保存到HDFS
        sortPartition.writeAsText(args[1]).setParallelism(2);

        // print()会调用execute，print和execute不能都调用
        //The last execution refers to the latest call to 'execute()', 'count()', 'collect()', or 'print()'.
        env.execute("BatchWordCount");
    }
}

```

**Scala版**

```scala
object BatchWordCount {
  def main(args: Array[String]): Unit = {

    val env = ExecutionEnvironment.getExecutionEnvironment

    //Source
    val lines: DataSet[String] = env.readTextFile("E:\\wordconut")

    //Transformation 开始
    // 导入隐式转换
    import org.apache.flink.api.scala._
    // \\s表示 空格,回车,换行等空白符, +号表示一个或多个的意思
    val words: DataSet[String] = lines.flatMap(_.split("\\s+")) //获取数据并按空白字符切分，生成一个个单词

    val wordAndOne: DataSet[(String, Int)] = words.map((_, 1))

    val grouped: GroupedDataSet[(String, Int)] = wordAndOne.groupBy(0)

    val summed: AggregateDataSet[(String, Int)] = grouped.sum(1)
    
    //排序
    val sorResult: DataSet[(String, Int)] = summed.sortPartition(1, Order.DESCENDING).sortPartition(0, Order.ASCENDING)
   
    sorResult.print()

    //Transformation 结束
   // sorResult.writeAsText("E:\\out")

    sorResult.print()

    env.execute("BatchWordCount")

  }
}
```

> 若直接按照样例执行，可能出现以下错误：
> 
> ```java
> Exception in thread "main" java.lang.RuntimeException: No new data sinks have been defined since the last execution. The last execution refers to the latest call to 'execute()', 'count()', 'collect()', or 'print()'.
> ```
> 
> 原因是print()方法自动会调用execute()方法，造成错误，所以调用打印的时候，注释掉env.execute()即可


### 4.3.3 运行程序

● 本地运行

[![](https://s6.jpg.cm/2021/11/05/IhFpky.jpg)](https://pic.imgdb.cn/item/6184cb312ab3f51d91ff5864.png)

● 提交到集群运行


```bash
bin/flink run -m node-1.51doit.cn:8081 -p 4 -c cn._51doit.flink.StreamingWordCount /root/hello-flink-java-1.0.jar node-1.51doit.cn 8888
```
