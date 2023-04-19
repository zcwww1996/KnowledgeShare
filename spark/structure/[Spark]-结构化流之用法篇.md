[TOC]

来源：https://www.cnblogs.com/NightPxy/p/9278873.html

结构化流使用Datasets和DataFrames.从Spark2.0开始,Spark-SQL中的Datasets和DataFrames,就已经能很好表示静态(有界)数据,动态(无界)数据

# 1. 数据源

结构化流提供了四种不中断数据源 file-system,kafka,socket.rate-source　

## 1.1 socket

从一个socket连接中读取 UTF-8 的文本数据. <=注意这是一种不可容错的数据源,建议仅在测试环境中使用.

配置参数:

- host 连接地址
- port 端口号

## 1.2 rate-source

这是结构流内置的模拟数据生成的数据源.这也是一种不可容错的数据源.只能在测试环境使用

它每秒,以指定的设置生成N行的数据.每行记录包含一个`timestamp(分发时间)` 和 value (消息序号(count),从0开始)

配置参数:

| 属性 | 描述 |
|---|---|
| `rowsPerSecond`  | 每秒生成N行.默认1 |
| `rampUpTime`  | 生成速度. 默认 0(秒) |
| `numPartitions`  | 生成数据的分区数.默认读取 Spark's default parallelism |

## 1.3 文件系统

### 1.3.1 概述&配置

  从一个文件目录读取文件流. <=可容错数据源

 支持的格式包括: text, csv, json, orc, parquet.需要注意的是,文件必须以原子方式放入目录(例如文件移动进目标目录).

 配置参数如下:

| 属性 | 描述 |
|---|---|
| path | 输入路径.支持文件通配符匹配,但不支持多个逗号分割的文件通配符匹配 |
| maxFilesPerTrigger | 每个触发器的最大新文件数量.默认不限 |
| latestFirst | 是否先处理新文件.当文件大量积压时比较有用.默认false |
| fileNameOnly | 是否只以文件名而忽略完整路径,来作为检查新文件的标准.默认false |

### 1.3.2 文件系统的元数据接口

默认情况下,基于文件系统的结构化流需要指定文件的元数据(schema),而不是依靠Spark进行类型推断

但可以配置 spark.sql.streaming.schemaInference  设为true(默认false)来重新启用 schema inference(元数据接口)

### 1.3.3 文件系统的分区发现

结构化流支持自动以目录为凭据的分区发现(key=value)

分区发现时,分区列必须出现在用户提供的schema中.

当分区列出现在查询中,此时它将会被路径上发现的分区信息自动填充.

## 1.2 kafka

从kafka中读取数据.<=可容错数据源  (只支持 kafka 0.10.0 或以上版本)

kafka 是最推荐使用的方式,后面详解

# 2. 事件时间窗口操作

## 2.1 概述

在很多情况下,我们并不希望针对整个流进行聚合.而是希望通过一个时间窗口(每5分钟内,或每小时),在一个自定义的小时间分片的范围内数据进行聚合.

并且这种时间分片的依据,是类似事件时间这种业务概念时间,而不是以结构化流自身维护的收到时间为依据.

这种情况下,结构化流提供了创建窗口函数(window).通过指定窗口大小(范围),再滑动窗口位置进行聚合(sliding event-time window),来非常方便的处理这种情况

## 2.2 窗口函数使用

 窗口函数需要以下两个参数:

窗口范围(_window length) _就是统计的时间范围

滑动间隔(_sliding interval_) 就是统计间隔,每多长时间出一次结果


```scala
object StructuredSteamingWindowApp extends App {
            val spark = SparkSession.builder().master("local[2]").appName("Structured-Steaming-Window-App").getOrCreate();
            
            import spark.implicits._;
            
            val lines = spark.readStream.format("socket").option("host", "192.168.178.1").option("port", "9999").load();
            
            //格式化分割:abcd|1531060985000 efg|1531060984000
            val words = lines.as[String]
                .flatMap(line => line.split(" "))
                .map(word => word.split("\\|"))
                .map(wordItem => (wordItem(0), new Timestamp(wordItem(1).toLong)))
                .as[(String, Timestamp)]
                .toDF("word", "timestamp");
            
            //窗口范围,就是统计数据的时间范围=>30秒范围内
            val windowDuration = "30 seconds";
            //滑动时间,就是每多长时间计算一个结果=>每10秒计算一次
            //以上综合就是 每10秒出一次最近30秒的结果
            val slideDuration = "30 seconds";
            //以窗口计算的事件时间+单词分组 统计计数
            val windowedCounts = words.groupBy(window($"timestamp", windowDuration, slideDuration), $"word").count().orderBy("window")
            
            //完整统计结果输出到控制台
            val query = windowedCounts.writeStream
                .outputMode("complete")
                .format("console")
                .option("truncate", "false")
                .option("checkpointLocation", "D:\\Tmp")
                .start()
            
            query.awaitTermination()
            }
```



输出结果:

![](https://images2018.cnblogs.com/blog/1409212/201807/1409212-20180708234432731-1818315902.png)

## 2.3 数据延迟与水印

事件窗口很好的解决了自定义的小时间分片的范围内数据进行聚合这种设计.但实际过程还有一个比较常见的问题:因某种原因,部分数据到达延迟了.

想象一下,一个应该12:05到达的数据,本应该进入12:00-12:10切片的,但实际在12:15才达到,结果被归入了12:10-12:20的分片中被聚合.

针对这种问题,结构化流提供一种解决方案: **水印**(**watermarking**)

水印是指让引擎自动的跟踪数据中的事件事件(current-event-time),并根据用户指定来清理旧状态.

用户可以指定水印的事件时间列(event time column),和数据预期的延迟阈值.

对于一个从T时间开始的窗口,引擎将保持状态并将延迟到达的数据(记录事件时间>系统当前时间-延迟阈值)重新更新状态.(就是水印时间之内的数据并入计算,水印之外抛弃)

使用:

.withWatermark("事件时间列", "水印状态持续时间")

使用水印时,务必注意以下:

**水印时间,是以当前最大的事件时间减去水印延时的时间,切记它跟窗口范围没有关系**

**输出模式必须是追加或者更新(不支持完全模式)**. 因为水印是去除过期数据,与完全模式保留所有聚合数据冲突了(完全模式下,水印将会失效)

**使用水印的聚合必须具有事件时间列或运行在有事件时间的窗口上.并且水印的时间列必须与事件时间列一致**.

**使用水印必须在聚合之前使用**,否则水印不会起效

**==追加模式下的水印==**:

**append的核心之处在于append不允许变更.所以必须是能产生最终结果的时候才能输出计算结果.而水印状态下,是可能有因为数据延时而后到前插的数据的**

**所以水印状态append模式下的最终结果输出,是在水印时间彻底离开一个窗口的范围后,才会对窗口数据进行计算**

_这是官网的图:_

![](https://images2018.cnblogs.com/blog/1409212/201807/1409212-20180709234926969-80060201.png)

**12:14**

Row(12:14,dog)到达,它属于窗口(12:10-12:20).<br>
此时的水印时间是12:04=12:14(水印的基准不是窗口时间12:15,而是当前记录里最大事件时间12:14)-10<br>
而水印时间(12:04),还在窗口(12:00-12:10)中,<br>
所以窗口(12:00-12:10)没有任何结果输出,并且如果此时有Row(12:09)延迟到现在才到,是可以被窗口(12:00-12:10)正确统计的  

**12:15** 

Row(12:15,cat) 同上.  

**12:21**

Row(12:21,owi)到达,它属于窗口(12:20-12:30)<br>
此时的水印时间是12:11(12:21-10),离开窗口(12:10-12:20)<br>
直到此时,当窗口(12:20-12:30)被12:25的触发器触发,才会生成12:10-12:20的最终结果.此时Row(12:08)数据会被忽略

# 3. 两个流数据之间的连接

从Spark2.3开始,结构化流提供了两个流数据之间的join.

两个流数据连接的困难之处在于,两个无界数据集,其中一个的记录,可能会与另一数据集的将来某一条记录匹配.

因次,流数据之间的连接,会将过去的输入缓冲为流状态.以便在收到每一条记录时都会尝试与过去的输入连接,并相应的生成匹配结果.

并且这种连接依然保持自动处理延迟和无序的数据,并可以使用水印限制状态.下面是一些流数据连接的介绍

## 3.1 内连接(inner join)和水印

流数据内联支持任意字段和任意条件内联.

但是随着流数据的不断进入,流状态数据也会不断的增加.所以必须定义连接的附加条件,能让流状态能够将明确不可能再有机会与将来数据匹配的数据清除.

换句话说,必须再内连接中定义以下步骤:

1) 在两个流上都定义水印延迟,从而让流数据能被水印优雅的去除(与流聚合类似)
2) 定义带事件时间的约束条件,从而能让引擎能够根据这些条件从流状态去捡出不符合条件的数据.比如如下两种方式
    - 连接条件上带事件时间过滤
    - 带事件时间的窗口

## 3.2 外连接(out join)和水印

外连接基本与内连接基本相同.但外连接需要更加注意的是可能大量存在的Null值,而这些Null值在未来又是可能有匹配的,所以在时间附加条件上必须要做的更加严格.

对于外连接,有一些重要的特征如下:

1) 对于Null或不匹配的值,必须有一个完善的方案(水印或者时间区域),来确定未来不会有匹配从而优雅的移除
2) 在微批处理的引擎中,水印在微批处理的最后做了改进: 由下一个批次才会使用更新后的水印清洗结果来输出外部结果.

　由于只有在出现新数据时才会触发微批处理,因此当某(任)一个数据流没有输入数据的时候,都会造成外部结果的延迟

另外,对于连接还有一些需要知道的:

1) 连接是级联.即可以是 df1.join(df2, ...).join(df3, ...).join(df4, ....)
2) 从Spark2.3开始,只有在查询输出模式为 Append output mode 时,才能使用连接.其它输出模式不支持
3) 在join之前,不能有 非map系(non-map-like) 的操作.比如说,以下操作是不允许的
    - 在join之前,不能有任何聚合操作
    - 在Update mode下,join之前不能使用mapGroupsWithState 或者flatMapGroupsWithState 之类的操作

关于连接具体支持如下:

| 左 | 右 | join类型 | 描述 |
|---|---|---|---|
| Static | Static | 支持所有join类型 | 这是静态数据集而非流式数据集 |
| Stream | Static | Inner | 支持连接,但不支持流状态 |
| Stream | Static |Left Outer | 支持连接,但不支持流状态 |
| Stream | Static | Right Outer | 不支持 |
| Stream | Static | Full Outer | 不支持 |
| Static | Stream | Inner | 支持连接,但不支持流状态 |
| Static | Stream| Left Outer | 不支持 |
| Static | Stream| Right Outer | 支持连接,但不支持流状态 |
| Static | Stream| Full Outer | 不支持 |
| Stream | Stream | Inner | 支持连接,支持流状态.并且水印和事件时间范围可选 |
|   |   | Left Outer | 有条件的支持连接和流状态.对右数据必须使用水印或者事件范围条件,对左数据是可选水印和事件时间范围条件 |
|   |   | Right Outer | 有条件的支持连接和流状态.对左数据必须使用水印或者事件范围条件,对右数据是可选水印和事件时间范围条件 |
|   |   | Full Outer | 不支持 |

# 4. 流数据的去重

结构化流支持以数据中的某一唯一标识符(unique identifier)为凭据,对流数据内的记录进行重复删除(与静态数据批处理的唯一标识列重复消除用法完全相同).

重复删除完成机制是将暂时存储先前的记录,以便可以查询过滤重复的记录.注意这种暂存不与水印绑定,你可以选择水印,也可以不选择使用水印来完成暂存查询过滤重复的功能

**1) 水印完成**

如果数据延迟有其上限(这个上限指超过延迟后可以直接丢弃),则可以在事件时间上定义水印,并使用一个Guid和事件时间列完成去重

**2) 非水印完成**

如果数据延迟没有界限(指再晚到达都必须接受处理不能丢弃),这将查询所有过去记录的存储状态

使用:

.dropDuplicates("去除条件列".....)   <=去重

# 5. 有状态的操作

???

# 6. streaming DataFrames/Datasets 与 DataFrames/Datasets 的一些不同之处


```bash
streaming Datasets 不支持 Multiple streaming aggregations （多个流聚合）(即 streaming DF 上的聚合链)

streaming Datasets 不支持 Limit and take first N rows 

streaming Datasets 上的 Distinct operations 不支持
```

只有在 aggregation 和 Complete Output Mode 下，streaming Datasets 才支持排序操作

有条件的流数据集连接(详见两个流数据集连接)

此外,一些立即返回结果的操作对流数据也没有意义,比如:

```bash
count(),流数据不能返回计算,只能使用ds.groupBy().count()返回一个流媒体数据集,其中包含一个运行计数

foreach()- 使用 ds.writeStream.foreach(...) 代替

show()- 使用 控制台接收器 代替
```


尝试进行这些不支持的操作,会抛出一些 类似"operation XYZ is not supported with streaming DataFrames/Datasets"的`AnalysisException`