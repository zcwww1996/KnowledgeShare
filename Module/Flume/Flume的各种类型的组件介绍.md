[TOC]

# 1.Source

## NetCat Source
绑定的端口（tcp、udp），将流经端口的每一个文本行数据作为Event输入；

**type**：source的类型，必须是netcat。

**bind**：要监听的(本机的)主机名或者ip。此监听不是过滤发送方。一台电脑不是说只有一个IP。有多网卡的电脑，对应多个IP。

**port**：绑定的本地的端口。

## Avro Source
监听一个avro服务端口，采集Avro数据序列化后的数据；

**type**：avrosource的类型，必须是avro。

**bind**：要监听的(本机的)主机名或者ip。此监听不是过滤发送方。一台电脑不是说只有一个IP。有多网卡的电脑，对应多个IP。

**port**：绑定的本地的端口。

## Exec Source
于Unix的command在标准输出上采集数据；

**type**:source的类型：必须是exec。

**command**：要执行命令。

## Spooling Directory Source
监听一个文件夹里的文件的新增，如果有则采集作为source。

**type**：source 的类型：必须是spooldir

**spoolDir**：监听的文件夹 【提前创建目录】

**fileSuffix**：上传完毕后文件的重命名后缀，默认为.COMPLETED

**deletePolicy**：上传后的文件的删除策略never和immediate，默认为never。

**fileHeader**：是否要加上该文件的绝对路径在header里，默认是false。

**basenameHeader**：是否要加上该文件的名称在header里，默认是false。

![](https://images2017.cnblogs.com/blog/610238/201710/610238-20171007165626458-550900744.png)

# 2.Sink

## HDFS Sink
将数据传输到hdfs集群中。

**type**：sink的类型 必须是hdfs。

**hdfs.path**：hdfs的上传路径。

**hdfs.filePrefix**：hdfs文件的前缀。默认是:FlumeData

**hdfs.rollInterval**:间隔多久产生新文件，默认是:30（秒） 0表示不以时间间隔为准。

**hdfs.rollSize**：文件到达多大再产生一个新文件，默认是:1024（bytes）0表示不以文件大小为准。

**hdfs.rollCount**：event达到多大再产生一个新文件，默认是:10（个）0表示不以event数目为准。

**hdfs.batchSize**：每次往hdfs里提交多少个event，默认为100

**hdfs.fileType**：hdfs文件的格式主要包括：SequenceFile, DataStream ,CompressedStream，如果使用了CompressedStream就要设置压缩方式。

**hdfs.codeC**：压缩方式：gzip, bzip2, lzo, lzop, snappy

注：%{host}可以使用header的key。以及%Y%m%d来表示时间，但关于时间的表示需要在header里有timestamp这个key。

## Logger Sink
将数据作为日志处理（根据flume中的设置的日志方式来显示）

要在控制台显示在运行agent的时候加入：-Dflume.root.logger=INFO,console 。

**type**：sink的类型：必须是 logger。

**maxBytesToLog**：打印body的最长的字节数 默认为16

## Avro Sink
数据被转换成Avro Event，然后发送到指定的服务端口上。

**type**：sink的类型：必须是 avro。

**hostname**：指定发送数据的主机名或者ip

**port**：指定发送数据的端口

## File Roll Sink
数据发送到本地文件。

type：sink的类型：必须是 file_roll。

sink.directory：存储文件的目录【提前创建目录】

batchSize：一次发送多少个event。默认为100

sink.rollInterval：多久产生一个新文件，默认为30s。单位是s。0为不产生新文件。【即使没有数据也会产生文件】

# 3.Channel

type 选择memory 时Channel的性能最好，但是如果flume进程意外挂掉可能会丢失数据。type选择file时Channel的容错性更好，但是性能上会比memory channel差。

使用file Channel时dataDirs配置多个不同盘下的目录可以提高性能。

capacity 参数决定Channel可容纳最大的event条数。

transactionCapacity 参数决定每次Source往channel里面写的最大event条数和每次Sink从channel里面读的最大event条数。**<font color="red">transactionCapacity需要大于Source和Sink的batchSize参数。</font>**
**<font color="green">capacity >= transactionCapacity >= batchsize</font>**
## Memory Channel
使用内存作为数据的存储。

**Type channel**的类型：必须为memory

**capacity：channel**中的最大event数目

**transactionCapacity**：channel中允许事务的最大event数目

## File Channel
使用文件作为数据的存储

**Type channel**的类型：必须为 file

**checkpointDir** ：检查点的数据存储目录【提前创建目录】

**dataDirs** ：数据的存储目录【提前创建目录】

**transactionCapacity**：channel中允许事务的最大event数目

## Spillable Memory Channel
使用内存作为channel超过了阀值就存在文件中

**Type channel**的类型：必须为SPILLABLEMEMORY

**memoryCapacity**：内存的容量event数

**overflowCapacity**：数据存到文件的event阀值数

**checkpointDir**：检查点的数据存储目录

**dataDirs**：数据的存储目录

# 4. Interceptor

## Timestamp Interceptor
时间戳拦截器 在header里加入key为timestamp，value为当前时间。

**type**：拦截器的类型，必须为timestamp

**preserveExisting**：如果此拦截器增加的key已经存在，如果这个值设置为true则保持原来的值，否则覆盖原来的值。默认为false

## Host Interceptor
主机名或者ip拦截器，在header里加入ip或者主机名

**type**：拦截器的类型，必须为host

**preserveExisting**：如果此拦截器增加的key已经存在，如果这个值设置为true则保持原来的值，否则覆盖原来的值。默认为false

**useIP**：如果设置为true则使用ip地址，否则使用主机名，默认为true

**hostHeader**：使用的header的key名字，默认为host

## Static Interceptor
静态拦截器，是在header里加入固定的key和value。

**type：avrosource**的类型，必须是static。

**preserveExisting**:如果此拦截器增加的key已经存在，如果这个值设置为true则保持原来的值，否则覆盖原来的值。默认为false

**key**:静态拦截器添加的key的名字

**value**:静态拦截器添加的key对应的value值

# 5.Channel Selector

## Multiplexing Channel Selector
根据header的key的值分配channel

**selector.type** 默认为replicating

**selector.header**：选择作为判断的key

**selector.default**：默认的channel配置

**selector.mapping.\***：匹配到的channel的配置

# 6.Sink Processor

## 负载均衡


```properties
a1.sinkgroups=g1

a1.sinkgroups.g1.sinks=k1 k2

a1.sinkgroups.g1.processor.type=load_balance

a1.sinkgroups.g1.processor.backoff=true

a1.sinkgroups.g1.processor.selector=round_robin

a1.sinkgroups.g1.processor.selector.maxTimeOut=30000
```


backoff：开启后，故障的节点会列入黑名单，过一定时间再次发送，如果还失败，则等待是指数增长；直到达到最大的时间。

如果不开启，故障的节点每次都会被重试。

selector.maxTimeOut：最大的黑名单时间（单位为毫秒）。

## 故障转移


```properties
a1.sinkgroups=g1

a1.sinkgroups.g1.sinks=k1 k2

a1.sinkgroups.g1.processor.type=failover

a1.sinkgroups.g1.processor.priority.k1=10

a1.sinkgroups.g1.processor.priority.k2=5

a1.sinkgroups.g1.processor.maxpenalty=10000
```
 maxpenalty 对于故障的节点最大的黑名单时间 (in millis 毫秒)
