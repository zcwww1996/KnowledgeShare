[TOC]


# 1. Flink 运行时的组件
Flink 运行时架构主要包括四个不同的组件，它们会在运行流处理应用程序时协同工作:
作业管理器(JobManager)、资源管理器(ResourceManager)、任务管理器(TaskManager)，
以及分发器(Dispatcher)。因为 Flink 是用 Java 和 Scala 实现的，所以所有组件都会运行在
Java 虚拟机上。每个组件的职责如下:


## 1.1 作业管理器(JobManager)
```bash
控制一个应用程序执行的主进程，也就是说，每个应用程序都会被一个不同的 JobManager 所控制执行。
JobManager 会先接收到要执行的应用程序，这个应用程序会包括: 作业图(JobGraph)、逻辑数据流图(logical dataflow graph)和打包了所有的类、库和其它 资源的 JAR 包。
JobManager 会把 JobGraph 转换成一个物理层面的数据流图，这个图被叫做 “执行图”(ExecutionGraph)，包含了所有可以并发执行的任务。JobManager 会向资源管理器(ResourceManager)请求执行任务必要的资源，也就是任务管理器(TaskManager)上 的插槽(slot)。一旦它获取到了足够的资源，就会将执行图分发到真正运行它们的 TaskManager 上。
而在运行过程中，JobManager 会负责所有需要中央协调的操作，比如说检 查点(checkpoints)的协调。
```


## 1.2 资源管理器(ResourceManager)
```bash
主要负责管理任务管理器(TaskManager)的插槽(slot)，TaskManger 插槽是 Flink 中 定义的处理资源单元。
Flink 为不同的环境和资源管理工具提供了不同资源管理器，比如 YARN、Mesos、K8s，以及 standalone 部署。
当 JobManager 申请插槽资源时，ResourceManager 会将有空闲插槽的TaskManager分配给JobManager。
如果ResourceManager没有足够的插槽 来满足 JobManager 的请求，它还可以向资源提供平台发起会话，
以提供启动 TaskManager 进程的容器。另外，ResourceManager 还负责终止空闲的 TaskManager，释放计算资源。
```


## 1.3 任务管理器(TaskManager)
```bash
Flink 中的工作进程。通常在 Flink 中会有多个 TaskManager 运行，每一个 TaskManager都包含了一定数量的插槽(slots)。插槽的数量限制了 TaskManager 能够执行的任务数量。
启动之后，TaskManager 会向资源管理器注册它的插槽收到资源管理器的指令后， TaskManager 就会将一个或者多个插槽提供给 JobManager 调用。
JobManager 就可以向插槽 分配任务(tasks)来执行了。在执行过程中，一个 TaskManager 可以跟其它运行同一应用程 序的 TaskManager 交换数据。
```


## 1.4 分发器(Dispatcher)
```bash
可以跨作业运行，它为应用提交提供了 REST 接口。当一个应用被提交执行时，分发器 就会启动并将应用移交给一个
JobManager。由于是 REST 接口，所以 Dispatcher 可以作为集 群的一个 HTTP 接入点，这样就能够不受防
火墙阻挡。Dispatcher 也会启动一个 Web UI，用 来方便地展示和监控作业执行的信息。Dispatcher 
在架构中可能并不是必需的，这取决于应 用提交运行的方式。
```

# 2. 任务提交流程
我们来看看当一个应用提交执行时，Flink 的各个组件是如何交互协作的:

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761472961-34da6dd3-9393-4cfb-9025-8653d387635a.png)

上图是从一个较为高层级的视角，来看应用中各组件的交互协作。如果部署的集群环境不同(例如 YARN，Mesos，Kubernetes，standalone 等)，其中一些步骤可以被省略，或是有些组件会运行在同一个 JVM 进程中。

具体地，如果我们将 Flink 集群部署到 YARN 上，那么就会有如下的提交流程:

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761524165-b23b219f-8f8f-4945-a4e3-ffb34d4a96cd.png)

Flink 任务提交后，Client 向 HDFS 上传 Flink 的 Jar 包和配置，之后向 YarnResourceManager 提交任务，ResourceManager 分配 Container 资源并通知对应的NodeManager 启动 ApplicationMaster

ApplicationMaster 启动后加载 Flink 的 Jar 包和配置构建环境，然后启动 JobManager，之后 ApplicationMaster 向 ResourceManager申请资源启动 TaskManager，ResourceManager 分配 Container 资源后，由ApplicationMaster 通知资源所在节点的 NodeManager 启动 TaskManager

NodeManager 加载 Flink 的 Jar 包和配置构建环境并启动 TaskManager，TaskManager启动后向 JobManager 发送心跳包，并等待 JobManager 向其分配任务。

# 3. 任务调度原理
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761701309-d0a05996-f661-4c63-a24f-f5837902e7cf.png)

客户端不是运行时和程序执行的一部分，但它用于准备并发送dataflow(JobGraph)给Master(JobManager)，然后，客户端断开连接或者维持连接以等待接收计算结果。

当 Flink 集群启动后，首先会启动一个 JobManger 和一个或多个的TaskManager。由 Client 提交任务给 JobManager，JobManager 再调度任务到各个TaskManager 去执行，然后 TaskManager 将心跳和统计信息汇报给 JobManager。

TaskManager 之间以流的形式进行数据的传输。上述三者均为独立的 JVM 进程。

**Client**为提交 Job 的客户端，可以是运行在任何机器上(与 JobManager 环境连通即可)。提交 Job 后，Client 可以结束进程(Streaming 的任务)，也可以不结束并等待结果返回。

**JobManager**主要负责调度 Job 并协调 Task 做 checkpoint，职责上很像Storm 的 Nimbus。从 Client 处接收到 Job 和 JAR 包等资源后，会生成优化后的执行计划，并以 Task 的单元调度到各个 TaskManager 去执行。

**TaskManager**在启动的时候就设置好了槽位数(Slot)，每个 slot 能启动一个Task，Task 为线程。从 JobManager 处接收需要部署的 Task，部署启动后，与自己的上游建立 Netty 连接，接收数据并处理。

## 3.1 TaskManger 与 Slots
Flink 中每一个 worker(TaskManager)都是一个 **JVM** 进程，它可能会在独立的线程上执行一个或多个 subtask。为了控制一个 worker 能接收多少个 task，worker 通过 task slot 来进行控制(一个 worker 至少有一个 task slot)。

每个 task slot 表示 TaskManager 拥有资源的一个固定大小的子集。假如一个TaskManager 有三个 slot，那么它会将其管理的内存分成三份给各个 slot。资源 slot化意味着一个 subtask 将不需要跟来自其他 job 的 subtask 竞争被管理的内存，取而代之的是它将拥有一定数量的内存储备。需要注意的是，这里不会涉及到 CPU 的隔离，slot 目前仅仅用来隔离 task 的受管理的内存。

通过调整 task slot 的数量，允许用户定义 subtask 之间如何互相隔离。

如果一个TaskManager一个slot，那将意味着每个task group运行在独立的JVM中(该JVM可能是通过一个特定的容器启动的)，而一个 TaskManager 多个 slot 意味着更多的subtask 可以共享同一个 JVM。

而在同一个 JVM 进程中的 task 将共享 TCP 连接(基于多路复用)和心跳消息。它们也可能共享数据集和数据结构，因此这减少了每个task 的负载。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761821532-eec490e2-160e-42c4-b319-4be4ca364c9f.png)

默认情况下，Flink 允许子任务共享 slot，即使它们是不同任务的子任务(前提是它们来自同一个 job)。 这样的结果是，一个 slot 可以保存作业的整个管道。

**Task Slot**是静态的概念，是指 **TaskManager**具有的并发执行能力，可以通过参数taskmanager.numberOfTaskSlots 进行配置;而并行度 **parallelism**是动态概念，即 **TaskManager**运行程序时实际使用的并发能力，可以通过参数 parallelism.default进行配置。


也就是说，假设一共有 3 个 TaskManager，每一个 TaskManager 中的分配 3 个TaskSlot，也就是每个 TaskManager 可以接收 3 个 task，一共 9 个 TaskSlot，如果我们设置 parallelism.default=1，即运行程序默认的并行度为 1，9 个 TaskSlot 只用了 1个，有 8个空闲，因此，设置合适的并行度才能高效率

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761928458-76adb598-a7da-4ae4-813a-fefc8dce3b2b.png)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581761958362-eb3ffa52-60e6-4a17-8464-de187164b45a.png)

## 3.2 程序与数据流(DataFlow)

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581762075663-09f7ccc0-2b21-4525-ae98-9ca047a1b094.png)

所有的 Flink 程序都是由三部分组成的: **Source**、**Transformation**和 **Sink**。

**Source 负责读取数据源**，**Transformation 利用各种算子进行处理加工，Sink负责输出**。


在运行时，Flink 上运行的程序会被映射成“逻辑数据流”(dataflows)，它包含了这三部分。每一个 **dataflow**以一个或多个 **sources**开始以一个或多个 **sinks**结束。dataflow 类似于任意的有向无环图(DAG)。在大部分情况下，程序中的转换运算(transformations)跟 dataflow 中的算子(operator)是一一对应的关系，但有时候，一个 transformation 可能对应多个 operator

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581762156659-0672f8f2-46d1-4cb3-a563-d32729f9e398.png)

## 3.3 执行图(ExecutionGraph)
由 Flink 程序直接映射成的数据流图是 StreamGraph，也被称为逻辑流图，因为它们表示的是计算逻辑的高级视图。为了执行一个流处理程序，Flink 需要将逻辑流图转换为物理数据流图(也叫执行图)，详细说明程序的执行方式。

Flink 中的执行图可以分成四层：StreamGraph -> JobGraph -> ExecutionGraph -> 物理执行图。

**StreamGraph**：是根据用户通过 Stream API 编写的代码生成的最初的图。用来
表示程序的拓扑结构。


**JobGraph**：StreamGraph 经过优化后生成了 JobGraph，提交给 JobManager 的数据结构。主要的优化为，将多个符合条件的节点 chain 在一起作为一个节点，这样可以减少数据在节点之间流动所需要的序列化/反序列化/传输消耗。


**ExecutionGraph**: JobManager 根 据 JobGraph 生 成 ExecutionGraph 。ExecutionGraph 是 JobGraph 的并行化版本，是调度层最核心的数据结构。


**物理执行图**：JobManager 根据 ExecutionGraph 对 Job 进行调度后，在各个TaskManager 上部署 Task 后形成的“图”，并不是一个具体的数据结构

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581762346936-52ffcb8f-de21-491f-af8f-e4efa96f4cac.png)

## 3.4 并行度(Parallelism)
Flink 程序的执行具有并行、分布式的特性

在执行过程中，一个流(stream)包含一个或多个分区(stream partition)，而每一个算子(operator)可以包含一个或多个子任务(operator subtask)，这些子任务在不同的线程、不同的物理机或不同的容器中彼此互不依赖地执行。

**一个特定算子的子任务(subtask)的个数被称之为其并行度(parallelism)。**

一般情况下，一个流程序的并行度，可以认为就是其所有算子中最大的并行度。一个程序中，不同的算子可能具有不同的并行度。 

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581762420190-7985b2a5-5a34-428c-ae35-df2e613f0a34.png)

Stream 在算子之间传输数据的形式可以是 one-to-one(forwarding)的模式也可以是 redistributing 的模式，具体是哪一种形式，取决于算子的种类。


**One-to-one**：stream(比如在 source 和 map operator 之间)维护着分区以及元素的顺序。那意味着 map 算子的子任务看到的元素的个数以及顺序跟 source 算子的子任务生产的元素的个数、顺序相同，map、fliter、flatMap 等算子都是 one-to-one 的对应关系，类似于 spark 中的窄依赖

**Redistributing**：stream(map()跟 keyBy/window 之间或者 keyBy/window 跟 sink之间)的分区会发生改变。每一个算子的子任务依据所选择的 transformation 发送数据到不同的目标任务。<br>
例如，keyBy() 基于 hashCode 重分区、broadcast 和 rebalance会随机重新分区，这些算子都会引起 redistribute 过程，而 redistribute 过程就类似于Spark 中的 shuffle 过程。类似于 spark 中的宽依赖

## 3.5 任务链(Operator Chains)
相同并行度的 **one to one**操作，Flink 这样相连的算子链接在一起形成一个 task，原来的算子成为里面的一部分。将算子链接成 task 是非常有效的优化:它能减少线程之间的切换和基于缓存区的数据交换，在减少时延的同时提升吞吐量。链接的行为可以在编程 API 中进行指定。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581762511376-dfde2eaa-ee71-462b-b10d-ca493bcb6d12.png)