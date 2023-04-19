[TOC]

参考：https://ververica.cn/developers/advanced-tutorial-2-checkpoint-application-practice

# Checkpoint 与 state 的关系

Checkpoint 是从 source 触发到下游所有节点完成的一次全局操作。下图可以有一个对 Checkpoint 的直观感受，红框里面可以看到一共触发了 569K 次 Checkpoint，然后全部都成功完成，没有 fail 的。

![](https://ververica.cn/wp-content/uploads/2019/12/1-1024x296.png)

**state 其实就是 Checkpoint 所做的主要持久化备份的主要数据**，看下图的具体数据统计，其 state 也就 9kb 大小 。

![](https://ververica.cn/wp-content/uploads/2019/12/2-1024x369.png)

# 什么是 state

我们接下来看什么是 state。先看一个非常经典的 word count 代码，这段代码会去监控本地的 9000 端口的数据并对网络端口输入进行词频统计，我们本地行动 netcat，然后在终端输入 hello world，执行程序会输出什么？

![](https://ververica.cn/wp-content/uploads/2019/12/3-1024x237.png)

答案很明显，`**(hello, 1)**` 和 `**(word,1)**`

那么问题来了，如果再次在终端输入 hello world，程序会输入什么？

答案其实也很明显，`**(hello, 2)**` 和 `**(world, 2)**`。为什么 Flink 知道之前已经处理过一次 hello world，这就是 state 发挥作用了，这里是被称为 keyed state 存储了之前需要统计的数据，所以帮助 Flink 知道 hello 和 world 分别出现过一次。

回顾一下刚才这段 word count 代码。keyby 接口的调用会创建 keyed stream 对 key 进行划分，这是使用 keyed state 的前提。在此之后，sum 方法会调用内置的 StreamGroupedReduce 实现。

![](https://ververica.cn/wp-content/uploads/2019/12/4-1024x540.png)

## 什么是 keyed state

对于 keyed state，有两个特点：

*   只能应用于 KeyedStream 的函数与操作中，例如 Keyed UDF, window state
    
*   keyed state 是已经分区/划分好的，每一个 key 只能属于某一个 keyed state
    

对于如何理解已经分区的概念，我们需要看一下 keyby 的语义，大家可以看到下图左边有三个并发，右边也是三个并发，左边的词进来之后，通过 keyby 会进行相应的分发。例如对于 hello word，hello 这个词通过 hash 运算永远只会到右下方并发的 task 上面去。

![](https://ververica.cn/wp-content/uploads/2019/12/5-1024x546.png)

## 什么是 operator state

*   又称为 non-keyed state，每一个 operator state 都仅与一个 operator 的实例绑定。
    
*   常见的 operator state 是 source state，例如记录当前 source 的 offset
    

再看一段使用 operator state 的 word count 代码：

![](https://ververica.cn/wp-content/uploads/2019/12/6-1024x435.png)

这里的**`fromElements`**会调用**`FromElementsFunction`**的类，其中就使用了类型为 list state 的 operator state。根据 state 类型做一个分类如下图：

![](https://ververica.cn/wp-content/uploads/2019/12/7-1024x501.png)

除了从这种分类的角度，还有一种分类的角度是从 Flink 是否直接接管：

*   Managed State：由 Flink 管理的 state，刚才举例的所有 state 均是 managed state
    
*   Raw State：Flink 仅提供 stream 可以进行存储数据，对 Flink 而言 raw state 只是一些 bytes
    

在实际生产中，都只推荐使用 managed state，本文将围绕该话题进行讨论。

# 如何在 Flink 中使用 state

下图就前文 word count 的 sum 所使用的`**StreamGroupedReduce**`类为例讲解了如何在代码中使用 keyed state：

![](https://ververica.cn/wp-content/uploads/2019/12/8-1024x571.png)

下图则对 word count 示例中的`**FromElementsFunction**`类进行详解并分享如何在代码中使用 operator state：

![](https://ververica.cn/wp-content/uploads/2019/12/9.png)

# Checkpoint 的执行机制

在介绍 Checkpoint 的执行机制前，我们需要了解一下 state 的存储，因为 state 是 Checkpoint 进行持久化备份的主要角色。

## Statebackend 的分类

下图阐释了目前 Flink 内置的三类 state backend，其中`**MemoryStateBackend**`和`**FsStateBackend**`在运行时都是存储在 java heap 中的，只有在执行 Checkpoint 时，`**FsStateBackend**`才会将数据以文件格式持久化到远程存储上。

而`**RocksDBStateBackend**`则借用了 RocksDB（内存磁盘混合的 LSM DB）对 state 进行存储。

![](https://ververica.cn/wp-content/uploads/2019/12/10.png)

对于**`HeapKeyedStateBackend`**，有两种实现：

*   支持异步 Checkpoint（默认）：存储格式 CopyOnWriteStateMap
    
*   仅支持同步 Checkpoint：存储格式 NestedStateMap
    

特别在 MemoryStateBackend 内使用**`HeapKeyedStateBackend`**时，Checkpoint 序列化数据阶段默认有最大 5 MB数据的限制

对于**`RocksDBKeyedStateBackend`**，每个 state 都存储在一个单独的 column family 内，其中 keyGroup，Key 和 Namespace 进行序列化存储在 DB 作为 key。

![](https://ververica.cn/wp-content/uploads/2019/12/11.png)

## Checkpoint 执行机制详解

本小节将对 Checkpoint 的执行流程逐步拆解进行讲解，下图左侧是 Checkpoint Coordinator，是整个 Checkpoint 的发起者，中间是由两个 source，一个 sink 组成的 Flink 作业，最右侧的是持久化存储，在大部分用户场景中对应 HDFS。

**a. 第一步，Checkpoint Coordinator 向所有 source 节点 trigger Checkpoint；。**

![](https://ververica.cn/wp-content/uploads/2019/12/12.png)

**b. 第二步，source 节点向下游广播 barrier，这个 barrier 就是实现 Chandy-Lamport 分布式快照算法的核心，下游的 task 只有收到所有 input 的 barrier 才会执行相应的 Checkpoint。**

![](https://ververica.cn/wp-content/uploads/2019/12/13-1024x383.png)

**c. 第三步，当 task 完成 state 备份后，会将备份数据的地址（state handle）通知给 Checkpoint coordinator。**

![](https://ververica.cn/wp-content/uploads/2019/12/14-1024x440.png)

**d. 第四步，下游的 sink 节点收集齐上游两个 input 的 barrier 之后，会执行本地快照，这里特地展示了 RocksDB incremental Checkpoint 的流程，首先 RocksDB 会全量刷数据到磁盘上（红色大三角表示），然后 Flink 框架会从中选择没有上传的文件进行持久化备份（紫色小三角）。**

![](https://ververica.cn/wp-content/uploads/2019/12/15-1024x382.png)

**e. 同样的，sink 节点在完成自己的 Checkpoint 之后，会将 state handle 返回通知 Coordinator。**

![](https://ververica.cn/wp-content/uploads/2019/12/16.png)

**f. 最后，当 Checkpoint coordinator 收集齐所有 task 的 state handle，就认为这一次的 Checkpoint 全局完成了，向持久化存储中再备份一个 Checkpoint meta 文件。**

![](https://ververica.cn/wp-content/uploads/2019/12/17.png)

## Checkpoint 的 EXACTLY\_ONCE 语义

为了实现 EXACTLY ONCE 语义，Flink 通过一个 input buffer 将在对齐阶段收到的数据缓存起来，等对齐完成之后再进行处理。而对于 AT LEAST ONCE 语义，无需缓存收集到的数据，会对后续直接处理，所以导致 restore 时，数据可能会被多次处理。下图是官网文档里面就 Checkpoint align 的示意图：

![](https://ververica.cn/wp-content/uploads/2019/12/18-1024x249.png)

需要特别注意的是，Flink 的 Checkpoint 机制只能保证 Flink 的计算过程可以做到 EXACTLY ONCE，端到端的 EXACTLY ONCE 需要 source 和 sink 支持。

## Savepoint 与 Checkpoint 的区别

作业恢复时，二者均可以使用，主要区别如下：

<table><tbody><tr><td><strong>Savepoint</strong></td><td><strong>Externalized Checkpoint</strong></td></tr><tr><td>用户通过命令触发，由用户管理其创建与删除</td><td>Checkpoint 完成时，在用户给定的外部持久化存储保存</td></tr><tr><td>标准化格式存储，允许作业升级或者配置变更</td><td>当作业 FAILED（或者CANCELED）时，外部存储的 Checkpoint 会保留下来</td></tr><tr><td>用户在恢复时需要提供用于恢复作业状态的 savepoint 路径</td><td>用户在恢复时需要提供用于恢复的作业状态的 Checkpoint 路径</td></tr></tbody></table>