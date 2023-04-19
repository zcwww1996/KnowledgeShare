[TOC]
# Flume安装
**前提是已有hadoop环境**
1. 上传安装包到数据源所在节点上
2. 解压  tar -zxvf apache-flume-1.6.0-bin.tar.gz
3. flume/conf下的flume-env.sh，在里面配置JAVA_HOME

# Flume采集

## netcat-logger.conf

1、vi netcat-logger.conf

```java
# 定义这个agent中各组件的名字
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# 描述和配置source组件：r1
a1.sources.r1.type = netcat
a1.sources.r1.bind = localhost
a1.sources.r1.port = 44444

# 描述和配置sink组件：k1
a1.sinks.k1.type = logger

# 描述和配置channel组件，此处使用是内存缓存的方式
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100

# 描述和配置source  channel   sink之间的连接关系
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```
2、启动agent去采集数据
**bin/flume-ng <font color="red">agent</font> <font color="orange">-c conf</font> <font color="blue">-f conf/netcat-logger.conf</font> <font color="purple">-n a1</font> <font color="green">-Dflume.root.logger=INFO,console</font>**

- **<font color="red">agent</font>**  启动agent客户端

- **<font color="orange">-c conf</font>**  指定flume自身的配置文件所在目录

- **<font color="blue">-f conf/netcat-logger.conf</font>**   指定我们所描述的采集方案

- **<font color="purple">-n a1</font>**   指定我们这个agent的名字

- **<font color="green">-Dflume.root.logger=INFO,console</font>**   指定日志级别为INFO，打印到控制台

## 采集目录到HDFS

spooldir特性：
   1、监视一个目录，只要目录中出现新文件，就会采集文件中的内容
   2、采集完成的文件，会被agent自动添加一个后缀：COMPLETED
   3、所监视的目录中不允许重复出现相同文件名的文件
   

**注意：不能往监控目中重复丢同名文件，否则会报错，报错后需要重启flume-ng**

启动命令：  
bin/flume-ng agent -c ./conf -f ./conf/spool-logger.conf -n a1 -Dflume.root.logger=INFO,console

测试： 往/home/hadoop/flumeSpool放文件（mv ././xxxFile /home/hadoop/flumeSpool），但是不要在里面生成文件

```java
#定义三大组件的名称
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1

# 配置source组件
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /home/hadoop/logs/
agent1.sources.source1.fileHeader = false

#配置拦截器
agent1.sources.source1.interceptors = i1
agent1.sources.source1.interceptors.i1.type = host
agent1.sources.source1.interceptors.i1.hostHeader = hostname

# 配置sink组件
agent1.sinks.sink1.type = hdfs
agent1.sinks.sink1.hdfs.path =hdfs://hdp-node-01:9000/weblog/flume-collection/%y-%m-%d/%H-%M
agent1.sinks.sink1.hdfs.filePrefix = access_log
agent1.sinks.sink1.hdfs.maxOpenFiles = 5000
agent1.sinks.sink1.hdfs.batchSize= 100
agent1.sinks.sink1.hdfs.fileType = DataStream
agent1.sinks.sink1.hdfs.writeFormat =Text
agent1.sinks.sink1.hdfs.rollSize = 102400
agent1.sinks.sink1.hdfs.rollCount = 1000000
agent1.sinks.sink1.hdfs.rollInterval = 60
#agent1.sinks.sink1.hdfs.round = true
#agent1.sinks.sink1.hdfs.roundValue = 10
#agent1.sinks.sink1.hdfs.roundUnit = minute
agent1.sinks.sink1.hdfs.useLocalTimeStamp = true
# Use a channel which buffers events in memory
agent1.channels.channel1.type = memory
agent1.channels.channel1.keep-alive = 120
agent1.channels.channel1.capacity = 500000
agent1.channels.channel1.transactionCapacity = 600

# Bind the source and sink to the channel
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

Channel参数解释：
capacity：默认该通道中最大的可以存储的event数量
trasactionCapacity：每次最大可以从source中拿到或者送到sink中的event数量
keep-alive：event添加到通道中或者移出的允许时间

## 采集文件到HDFS
采集需求：比如业务系统使用log4j生成的日志，日志内容不断增加，需要把追加到日志文件中的数据**<font style="background-color: yellow;">实时</font>**采集到hdfs

根据需求，首先定义以下3大要素
1、采集源，即source——监控文件内容更新 :  exec  ‘tail -F file’
2、下沉目标，即sink——HDFS文件系统  :  hdfs sink
3、Source和sink之间的传递通道——channel，可用file channel 也可以用 内存channel
```java
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1

# Describe/configure tail -F source1
agent1.sources.source1.type = exec
agent1.sources.source1.command = tail -F /home/hadoop/logs/access_log
agent1.sources.source1.channels = channel1

#configure host for source
agent1.sources.source1.interceptors = i1
agent1.sources.source1.interceptors.i1.type = host
agent1.sources.source1.interceptors.i1.hostHeader = hostname

# Describe sink1
agent1.sinks.sink1.type = hdfs
#a1.sinks.k1.channel = c1
agent1.sinks.sink1.hdfs.path =hdfs://hdp-node-01:9000/weblog/flume-collection/%y-%m-%d/%H-%M
agent1.sinks.sink1.hdfs.filePrefix = access_log
agent1.sinks.sink1.hdfs.maxOpenFiles = 5000
agent1.sinks.sink1.hdfs.batchSize= 100
agent1.sinks.sink1.hdfs.fileType = DataStream
agent1.sinks.sink1.hdfs.writeFormat =Text
agent1.sinks.sink1.hdfs.rollSize = 102400
agent1.sinks.sink1.hdfs.rollCount = 1000000
agent1.sinks.sink1.hdfs.rollInterval = 60
agent1.sinks.sink1.hdfs.round = true
agent1.sinks.sink1.hdfs.roundValue = 10
agent1.sinks.sink1.hdfs.roundUnit = minute
agent1.sinks.sink1.hdfs.useLocalTimeStamp = true

# Use a channel which buffers events in memory
agent1.channels.channel1.type = memory
agent1.channels.channel1.keep-alive = 120
agent1.channels.channel1.capacity = 500000
agent1.channels.channel1.transactionCapacity = 600

# Bind the source and sink to the channel
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```

## 采集目录到kafka（数据均匀下沉各分区）

由于采集的数据没有key，会随机发往同一个分区
使用了**数据倾斜，防止数据采集到同一个分区**

vi spool-kafka.conf

```java
#定义三大组件的名称
agent1.sources = source1
agent1.sinks = sink1
agent1.channels = channel1

# 配置source组件
agent1.sources.source1.type = spooldir
agent1.sources.source1.spoolDir = /root/cmcc/logs/
agent1.sources.source1.fileHeader = false

#配置拦截器
agent1.sources.source1.interceptors = i1
agent1.sources.source1.interceptors.i1.type = host
agent1.sources.source1.interceptors.i1.hostHeader = hostname

#使flume的log均匀的下沉到各个分区
agent1.sources.source1.interceptors = i2
agent1.sources.source1.interceptors.i2.type=org.apache.flume.sink.solr.morphline.UUIDInterceptor$Builder
agent1.sources.source1.interceptors.i2.headerName=key
agent1.sources.source1.interceptors.i2.preserveExisting=false

# 配置sink组件
agent1.sinks.sink1.type = org.apache.flume.sink.kafka.KafkaSink
agent1.sinks.sink1.topic = cmccTopic
agent1.sinks.sink1.brokerList = linux01:9092,linux02:9092,linux03:9092
agent1.sinks.sink1.requiredAcks = 1
agent1.sinks.sink1.batchSize = 100

# Use a channel which buffers events in memory
agent1.channels.channel1.type = memory
agent1.channels.channel1.keep-alive = 120
agent1.channels.channel1.capacity = 10000
agent1.channels.channel1.transactionCapacity = 150

# Bind the source and sink to the channel
agent1.sources.source1.channels = channel1
agent1.sinks.sink1.channel = channel1
```