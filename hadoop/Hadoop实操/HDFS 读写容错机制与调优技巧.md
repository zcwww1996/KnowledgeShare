[TOC]

来源：https://mp.weixin.qq.com/s/C6UCs-VEZ8YCBt7WUvSJOA

# 1.HDFS 读写异常的容错机制

Hadoop 的设计理念就是部署在廉价的机器上，因此在容错方面做了周全的考虑，主要故障包括 DataNode 宕机，网络故障和数据损坏。本文介绍的容错机制只考虑 HDFS 读写异常场景。

## 1.1 读数据异常场景处理

[![HDFS读流程](https://i0.wp.com/i.loli.net/2021/10/08/WeOFohkiXQJCYq8.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_jpg/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0Ualu147fXuD9GibMJsaB7jMOk0kb5uL6LRST3I90N3ricE48SOHAv0cQ/640?wx_fmt=jpeg)

我们通过上图，简单回顾下 HDFS 的读数据流程，先打开文件，再调用`read()`方法读取数据。

在`read()`方法中，调用了`readWithStrategy()`方法读取`DataNode`存储的数据。

*   **readWithStrategy( )方法** - Hadoop3.0
    

```java
    protected synchronized int readWithStrategy(ReaderStrategy strategy)
            throws IOException {
        //检查dfsClient 是否已经关闭，如果关闭了，则抛出异常
        dfsClient.checkOpen();
        if (closed.get()) {
            throw new IOException("Stream closed");
        }

        //获取要读取的数据的长度
        int len = strategy.getTargetLength();
        //corruptedBlocks 用于保存损坏的数据块，使用CorruptedBlocks类封装
        //CorruptedBlocks 类，维护了一个Map<ExtendedBlock，Set<DatanodeInfo>>集合CorruptedBlocks 
        CorruptedBlocks corruptedBlocks = new CorruptedBlocks();
        failures = 0;
        if (pos < getFileLength()) { //判断读取位置是否在文件范围内，初始为0
            int retries = 2; //如果出现异常，则重试2次
            while (retries > 0) {
                try {
                    //pos 超过数据块边界，需要从新的数据块开始读取数据
                    if (pos > blockEnd || currentNode == null) {
                        //调用blockSeekTo（）方法获取保存这个数据块的一个数据节点
                        currentNode = blockSeekTo(pos);
                    }
                    //计算这次所要读取的长度
                    int realLen = (int) Math.min(len, (blockEnd - pos + 1L));
                    synchronized (infoLock) {
                        //判断最后一个数据块是否写完成
                        if (locatedBlocks.isLastBlockComplete()) {
                            realLen = (int) Math.min(realLen,
                                    locatedBlocks.getFileLength() - pos);
                        }
                    }
                    //调用 readBuffer 方法读取数据
                    int result = readBuffer(strategy, realLen, corruptedBlocks);

                    if (result >= 0) {
                        pos += result;//pos 移位
                    } else {
                        // got a EOS from reader though we expect more data on it.
                        throw new IOException("Unexpected EOS from the reader");
                    }
                    updateReadStatistics(readStatistics, result, blockReader);
                    dfsClient.updateFileSystemReadStats(blockReader.getNetworkDistance(),
                            result);
                    if (readStatistics.getBlockType() == BlockType.STRIPED) {
                        dfsClient.updateFileSystemECReadStats(result);
                    }
                    return result;
                } catch (ChecksumException ce) {
                    throw ce; //出现校验错误，则她出异常
                } catch (IOException e) {
                    checkInterrupted(e);
                    if (retries == 1) {
                        DFSClient.LOG.warn("DFS Read", e);
                    }
                    blockEnd = -1;
                    if (currentNode != null) {
                        addToDeadNodes(currentNode); //将当前失败的节点加入黑名单中
                    }
                    if (--retries == 0) { //重试超过两次，直接抛出异常
                        throw e;
                    }
                } finally {
                    //检查是否需要向 NameNode 汇报损坏的数据块
                    reportCheckSumFailure(corruptedBlocks,
                            getCurrentBlockLocationsLength(), false);
                }
            }
        }
        return -1;
    }
```


`readWithStrategy()`方法中使用 `pos` 控制数据的读取位置。

*   首先，调用 `blockSeekTo()`方法获取一个保存了目标数据块的 `DataNode`
    
*   其次，再调用 `readBuffer()` 方法从获取到的 `DataNode`中读取数据
    

从 `readWithStrategy()`方法实现可知，读失败场景比较简单。从上述代码 48 - 59 行可知，如果发生异常：

1.  先把当前的异常节点放入 `HashMap` 中，然后 `DFSIputStream` 会尝试重新去连接列表里的下一个 `DataNode` ，默认重试 2 次。
    
2.  同时，`DFSInputStream` 还会对获取到的数据进行 `checkSums` 核查，如果与存储在 `NameNode` 的校验和不一致，说明数据块损坏，`DFSInputStream` 就会试图从另一个拥有备份的 `DataNode` 中去读取数据。
    
3.  最后，`NameNode` 会去同步异常数据块。
    

## 1.2 写数据异常场景处理

[![写流程](https://i0.wp.com/i.loli.net/2021/10/08/thc7Vs5WXgyP68M.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0ibRa4cSGJzdNAlw2wSQsyUJ4EibCNKh5ajEJOI5YnXED6q5TgcPia7zfA/640?wx_fmt=png)

通过上图，简单回顾下写数据流程，客户端调用`creat()`方法，通过 RPC 与 `NameNode` 通信,请求创建一个文件，之后调用`write`,向数据流管道（Pipeline）写入数据。

写数据过程中以数据包（Packet）为单位，进行一个一个传输，数据包发送流程如下图：

[![数据包发送流程图](https://i0.wp.com/i.loli.net/2021/10/08/UWKfe4sTSwRbQzZ.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0fTrAwFjiag2BDRQWvy0ZUBcKRpO0a4RVIZK2PRHsicWRtYib6rCicicbSbQ/640?wx_fmt=png)

当构建完 DFSOutputStream 输出流时，客户端调用`write()`方法把数据包写入 dataQueue 队列，在将数据包发送到 DataNode 之前，`DataStreamer`会向 NameNode 申请分配一个新的数据块

然后，建立写这个数据块的数据流管道（pipeline），之后`DataStreamer` 会从 `dataQueue` 队列取出数据包，通过 pipeline 依次发送给各个 DataNode。

每个数据包（packet）都有对应的序列号，当一个数据块中所有的数据包都发送完毕，并且都得到了 ack 消息确认后，`PacketProcessor` 会将该数据包移出 `ackQueue`，之后，`Datastreamer`会将当前数据块的 pipeline 关闭。

通过不断循环上述过程，直到该文件（一个文件会被切分为多个`Block`）的所有数据块都写完成。在数据块发送过程中，如果某台`DataNode`宕机，`HDFS`主要会做一下容错：

1.  首先 `pipiline` 被关闭，`ackQueue`队列中等待确认的 `packet` 会被放入数据队列的起始位置不再发送，以防止在宕机的节点的下游节点再丢失数据。
    
2.  然后，存储在正常的 `DataNode` 上的 Block 块会被指定一个新的标识，并将该标识传递给 `NameNode` ,以便故障 `DataNode` 在恢复后，就可以删除自己存储的那部分已经失效的数据块。
    
3.  宕机节点会从数据流管道移除，剩下的 2 个好的 `DataNode` 会组成一个新的 `Pipeline` ,剩下的 `Block` 的数据包会继续写入新的 `pipeline` 中。
    
4.  最后，在写流程结束后，`NameNode` 发现节点宕机，导致部分 `Block` 块的备份数小于规定的备份数，此时 `NameNode` 会安排节点的备份数满足 `dfs.replication` 的配置要求。
    

以上过程，针对客户端是透明的。

# 2. HDFS 调优技巧

## 2.1 HDFS 小文件优化方法

`HDFS` 集群中 `NameNode` 负责存储数据块的元数据，其中包括**文件名、副本数、文件的BlockId、以及Block 所在的服务器**等信息，这个元数据的大小大约为 150 byte。

对于元数据来说，不管实际文件是大还是小，其大小始终在 150 byte 左右。如果 `HDFS` 存储了大量的小文件，会产生很多的元数据文件，这样便会导致`NameNode`内存消耗过大；此外，元数据文件过多，使得寻址时间大于数据读写时间，这样显得很低效。

*   数据源头解决：依赖于 `HDFS` 存储的数据写入前，通过 SequenceFile 将小文件或小批数据合成大文件再上传到 `HDFS` 上。
    
*   事后解决：使用 `Hadoop Archive` 归档命令，将已经存储在 `HDFS` 上的多个小文件打包成一个 HAR 文件，即将小文件合并成大文件。
    

## 2.2 存储优化

**纠删码存储**

Hadoop 3.X 之前，数据默认以 3 副本机制存储，这样虽然提高了数据的可靠性，但所带来的是 200% 的存储开销。对于 I/O 频率较低的冷热数据集，在正常操作期间很少访问额外的块副本，但仍然消耗与第一个副本相同的资源量。

因此，一个自然的改进就是使用纠删码代替复制，它**保证了相同级别的数据可靠性，存储空间却更少**。它是将一个文件拆分成一些数据单元和一些校验单元。具体数据单元与校验单元的配比是根据纠删码策略确定的。

*   **RS-3-2-1024K** : 使用 RS 编码，每 3 个数据单元，生成 2 个校验单元，共 5 个单元，也就是说，这 5 个单元中，只要有任意的 3 个单元存在（无论是数据单元还是校验单元），就可以通过计算得到原始数据。每个单元的大小是 1024k。
    

此外，还有**RS-10-4-1024k、RS-6-3-1024k、RS-LEGACY-6-3-1024k、XOR-2-1-1024k** 这四种策略。所有的纠删码策略，只能在目录上设置。

默认情况下，所有内建的 EC 策略是不可用的，可以通过以下两种方式开启对纠删码策略的支持：

1.  通过以下配置项，指定想要使用的 EC 策略。

[![EC策略](https://i0.wp.com/i.loli.net/2021/10/08/S21dOYEGeHRMgn8.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY05xKqKlUYsAc6PicGZbMjP9vmfbCGVjYjPdmbG4Lib1Ku5Aqh05AxDQ4g/640?wx_fmt=png)
    
2.  开发人员还可以基于集群规模和期望的容错性，使用命令行的方式：`hdfs ec -enablePolicy -policy [policyName]`，在集群上开启对某种纠删码策略的支持，`policyName`为纠删码策略名。
    
    之后，开发人员便可以通过 `hdfs ec -setPolicy -path [directoryName] -policy [policyName]` 命令，实现对某个文件目录指定使用上述开启的 EC 策略了。如果缺少 `[policyName]` 这个参数，集群会使用系统的默认值：**RS-6-3-1024k**。
    

**异构存储**

异构存储是另外一种存储优化，主要解决不同的数据，存储在不同类型的硬盘中，以获取最佳性能。

1.  存储类型：
    
    **RAM\_DISK**:存储在内存镜像文件系统；
    
    **SSD**：SSD 固态硬盘
    
    **DISK**:普通磁盘存储，在 `HDFS` 中，如果没有主动声明数据目录存储类型，默认都是 **DISK**
    
    **ARCHIVE**: 这个没有特指哪种存储介质，主要指的是计算能力比较弱而存储密度比较高的存储介质，用来解决数据量的容量扩增的问题，一般用于归档
    
2.  存储策略：
    

**Lazy\_Persist**:策略 ID 为15，它只有一个副本保存在内存中，其余副本都保存在磁盘中。

**ALL\_SSD**:策略 ID 为12，其所有副本数都存在固态硬盘中。

**One\_SSD**:策略 ID 为10，它有一个副本保存在固态，其余副本都保存在磁盘中

**HOT(default)**:策略 ID 为7，所有副本都保存在磁盘中，这是默认的存储策略。

**Warm**:策略 ID 为5，一个副本在磁盘，其余副本都保存在归档存储上。

**Cold**:策略 ID 为2，所有副本都保存在归档存储上。

## 2.3 HDFS 调优参数

1.  `NameNode` 内存生产配置
    
    `HADOOP_NAMENODE_OPTS=-Xmx102400m` ，Hadoop 3.x 中，其内存是动态分配的。
    
    **cloudera 给出的经验值**：`NameNode` 最小值 1G，每增加 100 万个 `block` 增加 1G 值；
    
    `DataNode` 最小值 4G，一个 `DataNode` 上的副本总数低于 400 万，调为 4G，超 400 万，每增加 100 万，增加 1G。
    
2.  `NameNode` 同时与 `DataNode` 通信的线程数，即心跳并发配置（hdfs-site.xml）

[![并发心跳](https://i0.wp.com/i.loli.net/2021/10/08/eKwxOjr7n1c53iY.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0oZzia6XA4fm5vHF8pBUx2Yp8qMUqayhwYPKTBxcibU1Kqk8WHZZPv0UQ/640?wx_fmt=png)

**企业经验**：`dfs.namenode.handler.count`\=20 × `𝑙𝑜𝑔 𝑒^𝐶𝑙𝑢𝑠𝑡𝑒𝑟 𝑆𝑖𝑧𝑒`，比如集群规模（DataNode 台 数）为 3 台时，此参数设置为 21
    
3.  `DataNode` 进行文件传输时最大线程数（hdfs-site.xml）

[![最大传输线程数](https://i0.wp.com/i.loli.net/2021/10/08/5KJDlqGu3PtaVF7.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY09iaeBNibU112uLBcrmHWt658ictIZwpEpibFlZFNsVW91IXN7zEQvHxQFQ/640?wx_fmt=png)
    
4.  `DataNode` 的最大连接数

[![最大连接数](https://i0.wp.com/i.loli.net/2021/10/08/vpxVYyOjPbaGRUA.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0CA41ibIb9AZhLCZh4v9dFTNA1ia57FOaY69HuZHw7fFkBDLYAG6l9icow/640?wx_fmt=png)
    
对于 DataNode 来说，**如同 Linux 上的文件句柄的限制**，当 DataNode上面的连接数超过配置中的设置时， DataNode就会拒绝连接，可以将其修改设置为65536。
    
5.  Hadoop 的缓冲区大小(core-site.xml)

[![缓冲区大小](https://i0.wp.com/i.loli.net/2021/10/08/sla8T2JVGCRYzdj.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY0JbZc6zumqtfcaXSyvAyWLt7NA1GKHiagEov4tg44gUfOGUC82wEfNoQ/640?wx_fmt=png)
    
6.  开启回收站工作机制（core-sit.xml）
 
    开启回收站功能，可以将删除的文件在不超时的情况下恢复，起到防止误操作。

[![hdfs回收站](https://i0.wp.com/i.loli.net/2021/10/08/aSLNBZpjbsUJClA.jpg)](https://mmbiz.qpic.cn/sz_mmbiz_png/2puJdtWcpQy4XMuEQdxZNvsCibbweZNY02vNMgVibDL1bypTVFFHk5rje3YUllxMg6b3NukQXJrUkqaibibYmGd2zw/640?wx_fmt=png)
    
> 注意：<br>
> 1. 如果是在网页删除的文件，不会进入回收站
> 2. 要求：`fs.trash.checkpoint.interval` <= `fs.trash.interval`
    

# 3. 高频面试题

1.  客户端在写 `DataNode` 的过程中，`DataNode` 宕机是否对写有影响？
    
2.  是否要完成所有目标 `DataNode` 节点的写入，才算一次写成功操作？
    
3.  读的过程如何保证数据没有被损坏，如果损坏如何处理？
    
4.  交互过程数据校验的基本单位是什么？
    
5.  数据写入各个 `DataNode` 时如何保证数据一致性？
    
6.  短路读机制
    
7.  短路读机制是如何保证文件的安全性的？