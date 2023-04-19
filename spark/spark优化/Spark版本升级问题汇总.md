
**起因**：部门准备将数据仓库开发工具从Hive SQL大规模迁移至Spark SQL。此前集群已经自带了Spark-1.5.2，系HDP-2.3.4自带的Spark组件，现在需要将之升级到目前的最新版本（2.2.1）。作为一个提供给第三方使用的开发工具，应该避免第三方过度浪费时间于工具本身的使用（为SQL任务调试合理的资源分配），故需要引入spark的DRA机制(Dynamic Resource Allocation)，实现spark任务的资源动态分配。

**集群环境：HDP-2.3.4, Hadoop-2.7.1，Hive-1.2.1**

从官网下载spark-2.2.1版本：[http://spark.apache.org/downloads.html](https://link.jianshu.com/?t=http%3A%2F%2Fspark.apache.org%2Fdownloads.html)  
这里需要额外说明，spark-2.2.1编译依赖的Hive版本为1.2.1，如果与集群所使用版本不符，需要自己根据Hive版本重新编译。

1.  拷贝至部署服务器指定目录：/usr/custom/，解压并重命名为：spark-2.2.1；
2.  在/usr/custom/目录下创建软链接：spark -> /usr/custom/spark-2.2.1

```
ln -s /usr/custom/spark-2.2.1 /usr/custom/spark 
```

备注：第三方开发工具引用地址：/usr/custom/spark，后续spark版本更新迭代与第三方开发工具无感知。

1.  进入目录 /usr/custom/spark-2.2.1/conf/，修改spark-env.sh：

```
mv  spark-env.sh.template  spark-env.sh
vim spark-env.sh 
```

添加如下内容（HDP安装目录：/usr/hdp/current/）：

```
export HADOOP_HOME=/usr/hdp/current/hadoop-client
export HADOOP_CONF_DIR=/usr/hdp/current/hadoop-client/conf
export YARN_CONF_DIR=/usr/hdp/current/hadoop-yarn-client/etc/hadoop
export HADOOP_YARN_HOME=/usr/hdp/current/hadoop-yarn-client
export HIVE_CONF_DIR=/usr/hdp/current/hive-client/conf
export HIVE_HOME=/usr/hdp/current/hive-client

export JAVA_HOME=/usr/local/jdk1.8.0_111
export SPARK_LOG_DIR=/var/log/spark2
export SPARK_CONF_DIR=/usr/custom/spark/conf 
```

2.  进入目录 /usr/custom/spark-2.2.1/conf/，修改spark-defaults.conf：

```
mv spark-defaults.conf.template spark-defaults.conf
vim spark-defaults.conf 
```

添加如下内容:

```
spark.eventLog.enabled   true
spark.eventLog.dir       hdfs://${clusterName}/tmp/spark-events
spark.eventLog.compress  true

spark.ui.enabled         true
spark.ui.killEnabled     false
spark.ui.port            18080

spark.history.ui.port              18080
spark.history.fs.cleaner.enabled   true
spark.history.fs.logDirectory      hdfs://${clusterName}/tmp/spark-events
spark.history.fs.cleaner.interval  1d
spark.history.fs.cleaner.maxAge    3d

spark.yarn.appMasterEnv.JAVA_HOME=/usr/local/jdk1.8.0_111
spark.executorEnv.JAVA_HOME=/usr/local/jdk1.8.0_111
spark.debug.maxToStringFields=50 
```

**【填坑一】Ranger依赖**  
如果Hive依托于Ranger进行权限管理，那么这里就需要对spark-defaults.conf额外添加：

```
spark.yarn.dist.files=/usr/custom/spark/conf/hive-site.xml,/usr/custom/spark/conf/ranger-hdfs-audit.xml,/usr/custom/spark/conf/ranger-hdfs-security.xml 
```

从hive-client/conf下拷贝hive-site.xml到/usr/custom/spark-2.2.1/conf，并删除其中关于ranger的相关配置，ranger不支持spark的权限控制。然而，spark SQL不可避免需要读取Hive metastore元数据库，这一步仍然需要Ranger的验证，故需要拷贝ranger-hdfs-audit.xml和ranger-hdfs-security.xml到/usr/custom/spark-2.2.1/conf目录下。设置这三个配置文件spark启动时自动加载（spark.yarn.dist.files)。

3.创建目录: **/var/log/spark2**，并开放所有用户可读，yarn history server日志保存路径；创建hdfs目录：**/tmp/spark-events**，并开放所有用户可读可写， spark application日志保存路径；

4.在部署服务器上启动：/usr/custom/spark/sbin/start-history-server.sh，登陆**http://${hostname}:18080/**验证。

引入DRA机制需要yarn和hadoop底层支持，故需要增加以下配置：  
a. 进入ambari管理页面：**Yarn -> Configs -> Advanced**  
**Node Manager**：

```
yarn.nodemanager.aux-services : mapreduce_shuffle,spark_shuffle 
```

**Custom yarn-site**:

```
yarn.nodemanager.aux-services.spark_shuffle.class : org.apache.spark.network.yarn.YarnShuffleService 
```

b. 添加spark-shuffle.jar支持：  
shuffle jar路径：/usr/custom/spark/yarn/spark-2.2.1-yarn-shuffle.jar

在集群每一台NodeManager节点上做如下配置：

1.  将shuffle的jar拷贝到/usr/cbas/spark2/lib/目录下，如果新加机器没有该目录，需要手动创建;
    
2.  在**/usr/cbas/spark2/lib/**目录下创建软连接：**spark-yarn-shuffle.jar -> /usr/cbas/spark2/lib/spark-2.2.1-yarn-shuffle.jar**
    

```
ln -s /usr/cbas/spark2/lib/spark-2.2.1-yarn-shuffle.jar spark-yarn-shuffle.jar 
```

3.  在**/usr/hdp/2.3.4.7-4/hadoop-yarn/lib/**目录下创建软连接：**spark-yarn-shuffle.jar -> /usr/cbas/spark2/lib/spark-yarn-shuffle.jar**

```
ln -s /usr/cbas/spark2/lib/spark-yarn-shuffle.jar spark-yarn-shuffle.jar 
```

(两重软连接是为了以后如果需要升级spark，更换spark-yarn-shuffle.jar不需要重启NodeManager)

3.  重启所有的NodeManager服务；

在/usr/custom/spark/conf/spark-defaults.conf文件中添加如下配置：

```
 spark.driver.extraLibraryPath=/ur/hdp/current/hadoop-client/lib/native
spark.executor.extraLibraryPath=/usr/hdp/current/hadoop-client/lib/native

spark.dynamicAllocation.enabled=true                          
spark.shuffle.service.enabled=true                            
spark.dynamicAllocation.schedulerBacklogTimeout=3s            
spark.dynamicAllocation.sustainedSchedulerBacklogTimeout=3s   
spark.dynamicAllocation.executorIdleTimeout=30s               
#spark.dynamicAllocation.cachedExecutorIdleTimeout=infinity   
spark.dynamicAllocation.initialExecutors=3                    
spark.dynamicAllocation.maxExecutors=200                      
spark.dynamicAllocation.minExecutors=0                        

spark.yarn.driver.memoryOverhead=1024                         
spark.yarn.executor.memoryOverhead=1024                       

spark.driver.memory=3g                                        
spark.executor.memory=3g                                      
spark.executor.cores=2                                        
spark.serializer=org.apache.spark.serializer.KryoSerializer 
```

以上配置，spark application提交一个任务时：

```
初始分配资源： driver.memory (3g)+ driver.memoryOverhead (1g) + ( executor.memory (3g) + executor.memoryOverhead (1g) ) * 3  = 16g； 
允许分配的最大资源： driver.memory (3g) + driver.memoryOverhead (1g) + ( executor.memory (3g) + executor.memoryOverhead (1g)) * 200  = 804g； 
```

spark配置优先级：**SparkConf core代码 > spark-submit --选项 > spark-defaults.conf配置 > spark-env.sh配置 > 默认值**  
因此，spark-defaults.conf里面设置的配置参数可以被(SparkConf core代码 && spark-submit --选项 )覆盖，对于逻辑复杂的大任务可以指定不开启动态资源分配**（spark.dynamicAllocation.enabled=false）**；

**【填坑二】 jersey版本冲突之一**  
报错信息类似如下：

```
Exception in thread "main" java.lang.NoClassDefFoundError: com/sun/jersey/api/client/config/ClientConfig
        at org.apache.hadoop.yarn.client.api.TimelineClient.createTimelineClient(TimelineClient.java:45)
        at org.apache.hadoop.yarn.client.api.impl.YarnClientImpl.serviceInit(YarnClientImpl.java:163)
        at org.apache.hadoop.service.AbstractService.init(AbstractService.java:163)
        at org.apache.spark.deploy.yarn.Client.submitApplication(Client.scala:150)
        at org.apache.spark.scheduler.cluster.YarnClientSchedulerBackend.start(YarnClientSchedulerBackend.scala:56)
        at org.apache.spark.scheduler.TaskSchedulerImpl.start(TaskSchedulerImpl.scala:149)
        at org.apache.spark.SparkContext.<init>(SparkContext.scala:500)
        at org.apache.spark.SparkContext$.getOrCreate(SparkContext.scala:2256)
        at org.apache.spark.sql.SparkSession$Builder$$anonfun$8.apply(SparkSession.scala:831)
        at org.apache.spark.sql.SparkSession$Builder$$anonfun$8.apply(SparkSession.scala:823)
        at scala.Option.getOrElse(Option.scala:121)
        at org.apache.spark.sql.SparkSession$Builder.getOrCreate(SparkSession.scala:823)
        at org.apache.spark.sql.hive.thriftserver.SparkSQLEnv$.init(SparkSQLEnv.scala:57)
        at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver.<init>(SparkSQLCLIDriver.scala:288)
        at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver$.main(SparkSQLCLIDriver.scala:137)
        at org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver.main(SparkSQLCLIDriver.scala)
        at sun.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
        at sun.reflect.NativeMethodAccessorImpl.invoke(NativeMethodAccessorImpl.java:57)
        at sun.reflect.DelegatingMethodAccessorImpl.invoke(DelegatingMethodAccessorImpl.java:43)
        at java.lang.reflect.Method.invoke(Method.java:606)
        at org.apache.spark.deploy.SparkSubmit$.org$apache$spark$deploy$SparkSubmit$$runMain(SparkSubmit.scala:729)
        at org.apache.spark.deploy.SparkSubmit$.doRunMain$1(SparkSubmit.scala:185)
        at org.apache.spark.deploy.SparkSubmit$.submit(SparkSubmit.scala:210)
        at org.apache.spark.deploy.SparkSubmit$.main(SparkSubmit.scala:124)
        at org.apache.spark.deploy.SparkSubmit.main(SparkSubmit.scala)
Caused by: java.lang.ClassNotFoundException: com.sun.jersey.api.client.config.ClientConfig
...... 
```

**原因**：Hadoop-2.x依赖jersey-1.9，而Spark-2.x依赖jersey-2.22，两个版本发生过重大重构，代码包路径已发生变更。  
jersey-1.9： _com.sun.jersey_  
jersey-2.22：_org.glassfish.jersey_

**解决方案**：将jersey-core-1.9.jar 和 jersey-client-1.9.jar拷贝至/usr/custom/spark/jars/目录下，再重命名避免加载器加载版本冲突问题。

**【填坑三】 jersey版本冲突之二**  
报错信息类似如下：

```
java.lang.NoSuchMethodError: javax.ws.rs.core.Application.getProperties()Ljava/util/Map;
    org.glassfish.jersey.server.ApplicationHandler.<init>(ApplicationHandler.java:331)
    org.glassfish.jersey.servlet.WebComponent.<init>(WebComponent.java:392)
    org.glassfish.jersey.servlet.ServletContainer.init(ServletContainer.java:177)
    org.glassfish.jersey.servlet.ServletContainer.init(ServletContainer.java:369) 
```

**原因**：当用户在Spark history server上查看application的运行情况时，发现history server无法读取日志，报错信息如上，其本质上也是jersey版本冲突导致的。

在jersey-core-1.9.jar的源代码里，自己集成了javax.ws.rs相关的代码。而在jersey-2.x版本，已经将该部分代码剔除，转而依赖javax.ws.rs-api-2.0.1.jar (javax.ws.rs)。在jersey-core-1.9.jar里是没有javax.ws.rs.core.Application.getProperties()这个方法。

**解决方法一**：在github上下载jersey-core-1.9.jar的源码，自行添加javax.ws.rs.core.Application.getProperties()，并重新编译打包；  
**解决方法二**：在github上下载jersey-core-1.9.jar的源码，将所有javax.ws.rs相关代码剔除，转而依赖javax.ws.rs-api-2.0.1.jar；

**【填坑四】 Spark SQL无法insert/overwrite无分区表**  
报错信息类似如下(找不到对应参数的loadTable方法)：

```
NoSuchMethodException:org.apche.hadoop.,hive.ql.metada.Hive. loadTable(org.apche.hadoop.fs.Path,java.lang.String,boolean,boolean) 
```

**原因**：  
hive SQL自带loadTable方法：

```
public void loadTable(Path loadPath, String tableName, LoadFileType loadFileType, boolean isSrcLocal,
   boolean isSkewedStoreAsSubdir, boolean isAcid, boolean hasFollowingStatsTask,
   Long txnId, int stmtId, boolean isMnTable) throws HiveException { 
```

spark SQL自带loadTable方法：

```
public void loadTable(Path loadPath, String tableName, boolean replace, boolean holdDDLTime, boolean isSrcLocal,
   boolean isSkewedStoreAsSubdir, boolean isAcid) throws HiveException {
      List<Path> newFiles = new ArrayList<Path>();
      Table tbl = this.getTable(tableName);
      HiveConf setssionConf = SessionState.getSessionConf(); 
```

spark SQL 读取 hive表的元数据时，需要与 hive metastore 进行交互。spark实例化HiveMetastoreClient时需要依赖spark.sql.hive.metastore.jars参数指向的jars目录；  
之前的做法是将spark.sql.hive.metastore.jars直接指向/usr/hdp/current/hive-client/lib/\*, 用hive自带的jars依赖来连接hive metastore，导致spark进行insert/overwrite操作时报错方法不存在；

**解决方案**：将spark2连接hive需要加载的hive相关的jars(hive-\*.spark2.jar，/usr/custom/spark/jars目录下) ，与/usr/hdp/current/hive-client/lib/目录下的jars，剔除hive-client中跟spark自带hive-\*系列冲突的jars，多次测试解决spark自带hive-\*系列需要依赖的其它jars，重新整合成一份新的jars目录(假设路径存在于：/usr/cbas/spark2/hive-jars/)。  
在spark-defaults.conf中添加如下配置：

```
spark.sql.hive.metastore.jars=/usr/cbas/spark2/hive-jars/* 
```

**【填坑五】 LZO压缩**

```
Caused by: java.lang.IllegalArgumentException: Compression codec com.hadoop.compression.lzo.LzoCodec not found.
        at org.apache.hadoop.io.compress.CompressionCodecFactory.getCodecClasses(CompressionCodecFactory.java:135)
        at org.apache.hadoop.io.compress.CompressionCodecFactory.<init>(CompressionCodecFactory.java:175)
        at org.apache.hadoop.mapred.TextInputFormat.configure(TextInputFormat.java:45) 
```

**原因**：LZO压缩算法的jar不存在  
**解决方法**：拷贝Hadoop自带的lzo jar(hadoop-lzo-0.6.0.2.3.4.7-4.jar)到/usr/custom/spark/jars/目录下。

**【填坑六】 spark-sql无法加载Hive UDF的jar**

```
/usr/custom/spark/bin/spark-sql  --deploy-mode client
add jar hdfs://${clusterName}/user/hive/udf/udf.jar 
```

报错信息如下：

```
java.lang.ExceptionInInitializerError
at org.apache.hadoop.hdfs.DFSInputStream.blockSeekTo(DFSInputStream.java:662)
at org.apache.hadoop.hdfs.DFSInputStream.readWithStrategy(DFSInputStream.java:889)
at org.apache.hadoop.hdfs.DFSInputStream.read(DFSInputStream.java:947)
at java.io.DataInputStream.read(DataInputStream.java:100)
......
Caused by: java.lang.NullPointerException
at org.apache.hadoop.hdfs.BlockReaderFactory.getRemoteBlockReaderFromTcp(BlockReaderFactory.java:746)
at org.apache.hadoop.hdfs.BlockReaderFactory.build(BlockReaderFactory.java:376)
at org.apache.hadoop.hdfs.DFSInputStream.blockSeekTo(DFSInputStream.java:662)
at org.apache.hadoop.hdfs.DFSInputStream.readWithStrategy(DFSInputStream.java:889)
at org.apache.hadoop.hdfs.DFSInputStream.read(DFSInputStream.java:947)
...... 
```

**解决方法**：  
在/usr/custom/spark/conf/spark-defaults.conf文件中添加如下配置：`spark.jars=hdfs://cbasNA/user/hive/udf/udf.jar`

**结束语**：spark SQL处理多shuffle任务的性能，较于Hive的高效毋庸置疑。然而在正式生产环境想要大规模的应用，仍然需要对底层源码的大量重构，比如：

1.  spark SQL目前只支持client模式；
2.  ~如果输入是大量小于一个逻辑块的小文件，会导致RDD partition过多(默认TextInputFormat，输入时无法对小文件合并)；~ 参考：[https://issues.apache.org/jira/browse/SPARK-13664](https://link.jianshu.com/?t=https%3A%2F%2Fissues.apache.org%2Fjira%2Fbrowse%2FSPARK-13664)
3.  当任务逻辑复杂需要加大spark.sql.shuffle.partitions，而结果数据普遍较小时，会产生大量的小文件；

```
修改源代码，在spark SQL执行计划后面添加重分区操作：
spark.sql.result.repartitions.enabled=true   =>  启用用户自定义输出文件个数功能，默认不启用
spark.sql.result.repartitions.num=10          =>  设置输出文件个数，默认为3 
```
