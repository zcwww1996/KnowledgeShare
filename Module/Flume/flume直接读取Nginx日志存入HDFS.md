[TOC]
# 直接读取Nginx日志存入HDFS
## 1.1 配置

flume的配置文件在conf目录下如：/usr/local/flume/conf，当刚解压的时候会有一名flume-conf.properties.template的配置模版文件，需要将它以用下命令复制一份：
`cp flume-conf.properties.template flume-conf.properties`

然后按以下内容配置，根据自己的实际环境做相的修改：

注意：本示例仅用于从日志文件中按行读取后直接写入hdfs中。

```bash
    # the channels and the sinks.  
    # Sources, channels and sinks are defined per agent,  
    # in this case called 'agent' 
    
    #定义source名此片为source1
    agent1.sources = source1
    #定义channel名此片为channel1
    agent1.channels = channel1
    #定义sink名此片为sink1
    agent1.sinks = sink1                         
      
      
    # For each one of the sources, the type is defined
    #source的类型 exec
    agent1.sources.source1.type = exec  
    agent1.sources.source1.shell = /bin/bash -c  
    #要采集的日志文及命令 
    agent1.sources.source1.command = tail -n +0 -F /usr/local/nginx/logs/access.log  
    agent1.sources.source1.channels = channel1  
    agent1.sources.source1.threads = 5;  
      
    # The channel can be defined as follows.  
    agent1.channels.channel1.type = memory  
    agent1.channels.channel1.capacity = 100  
    agent1.channels.channel1.transactionCapacity = 100  
    agent1.channels.channel1.keep-alive = 30  
      
    # Each sink's type must be defined df
    //sink的类型：hdfs 
    agent1.sinks.sink1.type = hdfs  
      
    #Specify the channel the sink should use  
    agent1.sinks.sink1.channel = channel1
    #hdfs的api地址  
    agent1.sinks.sink1.hdfs.path = hdfs://192.168.89.29:9000/flume
    agent1.sinks.sink1.hdfs.writeFormat = Text  
    agent1.sinks.sink1.hdfs.fileType = DataStream  
    agent1.sinks.sink1.hdfs.rollInterval = 0  
    agent1.sinks.sink1.hdfs.rollSize = 100  
    agent1.sinks.sink1.hdfs.rollCount = 0  
    agent1.sinks.sink1.hdfs.batchSize = 100  
    agent1.sinks.sink1.hdfs.txnEventMax = 100  
    agent1.sinks.sink1.hdfs.callTimeout = 60000  
```

## 1.2 增加运行时需要的hadoop的jar包

hdfs是大数据分析平台hadoop文件的一个了系统，做为Client调用其API接口写进数据，flume依赖它的jar包，且这依赖的这些jar包并没有放在flume的发行包里面。
使用的方式用两种：
1. 是设置环境变量，把hadoop目录的相关jar目录加入到环境变量中；
2. 是把相关的jar文件复制到flume的jar目录下，如本例为：/usr/local/flume/lib。

本例采用的是方法二。hadoop的jar目录为：`/usr/local/hadoop/share/hadoop`

我采用的方法是启动后，根据报的错误一个一个的找到所依赖的jar包。总共如下几个：

```bash
hadoop-common-2.7.3.jar                 //hadoop/share/hadoop/common目录下
hadoop-auth-2.7.3.jar                   //hadoop/share/hadoop/common/lib
commons-configuration-1.6.jar           //hadoop/share/hadoop/common/lib
hadoop-hdfs-2.7.3.jar                   //hadoop/share/hadoop/hdfs目录下
hadoop-mapreduce-client-core-2.7.3.jar  //hadoop/share/hadoop/mapreduce
htrace-core-3.1.0-incubating.jar        //hadoop/share/hadoop/common/lib
commons-io.2.4.jar                      //hadoop/share/hadoop/common/lib
```

## 1.3 启动

启动命令：bin/flume-ng agent --conf conf --conf-file conf/flume-conf.properties --name agent1 -Dflume.root.logger=INFO,console

其中agent1 的要种配置文件中配置的名字一样。


## 1.4 碰到的问题

1. Using this sink requires hadoop to be installed so that Flume can use the Hadoop jars to communicate with the HDFS cluster. Note that a version of Hadoop that supports the sync() call is required.

大意就是：使用这种sink需要hadoop的jar包。由于也没说具体哪个jar包

当没有把hadoop的jar拷进来会报这个错。

2. org.apache.hadoop.security.AccessControlException: Permission denied: user=root, access=WRITE, inode="/flume/FlumeData.1482765009917.tmp":hadoop:supergroup:drwxr-xr-x

这个是hdfs没有开放权限给客户端写入，解决方法有如下：

解决方法：

到服务器上修改hadoop的配置文件：conf/hdfs-core.xml, 找到 dfs.permissions 的配置项 , 将value值改为 false

```xml
<property>
        <name>dfs.permissions</name>
        <value>false</value>
</property>
```
