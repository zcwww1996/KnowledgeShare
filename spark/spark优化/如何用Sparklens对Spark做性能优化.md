[TOC]

Sparklens是一个可以帮助你了解你的Spark job效率的开源工具。Spark是个近些年来非常受欢迎的基于内存并行计算框架架，它有丰富的API支持，还支持 Spark SQL，MLlib，GraphX和Spark Streaming。在提交Spark Job的时候，我们会需要设置一些config, 虽然这极大的增加了开发者对于Spark job的控制灵活性，但是也会给优化Spark Job带来一些困难。我们平时写Spark Job的时候最长苦恼的应该就是如果和调节memoery,vcore这些参数，资源申请少了会造成job的失败，多了就会造成资源的浪费。所以Sparklens就是这么一个让你更加了解你的job运行情况，从而有效的进行Spark tuning的工具。

Sparklens源码：https://github.com/qubole/sparklens<br>
实时分析（Streaminglens）：https://github.com/qubole/streaminglens<br>
大数据监控工具（Dr.Elephant）：https://github.com/linkedin/dr-elephant

qubole公司优化jar包：https://cloud.189.cn/web/main/file/folder/21463114742280322

# 1. Sparklens如何使用


## 1.1 和你的Job一起运行

这种方式是指当你的job在运行的时候，sparklens会实时的开始收集job的运行信息，最后打印结果。只要你的spark客户端能联网，那么你只需要在你的提交命令里面加上如下两行（当然如果没有网，你也可以从网上下载package之后打包运行）：

**方法1（在线离线均可）**：
```
--packages qubole:sparklens:0.3.2-s_2.11 \
--conf spark.extraListeners=com.qubole.sparklens.QuboleJobListener \
```

- 联网默认下载到 当前用户根目录`/.ivy/repository/qubole/sparklens/0.3.2-s_2.11/sparklens-0.3.2-s_2.11.jar`
- 离线时可以在 当前用户根目录`/.ivy/repository/qubole/sparklens/0.3.2-s_2.11/sparklens-0.3.2-s_2.11.jar`或`/.m2/repository/qubole/sparklens/0.3.2-s_2.11/sparklens-0.3.2-s_2.11.jar`手动上传文件`sparklens-0.3.2-s_2.11.jar`

**方法2（离线）**：
```
--jars sparklens-0.3.2-s_2.11.jar \
--conf spark.extraListeners=com.qubole.sparklens.QuboleJobListener \
--conf spark.sparklens.data.dir=/user/zhangchao/sparklens \
```


**方法3（离线）**：

代码：

```Scala
val spark = SparkSession.builder().appName("NonHumanDailyReport_" + province)
  .config("spark.extraListeners", "com.qubole.sparklens.QuboleJobListener")
  .getOrCreate()
```

依赖：

```gradle
dependencies {
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.8.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.8.0'
    compileOnly group: 'org.scala-lang', name: 'scala-library', version: '2.11.12'
    compileOnly group: 'org.apache.spark', name: 'spark-sql_2.11', version: '2.4.0'
    implementation group: 'qubole', name: 'sparklens', version: '0.3.2-s_2.11'
}

repositories {
    mavenLocal()
    maven {
        allowInsecureProtocol = true
        url 'https://repos.spark-packages.org/'
    }
    mavenCentral()
}
```


## 1.2 Job跑完之后使用

这种方式会在你的job运行时生成一个Json文件在HDFS(默认)上，路径默认是/tmp/Sparklens。如果你想要设置自己的路径，可以通过配置`spark.sparklens.data.dir`来完成。

1.  首先你需要在运行你自己的spark job的时候在提交命令里面加上：

```
--packages qubole:sparklens:0.3.2-s_2.11
--conf spark.extraListeners=com.qubole.sparklens.QuboleJobListener
--conf spark.sparklens.reporting.disabled=true
--conf spark.sparklens.data.dir=/user/zhangchao/sparklens
```

2.  之后根据生成的Json文件，另外启动一个Job来获取分析你的Sparklens的报告结果

```
./bin/spark-submit --packages qubole:sparklens:0.3.1-s_2.11 --class com.qubole.sparklens.app.ReporterApp qubole-dummy-arg <filename>

```

## 1.3 从event-log里面获取

这种方式就是根据event log来生成Sparklens的报告，当然你需要在你的spark job里面enable event-log。

```
./bin/spark-submit --packages qubole:sparklens:0.3.1-s_2.11 --class com.qubole.sparklens.app.ReporterApp qubole-dummy-arg <filename> source=history

```

# 2. Sparklens生成的报告


这里给出一个spark程序来具体分析下,资源申请了50个executor(Spark on Yarn，相当于Yarn的container),20G的memory。 应用的是第一种Sparklens的使用方式，在job结束之后，从输出的log里拿到了Sparklens的报告结果。

具体参数：

```bash
--deploy-mode client \
--queue ss_deploy                     \
--driver-memory 25g                  \
--num-executors 50                  \
--executor-cores 2          \
--executor-memory 20g                \
--conf spark.default.parallelism=800  \
```

Sparklens的报告结果:

```bash
Printing application meterics. These metrics are collected at task-level granularity and aggregated across the app (all tasks, stages, and jobs).

 AggregateMetrics (Application Metrics) total measurements 1249 
                NAME                        SUM                MIN           MAX                MEAN         
 diskBytesSpilled                            0.0 KB         0.0 KB         0.0 KB              0.0 KB
 executorRuntime                             2.5 hh         1.0 ms         1.1 mm              7.2 ss
 inputBytesRead                             58.3 GB         0.0 KB       119.9 MB             47.8 MB
 jvmGCTime                                   2.4 mm         0.0 ms         8.0 ss            115.0 ms
 memoryBytesSpilled                          0.0 KB         0.0 KB         0.0 KB              0.0 KB
 outputBytesWritten                        164.5 MB         0.0 KB         8.2 MB            134.9 KB
 peakExecutionMemory                        76.5 GB         0.0 KB       235.3 MB             62.7 MB
 resultSize                                 12.5 MB         1.3 KB         9.3 MB             10.3 KB
 shuffleReadBytesRead                        1.3 GB         0.0 KB        36.9 MB              1.1 MB
 shuffleReadFetchWaitTime                    0.0 ms         0.0 ms         0.0 ms              0.0 ms
 shuffleReadLocalBlocks                       2,209              0             13                   1
 shuffleReadRecordsRead                  23,323,496              0        584,617              18,673
 shuffleReadRemoteBlocks                    104,191              0            508                  83
 shuffleWriteBytesWritten                    1.3 GB         0.0 KB         3.7 MB              1.1 MB
 shuffleWriteRecordsWritten              23,323,496              0         58,904              18,673
 shuffleWriteTime                           21.2 ss         0.0 ms       190.8 ms             17.0 ms
 taskDuration                                2.5 hh        10.0 ms         1.2 mm              7.3 ss




Total Hosts 13, and the maximum concurrent hosts = 13


Host lf319-rh2288l-107 startTime 05:32:30:295 executors count 3
Host lf319-rh2288l-181 startTime 05:32:37:582 executors count 2
Host lf319-rh2288l-172 startTime 05:32:31:064 executors count 5
Host lf319-xlpod3-045 startTime 05:32:31:219 executors count 4
Host lf-319-xd-087 startTime 05:32:30:346 executors count 3
Host lf319-xlpod3-071 startTime 05:32:29:915 executors count 9
Host lf-319-xd-027 startTime 05:32:35:799 executors count 2
Host lf319-rh2288l-159 startTime 05:32:53:504 executors count 2
Host lf319-rh2288l-132 startTime 05:32:29:447 executors count 5
Host lf319-xlpod3-082 startTime 05:32:30:687 executors count 4
Host lf319-rh2288l-158 startTime 05:32:31:005 executors count 3
Host lf-319-xd-026 startTime 05:32:30:739 executors count 4
Host lf319-rh2288l-155 startTime 05:32:30:449 executors count 4
Done printing host timeline
======================



Printing executors timeline....

Total Executors 50, and maximum concurrent executors = 50
At 05:32 executors added 50 & removed  0 currently available 50

Done printing executors timeline...
============================



Printing Application timeline 

05:32:16:325 app started 
05:32:33:005 JOB 0 started : duration 00m 01s 
[      0       ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:33:163      Stage 0 started : duration 00m 01s 
05:32:34:937      Stage 0 ended : maxTaskTime 1221 taskCount 512
05:32:34:952 JOB 0 ended 
05:32:36:609 JOB 1 started : duration 00m 00s 
[      1                  |||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:36:768      Stage 1 started : duration 00m 00s 
05:32:37:316      Stage 1 ended : maxTaskTime 479 taskCount 1
05:32:37:317 JOB 1 ended 
05:32:37:657 JOB 2 started : duration 00m 00s 
[      2               ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:37:792      Stage 2 started : duration 00m 00s 
05:32:38:399      Stage 2 ended : maxTaskTime 538 taskCount 1
05:32:38:400 JOB 2 ended 
05:32:38:613 JOB 3 started : duration 00m 01s 
[      3                       ||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:38:903      Stage 3 started : duration 00m 00s 
05:32:39:637      Stage 3 ended : maxTaskTime 630 taskCount 1
05:32:39:638 JOB 3 ended 
05:32:40:344 JOB 4 started : duration 00m 00s 
[      4 ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:40:352      Stage 4 started : duration 00m 00s 
05:32:41:032      Stage 4 ended : maxTaskTime 588 taskCount 1
05:32:41:033 JOB 4 ended 
05:32:40:351 JOB 5 started : duration 00m 01s 
[      5 ||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||| ]
05:32:40:357      Stage 5 started : duration 00m 01s 
05:32:42:000      Stage 5 ended : maxTaskTime 1333 taskCount 1
05:32:42:001 JOB 5 ended 
05:32:42:771 JOB 6 started : duration 02m 12s 
[      6       |||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||||           ]
[      7                                                                      ||||       ]
[      8                                                                          |||||| ]
05:32:53:473      Stage 6 started : duration 01m 43s 
05:34:37:164      Stage 6 ended : maxTaskTime 68302 taskCount 512
05:34:37:274      Stage 7 started : duration 00m 06s 
05:34:43:854      Stage 7 ended : maxTaskTime 6395 taskCount 200
05:34:43:914      Stage 8 started : duration 00m 11s 
05:34:55:292      Stage 8 ended : maxTaskTime 11321 taskCount 20
05:34:55:294 JOB 6 ended 
05:34:55:409 app ended 



Checking for job overlap...

 
 JobGroup 1  SQLExecID (-1)
 Number of Jobs 1  JobIDs(0)
 Timing [05:32:33:005 - 05:32:34:952]
 Duration  00m 01s
 
 JOB 0 Start 05:32:33:005  End 05:32:34:952
 
 
 JobGroup 2  SQLExecID (0)
 Number of Jobs 1  JobIDs(1)
 Timing [05:32:36:609 - 05:32:37:317]
 Duration  00m 00s
 
 JOB 1 Start 05:32:36:609  End 05:32:37:317
 
 
 JobGroup 3  SQLExecID (1)
 Number of Jobs 1  JobIDs(2)
 Timing [05:32:37:657 - 05:32:38:400]
 Duration  00m 00s
 
 JOB 2 Start 05:32:37:657  End 05:32:38:400
 
 
 JobGroup 4  SQLExecID (2)
 Number of Jobs 1  JobIDs(3)
 Timing [05:32:38:613 - 05:32:39:638]
 Duration  00m 01s
 
 JOB 3 Start 05:32:38:613  End 05:32:39:638
 
 
 JobGroup 5  SQLExecID (-1)
 Number of Jobs 1  JobIDs(4)
 Timing [05:32:40:344 - 05:32:41:033]
 Duration  00m 00s
 
 JOB 4 Start 05:32:40:344  End 05:32:41:033
 
 
 JobGroup 6  SQLExecID (-1)
 Number of Jobs 1  JobIDs(5)
 Timing [05:32:40:351 - 05:32:42:001]
 Duration  00m 01s
 
 JOB 5 Start 05:32:40:351  End 05:32:42:001
 
 
 JobGroup 7  SQLExecID (3)
 Number of Jobs 1  JobIDs(6)
 Timing [05:32:42:771 - 05:34:55:294]
 Duration  02m 12s
 
 JOB 6 Start 05:32:42:771  End 05:34:55:294
 
Found 1 overlapping JobGroups. Using threadpool for submitting parallel jobs? Some calculations might not be reliable.
Running with overlap:  JobGroupID 6 && JobGroupID 5 



 Time spent in Driver vs Executors
 Driver WallClock Time    00m 19s   12.45%
 Executor WallClock Time  02m 19s   87.55%
 Total WallClock Time     02m 39s
      


Minimum possible time for the app based on the critical path (with infinite resources)   01m 50s
Minimum possible time for the app with same executors, perfect parallelism and zero skew 01m 49s
If we were to run this app with single executor and single core                          02h 29m

       
 Total cores available to the app 100

 OneCoreComputeHours: Measure of total compute power available from cluster. One core in the executor, running
                      for one hour, counts as one OneCoreComputeHour. Executors with 4 cores, will have 4 times
                      the OneCoreComputeHours compared to one with just one core. Similarly, one core executor
                      running for 4 hours will OnCoreComputeHours equal to 4 core executor running for 1 hour.

 Driver Utilization (Cluster idle because of driver)

 Total OneCoreComputeHours available                             04h 25m
 Total OneCoreComputeHours available (AutoScale Aware)           03h 59m
 OneCoreComputeHours wasted by driver                            00h 32m

 AutoScale Aware: Most of the calculations by this tool will assume that all executors are available throughout
                  the runtime of the application. The number above is printed to show possible caution to be
                  taken in interpreting the efficiency metrics.

 Cluster Utilization (Executors idle because of lack of tasks or skew)

 Executor OneCoreComputeHours available                  03h 52m
 Executor OneCoreComputeHours used                       02h 29m        64.28%
 OneCoreComputeHours wasted                              01h 22m        35.72%

 App Level Wastage Metrics (Driver + Executor)

 OneCoreComputeHours wasted Driver               12.45%
 OneCoreComputeHours wasted Executor             31.28%
 OneCoreComputeHours wasted Total                43.72%

       


 App completion time and cluster utilization estimates with different executor counts

 Real App Duration 02m 39s
 Model Estimation  02m 21s
 Model Error       11%

 NOTE: 1) Model error could be large when auto-scaling is enabled.
       2) Model doesn't handles multiple jobs run via thread-pool. For better insights into
          application scalability, please try such jobs one by one without thread-pool.

       
 Executor count     5  ( 10%) estimated time 15m 31s and estimated cluster utilization 96.11%
 Executor count    10  ( 20%) estimated time 08m 04s and estimated cluster utilization 92.46%
 Executor count    25  ( 50%) estimated time 03m 43s and estimated cluster utilization 80.21%
 Executor count    40  ( 80%) estimated time 02m 42s and estimated cluster utilization 68.94%
 Executor count    50  (100%) estimated time 02m 21s and estimated cluster utilization 63.35%
 Executor count    55  (110%) estimated time 02m 14s and estimated cluster utilization 60.44%
 Executor count    60  (120%) estimated time 02m 08s and estimated cluster utilization 58.17%
 Executor count    75  (150%) estimated time 01m 57s and estimated cluster utilization 50.84%
 Executor count   100  (200%) estimated time 01m 50s and estimated cluster utilization 40.47%
 Executor count   150  (300%) estimated time 01m 50s and estimated cluster utilization 26.98%
 Executor count   200  (400%) estimated time 01m 50s and estimated cluster utilization 20.24%
 Executor count   250  (500%) estimated time 01m 50s and estimated cluster utilization 16.19%



Total tasks in all stages 1249
Per Stage  Utilization
Stage-ID   Wall    Task      Task     IO%    Input     Output    ----Shuffle-----    -WallClockTime-    --OneCoreComputeHours---   MaxTaskMem
          Clock%  Runtime%   Count                               Input  |  Output    Measured | Ideal   Available| Used%|Wasted%                                  
       0    1.00    0.91       512    0.0    0.0 KB    0.0 KB    0.0 KB    0.0 KB    00m 01s   00m 00s    00h 02m   45.7   54.3    0.0 KB 
       1    0.00    0.01         1    0.0   31.3 KB    0.0 KB    0.0 KB    0.0 KB    00m 00s   00m 00s    00h 00m    0.9   99.1    0.0 KB 
       2    0.00    0.01         1    0.0   64.0 KB    0.0 KB    0.0 KB    0.0 KB    00m 00s   00m 00s    00h 01m    0.9   99.1    0.0 KB 
       3    0.00    0.01         1    0.0   64.0 KB    0.0 KB    0.0 KB    0.0 KB    00m 00s   00m 00s    00h 01m    0.9   99.1    0.0 KB 
       4    0.00    0.01         1    0.0  443.7 KB    0.0 KB    0.0 KB    0.0 KB    00m 00s   00m 00s    00h 01m    0.9   99.1    0.0 KB 
       5    1.00    0.01         1    0.0    5.0 MB    0.0 KB    0.0 KB    0.0 KB    00m 01s   00m 00s    00h 02m    0.8   99.2    0.0 KB 
       6   81.00   91.87       512   95.4   58.3 GB    0.0 KB    0.0 KB  615.2 MB    01m 43s   01m 22s    02h 52m   79.3   20.7  130.0 MB 
       7    5.00    5.61       200    0.0    0.0 KB    0.0 KB  615.2 MB  735.5 MB    00m 06s   00m 05s    00h 10m   76.3   23.7  404.5 MB 
       8    8.00    1.58        20    0.0    0.0 KB  164.5 MB  735.5 MB    0.0 KB    00m 11s   00m 01s    00h 18m   12.4   87.6  469.3 MB 
Max memory which an executor could have taken = 469.3 MB


 Stage-ID WallClock  OneCore       Task   PRatio    -----Task------   OIRatio  |* ShuffleWrite% ReadFetch%   GC%  *|
          Stage%     ComputeHours  Count            Skew   StageSkew                                                
      0    1.39         00h 01m     512    5.12   244.20     0.69     0.00     |*   0.00           0.00    22.68  *|
      1    0.43         00h 00m       1    0.01     1.00     0.87     0.00     |*   0.00           0.00     0.00  *|
      2    0.48         00h 00m       1    0.01     1.00     0.89     0.00     |*   0.00           0.00     0.00  *|
      3    0.58         00h 00m       1    0.01     1.00     0.86     0.00     |*   0.00           0.00     0.00  *|
      4    0.53         00h 00m       1    0.01     1.00     0.86     0.00     |*   0.00           0.00     0.00  *|
      5    1.29         00h 00m       1    0.01     1.00     0.81     0.00     |*   0.00           0.00     0.00  *|
      6   81.24         02h 17m     512    5.12     4.59     0.66     0.01     |*   0.21           0.00     1.04  *|
      7    5.16         00h 08m     200    2.00     2.39     0.97     1.20     |*   0.86           3.58     8.14  *|
      8    8.91         00h 02m      20    0.20     1.67     0.99     0.22     |*   0.00           0.01     0.00  *|

PRatio:        Number of tasks in stage divided by number of cores. Represents degree of
               parallelism in the stage
TaskSkew:      Duration of largest task in stage divided by duration of median task.
               Represents degree of skew in the stage
TaskStageSkew: Duration of largest task in stage divided by total duration of the stage.
               Represents the impact of the largest task on stage time.
OIRatio:       Output to input ration. Total output of the stage (results + shuffle write)
               divided by total input (input data + shuffle read)

These metrics below represent distribution of time within the stage

ShuffleWrite:  Amount of time spent in shuffle writes across all tasks in the given
               stage as a percentage
ReadFetch:     Amount of time spent in shuffle read across all tasks in the given
               stage as a percentage
GC:            Amount of time spent in GC across all tasks in the given stage as a
               percentage

If the stage contributes large percentage to overall application time, we could look into
these metrics to check which part (Shuffle write, read fetch or GC is responsible)

```

我们重点看看几个在Spark UI上没有的东西：

## 2.1 Executor的数量

```
不同executor数量下的应用完成时间和集群利用率估算

 Real App Duration 02m 39s
 Model Estimation  02m 21s
 Model Error       11%

 NOTE: 1) 当启用自动扩展时，模型误差可能很大
       2) 模型不处理通过线程池运行的多个作业。为了更好地了解应用程序的可扩展性，请在没有线程池的情况下逐一尝试此类作业

       
 Executor count     5  ( 10%) estimated time 15m 31s and estimated cluster utilization 96.11%
 Executor count    10  ( 20%) estimated time 08m 04s and estimated cluster utilization 92.46%
 Executor count    25  ( 50%) estimated time 03m 43s and estimated cluster utilization 80.21%
 Executor count    40  ( 80%) estimated time 02m 42s and estimated cluster utilization 68.94%
 Executor count    50  (100%) estimated time 02m 21s and estimated cluster utilization 63.35%
 Executor count    55  (110%) estimated time 02m 14s and estimated cluster utilization 60.44%
 Executor count    60  (120%) estimated time 02m 08s and estimated cluster utilization 58.17%
 Executor count    75  (150%) estimated time 01m 57s and estimated cluster utilization 50.84%
 Executor count   100  (200%) estimated time 01m 50s and estimated cluster utilization 40.47%
 Executor count   150  (300%) estimated time 01m 50s and estimated cluster utilization 26.98%
 Executor count   200  (400%) estimated time 01m 50s and estimated cluster utilization 20.24%
 Executor count   250  (500%) estimated time 01m 50s and estimated cluster utilization 16.19%
```

以上Sparklens给出了，如果你申请N个Executor,你的Job的运行和资源使用效率情况。这个可以帮助我们更好的确定executor的申请数量。

> Executor count    50  (100%)，为当前申请的Executor数量下的情况

## 2.2 Executor Vcore数量

```
 OneCoreComputeHours:
 衡量集群中可用的总计算能力。executor中的一个核心，运行一个小时，算作一个OneCoreComputeHour。
 有4个核心的executor，与只有一个核心的executor相比，将有4倍的OneCoreComputeHours。
 同样，一个核心的executor运行4小时，其OnCoreComputeHours等于4个核心executor运行1小时

Driver利用率（由Driver导致的集群空闲）

 Total OneCoreComputeHours available                             04h 25m
 Total OneCoreComputeHours available (AutoScale Aware)           03h 59m
 OneCoreComputeHours wasted by driver                            00h 32m

 AutoScale Aware（自动缩放感知）:
 本工具的大多数计算都假定所有的executor在spark程序的整个运行时间内都是可用的。打印上面的数字是为了显示在解释效率指标时可能需要注意的事项 

 集群利用率 (Executors 由于缺少任务或偏斜而闲置 )

 Executor OneCoreComputeHours available                  03h 52m
 Executor OneCoreComputeHours used                       02h 29m        64.28%
 OneCoreComputeHours wasted                              01h 22m        35.72%

application层面的损耗指标 (Driver + Executor)

 OneCoreComputeHours wasted Driver               12.45%
 OneCoreComputeHours wasted Executor             31.28%
 OneCoreComputeHours wasted Total                43.72%

```

以上我们可以分析是否需要增加或者减少Vcore申请数量，来优化Spark job的效率

## 2.3 Spark是否高效  
这个高效不仅仅是你的资源使用情况，还可能和你本身Job的效率有关系

### 2.3.1 时间占比
```
Time spent in Driver vs Executors（在 Driver 和 Executors 上花费的时间）
 Driver WallClock Time    00m 19s   12.45%
 Executor WallClock Time  02m 19s   87.55%
 Total WallClock Time     02m 39s

```

这是Spark程序在Driver和Executor上的时间占比，很容易理解。
正常来说，Driver占用的时间应该特别少。但有时由于程序写的不好，就会出现Driver占用太多时间，甚至出现内存溢出的问题。
常见的几个原因如下:
- rdd.collect()
- sparkContext.broadcast
- spark.sql.autoBroadcastJoinThreshold 配置错误

Spraklens建议最大限度的减小driver的wall clock时间和占比。我们都知道, 如果你的driver在busy的情况下，executor就没有在做任何实质性的工作。如果你的driver busy了一分钟，而你又10个executor，那么你的spark job就相当于有10分钟的浪费。Sparklens建议通过把更多的执行process放到executor上去做，尽量避免和减少driver的busy。

### 2.3.2 3种情况下的时间预估（无限、无倾斜、单核）

```
Minimum possible time for the app based on the critical path (with infinite resources)   01m 50s
Minimum possible time for the app with same executors, perfect parallelism and zero skew 01m 49s
If we were to run this app with single executor and single core                          02h 29m
```
预估理想情况下应用程序需要的时间：<br>
在task的数量刚好等于cores的数量，并且没有数据倾斜时，估算一个运行时间，作为理想情况下应用程序需要的时间


## 2.4 数据倾斜
告诉我们每个stage是否有倾斜，图片为其他任务，数据倾斜情况

[![image](https://cdn.nlark.com/yuque/0/2021/png/8407327/1626503593618-08aa2f27-bc61-4534-b2e8-8168591b33b1.png)](https://s3.bmp.ovh/imgs/2022/09/08/514cd0e9f5686e9c.png)

Sparklens 报告在报告末尾提供了所提到的主要参数的详细定义，从而使其自给自足和清晰

|专有名词|解释说明|计算公式|
|---|---|---|
|PRatio|stage中的并行度|stage中的tasks `/` cores<br>PRatio > 1 => 任务数过多<br>PRatio < 1 => 核数过多|
|TaskSkew|stage中的倾斜度|最大task的持续时间 `/` 中值任务的持续时间|
|TaskStageSkew|最大任务对stage时间的影响|最大task的持续时间 `/` stage的总持续时间|
|OIRatio|stage中的输出到输入比率|该阶段的总输出(结果 + shuffle write) `/` 总输入(输入数据 + shuffle read)|

下面的这些指标代表stage内的时间分布

- ShuffleWrite：在给定阶段中所有任务的随机写入花费的时间量（以百分比表示）
- ReadFetch：在给定阶段中所有任务的随机读取时间花费的百分比
- GC：在给定阶段的所有任务中花费在 GC 上的时间量（以百分比表示） 

如果该stage占整个应用程序时间的比例很大，我们可以查看这些指标以确定是哪一部分（Shuffle write、read fetch 或 GC）影响

## 2.5 在线交互报告

上面所有的输出也可以在Sparklens UI中看到，生成的报告采用文本格式，其中包含上述所有指标和信息。

Qubole提供了[在线服务](http://sparklens.qubole.com/#upload) ，可从上传的JSON数据文件开始，生成具有交互式图表和表格的用户友好且优雅的报告