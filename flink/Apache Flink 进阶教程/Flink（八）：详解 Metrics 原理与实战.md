[TOC]

参考：https://ververica.cn/developers/advanced-tutorial-2-principles-and-practice-of-metrics

# 什么是 Metrics？

Flink 提供的 Metrics 可以在 Flink 内部收集一些指标，通过这些指标让开发人员更好地理解作业或集群的状态。由于集群运行后很难发现内部的实际状况，跑得慢或快，是否异常等，开发人员无法实时查看所有的 Task 日志，比如作业很大或者有很多作业的情况下，该如何处理？此时 Metrics 可以很好的帮助开发人员了解作业的当前状况。

## Metric Types

Metrics 的类型如下：

1.  首先，常用的如 Counter，写过 mapreduce 作业的开发人员就应该很熟悉 Counter，其实含义都是一样的，就是对一个计数器进行累加，即对于多条数据和多兆数据一直往上加的过程。
    
2.  第二，Gauge，Gauge 是最简单的 Metrics，它反映一个值。比如要看现在 Java heap 内存用了多少，就可以每次实时的暴露一个 Gauge，Gauge 当前的值就是heap使用的量。
    
3.  第三，Meter，Meter 是指统计吞吐量和单位时间内发生“事件”的次数。它相当于求一种速率，即事件次数除以使用的时间。
    
4.  第四，Histogram，Histogram 比较复杂，也并不常用，Histogram 用于统计一些数据的分布，比如说 Quantile、Mean、StdDev、Max、Min 等。
    

## Metric Group

Metric 在 Flink 内部有多层结构，以 Group 的方式组织，它并不是一个扁平化的结构，Metric Group + Metric Name 是 Metrics 的唯一标识。

Metric Group 的层级有 TaskManagerMetricGroup 和TaskManagerJobMetricGroup，每个 Job 具体到某一个 task 的 group，task 又分为 TaskIOMetricGroup 和 OperatorMetricGroup。Operator 下面也有 IO 统计和一些 Metrics，整个层级大概如下图所示。Metrics 不会影响系统，它处在不同的组中，并且 Flink支持自己去加 Group，可以有自己的层级。

```
•TaskManagerMetricGroup
    •TaskManagerJobMetricGroup
        •TaskMetricGroup
            •TaskIOMetricGroup
            •OperatorMetricGroup
                •${User-defined Group} / ${User-defined Metrics}
                •OperatorIOMetricGroup
•JobManagerMetricGroup
    •JobManagerJobMetricGroup
```

JobManagerMetricGroup 相对简单，相当于 Master，它的层级也相对较少。

Metrics 定义还是比较简单的，即指标的信息可以自己收集，自己统计，在外部系统能够看到 Metrics 的信息，并能够对其进行聚合计算。

# 如何使用 Metrics？

## System Metrics

System Metrics，将整个集群的状态已经涵盖得非常详细。具体包括以下方面：

*   Master 级别和 Work 级别的 JVM 参数，如 load 和 time；其 Memory 划分也很详细，包括 heap 的使用情况，non-heap 的使用情况，direct 的使用情况，以及 mapped 的使用情况；Threads 可以看到具体有多少线程；还有非常实用的 Garbage Collection。
    
*   Network 使用比较广泛，当需要解决一些性能问题的时候，Network 非常实用。Flink 不只是网络传输，还是一个有向无环图的结构，可以看到它的每个上下游都是一种简单的生产者消费者模型。Flink 通过网络相当于标准的生产者和消费者中间通过有限长度的队列模型。如果想要评估定位性能，中间队列会迅速缩小问题的范围，能够很快的找到问题瓶颈。
    

```
•CPU
•Memory
•Threads
•Garbage Collection
•Network
•Classloader
•Cluster
•Availability
•Checkpointing
•StateBackend
•IO
•详见: [https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/metrics.html#system-metrics](https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/metrics.html)
```

*   运维集群的人会比较关心 Cluster 的相关信息，如果作业太大，则需要非常关注 Checkpointing，它有可能会在一些常规的指标上无法体现出潜在问题。比如 Checkpointing 长时间没有工作，数据流看起来没有延迟，此时可能会出现作业一切正常的假象。另外，如果进行了一轮 failover 重启之后，因为 Checkpointing 长时间没有工作，有可能会回滚到很长一段时间之前的状态，整个作业可能就直接废掉了。
    
*   RocksDB 是生产环境当中比较常用的 state backend 实现，如果数据量足够大，就需要多关注 RocksDB 的 Metrics，因为它随着数据量的增大，性能可能会下降。
    

## User-defined Metrics

除了系统的 Metrics 之外，Flink 支持自定义 Metrics ，即 User-defined Metrics。上文说的都是系统框架方面，对于自己的业务逻辑也可以用 Metrics 来暴露一些指标，以便进行监控。

User-defined Metrics 现在提及的都是 datastream 的 API，table、sql 可能需要 context 协助，但如果写 UDF，它们其实是大同小异的。

Datastream 的 API 是继承 RichFunction ，继承 RichFunction 才可以有 Metrics 的接口。然后通过 RichFunction 会带来一个 getRuntimeContext().getMetricGroup().addGroup(…) 的方法，这里就是 User-defined Metrics 的入口。通过这种方式，可以自定义 user-defined Metric Group。如果想定义具体的 Metrics，同样需要用getRuntimeContext().getMetricGroup().counter/gauge/meter/histogram(…) 方法，它会有相应的构造函数，可以定义到自己的 Metrics 类型中。

```
继承 RichFunction
    •Register user-defined Metric Group: getRuntimeContext().getMetricGroup().addGroup(…)
    •Register user-defined Metric: getRuntimeContext().getMetricGroup().counter/gauge/meter/histogram(…)
```

## User-defined Metrics Example

下面通过一段简单的例子说明如何使用 Metrics。比如，定义了一个 Counter 传一个 name，Counter 默认的类型是 single counter（Flink 内置的一个实现），可以对 Counter 进行 inc（）操作，并在代码里面直接获取。

Meter 也是这样，Flink 有一个内置的实现是 Meterview，因为 Meter 是多长时间内发生事件的记录，所以它是要有一个多长时间的窗口。平常用 Meter 时直接 markEvent()，相当于加一个事件不停地打点，最后用 getrate（） 的方法直接把这一段时间发生的事件除一下给算出来。

Gauge 就比较简单了，把当前的时间打出来，用 Lambda 表达式直接把 System::currentTimeMillis 打进去就可以，相当于每次调用的时候都会去真正调一下系统当天时间进行计算。

Histogram 稍微复杂一点，Flink 中代码提供了两种实现，在此取一其中个实现，仍然需要一个窗口大小，更新的时候可以给它一个值。

这些 Metrics 一般都不是线程安全的。如果想要用多线程，就需要加同步，更多详情请参考下面链接。

```
•Counter processedCount = getRuntimeContext().getMetricGroup().counter("processed_count");
  processedCount.inc();
•Meter processRate = getRuntimeContext().getMetricGroup().meter("rate", new MeterView(60));
  processRate.markEvent();
•getRuntimeContext().getMetricGroup().gauge("current_timestamp", System::currentTimeMillis);
•Histogram histogram = getRuntimeContext().getMetricGroup().histogram("histogram", new DescriptiveStatisticsHistogram(1000));
  histogram.update(1024);
•[https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/metrics.html#metric-types]
```

## 获取 Metrics

获取 Metrics 有三种方法，首先可以在 WebUI 上看到；其次可以通过 RESTful API 获取，RESTful API 对程序比较友好，比如写自动化脚本或程序，自动化运维和测试，通过 RESTful API 解析返回的 Json 格式对程序比较友好；最后，还可以通过 Metric Reporter 获取，监控主要使用 Metric Reporter 功能。

获取 Metrics 的方式在物理架构上是怎样实现的？

了解背景和原理会对使用有更深刻的理解。WebUI 和 RESTful API 是通过中心化节点定期查询把各个组件中的 Metrics 拉上来的实现方式。其中，fetch 不一定是实时更新的，默认为 10 秒，所以有可能在 WebUI 和 RESTful API 中刷新的数据不是实时想要得到的数据；此外，fetch 有可能不同步，比如两个组件，一边在加另一边没有动，可能是由于某种原因超时没有拉过来，这样是无法更新相关值的，它是 try best 的操作，所以有时我们看到的指标有可能会延迟，或许等待后相关值就更新了。

红色的路径通过 MetricFetcher，会有一个中心化的节点把它们聚合在一起展示。而 MetricReporter 不一样，每一个单独的点直接汇报，它没有中心化节点帮助做聚合。如果想要聚合，需要在第三方系统中进行，比如常见的 TSDB 系统。当然，不是中心化结构也是它的好处，它可以免去中心化节点带来的问题，比如内存放不下等，MetricReporter 把原始数据直接 Reporter 出来，用原始数据做处理会有更强大的功能。

![](https://ververica.cn/wp-content/uploads/2019/12/img1-5.png)

## Metric Reporter

Flink 内置了很多 Reporter，对外部系统的技术选型可以参考，比如 JMX 是 java 自带的技术，不严格属于第三方。还有InfluxDB、Prometheus、Slf4j（直接打 log 里）等，调试时候很好用，可以直接看 logger，Flink 本身自带日志系统，会打到 Flink 框架包里面去。详见：

```
•Flink 内置了很多 Reporter，对外部系统的技术选型可以参考，详见：[https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/metrics.html#reporter](https://ci.apache.org/projects/flink/flink-docs-release-1.8/monitoring/metrics.html)
•Metric Reporter Configuration Example
 metrics.reporters: your_monitor,jmx
 metrics.reporter.jmx.class: org.apache.flink.metrics.jmx.JMXReporter
 metrics.reporter.jmx.port: 1025-10000
 metrics.reporter.your_monitor.class: com.your_company.YourMonitorClass
 metrics.reporter.your_monitor.interval: 10 SECONDS
 metrics.reporter.your_monitor.config.a: your_a_value
 metrics.reporter.your_monitor.config.b: your_b_value
```

Metric Reporter 是如何配置的？如上所示，首先 Metrics Reporters 的名字用逗号分隔，然后通过 metrics.reporter.jmx.class 的 classname 反射找 reporter，还需要拿到 metrics.reporter.jmx.port 的配置，比如像第三方系统通过网络发送的比较多。但要知道往哪里发，ip 地址、port 信息是比较常见的。此外还有 metrics.reporter.your\_monitor.class 是必须要有的，可以自己定义间隔时间，Flink 可以解析，不需要自行去读，并且还可以写自己的 config。

# 实战：利用 Metrics 监控

常用 Metrics 做自动化运维和性能分析。

## 自动化运维

![](https://ververica.cn/wp-content/uploads/2019/12/img2-2-1024x576.png)

自动化运维怎么做？

*   首先，收集一些关键的 Metrics 作为决策依据，利用 Metric Reporter 收集 Metrics 到存储/分析系统 (例如 TSDB)，或者直接通过 RESTful API 获取。
    
*   有了数据之后，可以定制监控规则，关注关键指标，Failover、Checkpoint,、业务 Delay 信息。定制规则用途最广的是可以用来报警，省去很多人工的工作，并且可以定制 failover 多少次时需要人为介入。
    
*   当出现问题时，有钉钉报警、邮件报警、短信报警、电话报警等通知工具。
    
*   自动化运维的优势是可以通过大盘、报表的形式清晰的查看数据，通过大盘时刻了解作业总体信息，通过报表分析优化。
    

## 性能分析

性能分析一般遵循如下的流程：

![](https://ververica.cn/wp-content/uploads/2019/12/img3-2-1024x506.png)

首先从发现问题开始，如果有 Metrics 系统，再配上监控报警，就可以很快定位问题。然后对问题进行剖析，大盘看问题会比较方便，通过具体的 System Metrics 分析，缩小范围，验证假设，找到瓶颈，进而分析原因，从业务逻辑、JVM、 操作系统、State、数据分布等多维度进行分析；如果还不能找到问题原因，就只能借助 profiling 工具了。

## 实战：“我的任务慢，怎么办”

“任务慢，怎么办？”可以称之为无法解答的终极问题之一。

其原因在于这种问题是系统框架问题，比如看医生时告诉医生身体不舒服，然后就让医生下结论。而通常医生需要通过一系列的检查来缩小范围，确定问题。同理，任务慢的问题也需要经过多轮剖析才能得到明确的答案。

除了不熟悉 Flink 机制以外，大多数人的问题是对于整个系统跑起来是黑盒，根本不知道系统在如何运行，缺少信息，无法了解系统状态。此时，一个有效的策略是求助 Metrics 来了解系统内部的状况，下面通过一些具体的例子来说明。

### 发现问题

比如下图 failover 指标，线上有一个不是 0，其它都是 0，此时就发现问题了。

![](https://ververica.cn/wp-content/uploads/2019/12/img4-1-1024x360.png)

再比如下图 Input 指标正常都在四、五百万，突然跌成 0，这里也存在问题。

![](https://ververica.cn/wp-content/uploads/2019/12/img5-2-1024x419.png)

业务延时问题如下图，比如处理到的数据跟当前时间比对，发现处理的数据是一小时前的数据，平时都是处理一秒之前的数据，这也是有问题的。

![](https://ververica.cn/wp-content/uploads/2019/12/img6-2.png)

### 缩小范围，定位瓶颈

当出现一个地方比较慢，但是不知道哪里慢时，如下图红色部分，OUT\_Q 并发值已经达到 100% 了，其它都还比较正常，甚至优秀。到这里生产者消费者模型出现了问题，生产者 IN\_Q 是满的，消费者 OUT\_Q 也是满的，从图中看出节点 4 已经很慢了，节点 1 产生的数据节点 4 处理不过来，而节点 5 的性能都很正常，说明节点 1 和节点 4 之间的队列已经堵了，这样我们就可以重点查看节点 1 和节点 4，缩小了问题范围。

![](https://ververica.cn/wp-content/uploads/2019/12/img7-1.png)

500 个 InBps 都具有 256 个 PARALLEL ，这么多个点不可能一一去看，因此需要在聚合时把 index 是第几个并发做一个标签。聚合按着标签进行划分，看哪一个并发是 100%。在图中可以划分出最高的两个线，即线 324 和线 115，这样就又进一步的缩小了范围。

![](https://ververica.cn/wp-content/uploads/2019/12/img8-1.png)

利用 Metrics 缩小范围的方式如下图所示，就是用 Checkpoint Alignment 进行对齐，进而缩小范围，但这种方法用的较少。

![](https://ververica.cn/wp-content/uploads/2019/12/img9-1-1024x462.png)

### 多维度分析

分析任务有时候为什么特别慢呢？

当定位到某一个 Task 处理特别慢时，需要对慢的因素做出分析。分析任务慢的因素是有优先级的，可以从上向下查，由业务方面向底层系统。因为大部分问题都出现在业务维度上，比如查看业务维度的影响可以有以下几个方面，并发度是否合理、数据波峰波谷、数据倾斜；其次依次从 Garbage Collection、Checkpoint Alignment、State Backend 性能角度进行分析；最后从系统性能角度进行分析，比如 CPU、内存、Swap、Disk IO、吞吐量、容量、Network IO、带宽等。

# Q&A

*   **Metrics 是系统内部的监控，那是否可以作为 Flink 日志分析的输出？**
    

可以，但是没有必要，都用 Flink 去处理其他系统的日志了，输出或报警直接当做 sink 输出就好了。因为 Metrics 是统计内部状态，你这是处理正常输入数据，直接输出就可以了。

*   **Reporter 是有专门的线程吗？**
    

每个 Reporter 都有自己单独的线程。在Flink的内部，线程其实还是挺多的，如果跑一个作业，直接到 TaskManager 上，jstack 就能看到线程的详情。