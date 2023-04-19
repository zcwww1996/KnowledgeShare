[TOC]

---

参考：[Spark 源码解析 : DAGScheduler中的DAG划分与提交](https://blog.csdn.net/zhouzx2010/article/details/51965196)

来源： https://www.cnblogs.com/arachis/p/spark_DAGScheduler.html

# 1. Spark任务执行的四个重要过程

1. 构建DAG,记录RDD的转换关系，就将记录了 RDD调用了什么方法，传入了什么函数，结束就是调用了 Action 
2. DAGScheduler切分Stage:按照shuffle进行切分的，如果有shuffle就切一刀（逻辑上的切分）会切分成多个Stage, —个Stege対应一个taskset (装着相同业务逻辑的Task集合，Taskset中Task的数量跟该Stage中的分区数据量一致） 
3. 将TaskSet中的Task调度到Executor中执行 
4. 在Executor中执行Taskset的业务逻辑

[![QwNfj1.md.png](https://s2.ax1x.com/2019/12/09/QwNfj1.md.png "spark任务执行的四个重要过程")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_iicfl.png)

# 2. DAG划分与提交概述

一些stage相关的知识点：

1. DAGScheduler将Job分解成具有前后依赖关系的多个stage
2. DAGScheduler是根据ShuffleDependency划分stage的
3. stage分为ShuffleMapStage和ResultStage；一个Job中包含一个ResultStage及多个ShuffleMapStage
4. 一个stage包含多个tasks，task的个数即该stage的finalRDD的partition数
5. 一个stage中的task完全相同，ShuffleMapStage包含的都是ShuffleMapTask；ResultStage包含的都是ResultTask

Stage有两种：
- ShuffleMapStage 
    1) 这种Stage是以Shuffle为<font style="background-color: yellow;">输出边界</font>
    2) 其输入边界可以是从外部获取数据，也可以是另一个ShuffleMapStage的输出
    3) 其输出可以是另一个Stage的开始
    4) ShuffleMapStage的最后Task就是ShuffleMapTask
    5) 在一个Job里可能有该类型的Stage，也可以能没有该类型Stage。
- ResultStage 
   1) 这种Stage是直接输出结果
   2) 其输入边界可以是从外部获取数据，也可以是另一个ShuffleMapStage的输出
   3) ResultStage的最后Task就是ResultTask
   4) 在一个Job里<font style="background-color: yellow;">肯定</font>有该类型Stage。


---

1、spark中 job可以理解为：Application中每一个action操作触发的作业。

2、spark中stage可以理解为：每一个job根据宽依赖划分的任务阶段。(stage的划分和stage的作业提交都通过Driver中DAGScheduler类实现的)

**DAG划分**<br>
这个DAG划分,底层是通过finalRDD里面的runjob触发DAGscheduler,递归通过栈结构遍历RDD寻找shuffle关系,切割stage,最终划分出正确的stage,然后递归后的回归的过程就是由前至后的stage提交过程

## 2.1 stage的划分：

1. DAGScheduler中的handleJobSubmitted方法根据最后一个Rdd（所谓最后一个Rdd指的是DAG图中触发action操作的Rdd）生成finalStage。

2. 生成finalStage的过程中调用了getParentStagesAndId方法，通过该方法，从最后一个Rdd开始向上遍历Rdd的依赖（可以理解为其父Rdd），如果遇到其依赖为shuffle过程，则调用getShuffleMapStage方法生成该shuffle过程所在的stage。完成Rdd遍历后，所有的stage划分完成。

3. getShuffleMapStage方法从传入的Rdd开始遍历，直到遍历到Rdd的依赖为shuffle为止，生成一个stage。

4. stage的划分就此结束。

## 2.2 stage的提交：

1. DAGScheduler中的submitStage方法用来提交stage，首先通过getMissingParentStages方法获取finalStage的父调度阶段。

2. 如果存在父调度阶段，则将该调度阶段存储在waitingStages列表中，同时递归调用submitStage。

3. 如果不存在父调度阶段（这里是递归出口）则调用submitMissingTasks方法执行该调度阶段。

4. 当一个调度阶段执行完成后，检查其任务是否全部完成，如果完成，则将waitingStage列表中父调度阶段完成的 调度阶段进行执行，直到所有调度阶段都执行完毕（递归出口）。

# 3. Spark DAGSheduler生成Stage过程


RDD.Action触发SparkContext.run，这里举最简单的例子rdd.count()


```scala
/**
 * Return the number of elements in the RDD.
 */
def count(): Long = sc.runJob(this, Utils.getIteratorSize _).sum
```

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_U35sK.thumb.700_0.png "触发runJob")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_U35sK.thumb.700_0.png)

Spark Action会触发SparkContext类的runJob，而runJob会继续调用DAGSchduler类的runJob

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_2RPwK.thumb.700_0.png "调用submitJob方法")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_2RPwK.thumb.700_0.png)

DAGSchduler类的runJob方法调用submitJob方法，并根据返回的completionFulture的value判断Job是否完成。


[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_yRW3d.thumb.700_0.png "onReceive方法")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_yRW3d.thumb.700_0.png)

onReceive用于DAGScheduler不断循环的处理事件，其中submitJob()会产生JobSubmitted事件,进而触发handleJobSubmitted方法。

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_LYdHX.thumb.700_0.png "handleJobSubmitted")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_LYdHX.thumb.700_0.png)

handleJobSubmitted方法首先创建ResultStage。正常情况下会根据finalStage创建一个ActiveJob。而finalStage就是由spark action对应的finalRDD生成的,而该stage要确认所有依赖的stage都执行完，才可以执行。也就是通过getMissingParentStages方法判断的。

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_uYRFz.thumb.700_0.jpeg "ActiveJob")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_uYRFz.thumb.700_0.jpeg)

这个方法用一个栈来实现递归的切分stage,然后返回一个宽依赖的HashSet，如果是宽依赖类型就会调用

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_duuil.thumb.700_0.jpeg "递归的切分stage")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_duuil.thumb.700_0.jpeg)

getMissingParentStages方法是由当前stage，返回他的父stage，父stage的创建由getShuffleMapStage返回，最终会调用newOrUsedShuffleStage方法返回ShuffleMapStage

之后提交stage,根据missingStage执行各个stage。划分DAG结束

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_JfVdm.thumb.700_0.png "提交stage,执行各个stage")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_JfVdm.thumb.700_0.png)

submitStage会依次执行这个DAG中的stage,如果有父stage就递归调用submitStage进行提交,否则就将当前Stage为起始stage，提交这个stage,加入watingstages中。

[![](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_t3uzi.thumb.700_0.jpeg "submitStage依次执行这个DAG中的stage")](https://c-ssl.duitang.com/uploads/item/201912/09/20191209150123_t3uzi.thumb.700_0.jpeg)


示例：
 

> scala> sc.makeRDD(Seq(1,2,3)).count
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.SparkContext:59] - Starting job: count at <console>:13
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Got job 0 (count at <console>:13) with 22 output partitions (allowLocal=false)
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Final stage: Stage 0(count at <console>:13)
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Parents of final stage: List()
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Missing parents: List()
> 
> 16/10/28 17:54:59 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Submitting Stage 0 (ParallelCollectionRDD[0] at makeRDD at <console>:13), which has no missing parents

 

> scala> sc.makeRDD(Seq(1,2,3)).map(l =>(l,1)).reduceByKey((v1,v2) => v1+v2).collect
> 16/10/28 18:00:07 [INFO] [org.apache.spark.SparkContext:59] - Starting job: collect at <console>:13
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Registering RDD 2 (map at <console>:13)
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Got job 1 (collect at <console>:13) with 22 output partitions (allowLocal=false)
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Final stage: Stage 2(collect at <console>:13)
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Parents of final stage: List(Stage 1)
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Missing parents: List(Stage 1)
> 16/10/28 18:00:07 [INFO] [org.apache.spark.scheduler.DAGScheduler:59] - Submitting Stage 1 (MappedRDD[2] at map at <console>:13), which has no missing parents
> 

**collect依赖于reduceByKey，reduceByKey依赖于map，而reduceByKey是一个Shuffle操作，故会先提交map (Stage 1 (MappedRDD[2] at map at <console>:13))**


# 4. Spark Master、Worker、Driver、Executor工作流程详解

![spark架构图.png](https://upload-images.jianshu.io/upload_images/16969231-5917173d48abb15a.png?imageMogr2/auto-orient/strip|imageView2/2/w/839)

## 4.1 角色介绍

Spark架构使用了分布式计算中master-slave模型，master是集群中含有master进程的节点，slave是集群中含有worker进程的节点。

- **Driver Program** ：运⾏main函数并且新建SparkContext的程序。
- **Application**：基于Spark的应用程序，包含了driver程序和集群上的executor。
  - **Cluster Manager**：指的是在集群上获取资源的外部服务。目前有三种类型
  - **Standalone**: spark原生的资源管理，由Master负责资源的分配
  - **Apache Mesos**:与hadoop MR兼容性良好的一种资源调度框架
 - **Hadoop Yarn**: 主要是指Yarn中的ResourceManager
- **Worker Node**： 集群中任何可以运行Application代码的节点，在Standalone模式中指的是通过slaves文件配置的Worker节点，在Spark on Yarn模式下就是NodeManager节点
- **Executor**：是在一个worker node上为某应⽤启动的⼀个进程，该进程负责运⾏行任务，并且负责将数据存在内存或者磁盘上。每个应⽤都有各自独立的executor。
- **Task** ：被送到某个executor上的工作单元。


在基于standalone的Spark集群，Cluster Manger就是Master。

### 4.1.1 Master

Master负责分配资源，在集群启动时，Driver向Master申请资源，Worker负责监控自己节点的内存和CPU等状况，并向Master汇报。

从资源方面，可以分为两个层面：<br/>
1）资源的管理和分配<br/>
资源的管理和分配，由Master和Worker来完成。Master给Worker分配资源,Master时刻知道Worker的资源状况。<br/>
客户端向服务器提交作业，实际是提交给Master。

2）资源的使用<br/>
资源的使用，由Driver和Executor。程序运行时候，向Master请求资源。

### 4.1.2 Driver

![spark-driver.jpg](https://upload-images.jianshu.io/upload_images/16969231-7cd53a962a9bc2cf.jpg?imageMogr2/auto-orient/strip|imageView2/2/w/440)

**Driver运行应用程序的时候，具有main方法，且创建了SparkContext，是整个程序运行的调度的核心**。内部会有高层调度器(DAGScheduler)和底层调度器(TaskScheduler)，高层调度器把整个作业(Job)划分成几个小的阶段(Stage)，底层调度器负责每个阶段内的任务(Task)该怎么去执行。由SchedulerBackend管理整个集群中为当前运行的应用程序分配的计算资源，分配的计算资源其实就是Executors，同时向Master注册当前的应用程序，如果注册成功，Master会向其分配资源。下一步根据action级别的操作出发job，job中会有一系列的RDD，从后往前推，如果是宽依赖的话就划分为不同的Stage，划分完成后提交给底层调度器TaskScheduler，TaskScheduler拿到具体的任务的集合，然后根据数据的本地性原则，把任务发放到对应的Executor上去执行，当Executor运行成功或者出现状况的话都会向Driver进行汇报，最后运行完成之后关闭SparkContext，所创建的对象也会随之关闭。

### 4.1.3 worker

主要功能：管理当前节点内存，CPU的使用状况，接收master分配过来的资源指令，通过ExecutorRunner启动程序分配任务，worker就类似于包工头，管理分配新进程，做计算的服务，相当于process服务。

需要注意的是：

 1) **worker不会汇报当前信息给master**，worker心跳给master主要只有workid，它不会发送资源信息以心跳的方式给mater，master分配的时候就知道worker，只有出现故障的时候才会发送资源。

 2) **worker不会运行代码**，具体运行的是Executor是可以运行具体appliaction写的业务逻辑代码，操作代码的节点，它不会运行程序的代码的。

### 4.1.4 executor

Executor是运行在Worker所在的节点上为当前应用程序而开启的进程里面的一个对象，这个对象负责了具体task的运行，具体是通过线程池并发执行和复用的方式实现的。

- 这里要补充一点：Hadoop的MR是运行在一个又一个JVM上，而JVM比较重量级且不能被复用，而Spark中是通过线程池并发执行和复用的方式执行tasks，极大的方便了迭代式计算，所以Spark的性能大大提高。每个task的计算逻辑一样，只是处理的数据不同而已。

- Spark应用程序的运行并不依赖于Cluster Manager，如果应用程序注册成功，Master就已经提前分配好了计算资源，运行的过程中跟本就不需要Cluster Manager的参与(可插拔的)，这种资源分配的方式是粗粒度的。

- 关于数据本地性，是在DAGScheduler划分Stage的时候确定的，TaskScheduler会把每个Stage内部的一系列Task发送给Executor，而具体发给哪个Executor就是根据数据本地性原则确定的，有关数据本地性的详细内容也会在后面的文章中进行说明


## 4.2 总结
从Driver和Worker的角度，也就是资源管理和分配的角度，来讲解Master、Worker、Driver、Executor工作流程。

程序运行的时候，Driver向Master申请资源；
Master让Worker给程序分配具体的Executor。

下面就是Driver具体的调用过程：

通过DAGScheduler划分阶段，形成一系列的TaskSet，然后传给TaskScheduler，把具体的Task交给Worker节点上的Executor的线程池处理。线程池中的线程工作，通过BlockManager来读写数据。

这就是4大组件：Worker、Master、Executor、Driver之间的协同工作。