[TOC]

# Spark on Yarn的运行原理

来源： https://blog.csdn.net/u013573813/article/details/69831344

## 一、YARN是集群的资源管理系统

1. ResourceManager：负责整个集群的资源管理和分配。

2. ApplicationMaster：YARN中每个Application对应一个AM进程，负责与RM协商获取资源，获取资源后告诉NodeManager为其分配并启动Container。

3. NodeManager：每个节点的资源和任务管理器，负责启动/停止Container，并监视资源使用情况。

4. Container：YARN中的抽象资源。

## 二、SPARK的概念

1. Driver：和ClusterManager通信，进行资源申请、任务分配并监督其运行状况等。

2. ClusterManager：这里指YARN。

3. DAGScheduler：把spark作业转换成Stage的DAG图。

4. TaskScheduler：把Task分配给具体的Executor。

## 三、SPARK on YARN

### 3.1 yarn-cluster模式下

[![yarn-cluster](https://ae01.alicdn.com/kf/H290ef137cb3b472abf6386e9291ad73fI.png "yarn-cluster")](https://ftp.bmp.ovh/imgs/2020/05/44bc45cd6556ccd5.png)

1) ResourceManager接到请求后在集群中选择一个NodeManager分配Container，并在Container中启动ApplicationMaster进程；

2) 在ApplicationMaster进程中初始化sparkContext；

3) ApplicationMaster向ResourceManager申请到Container后，通知NodeManager在获得的Container中启动excutor进程；

4) sparkContext分配Task给excutor，excutor发送运行状态给ApplicationMaster。

### 3.2 yarn-client模式下

[![yarn-client](https://ae01.alicdn.com/kf/H40d03b20edd64851b805637ba773be7fb.jpg "yarn-client")](https://s1.ax1x.com/2020/05/29/tnngyQ.png)

1) ResourceManager接到请求后在集群中选择一个NodeManager分配Container，并在Container中启动ApplicationMaster进程；

2) driver进程运行在client中，并初始化sparkContext；

3) sparkContext初始化完后与ApplicationMaster通讯，通过ApplicationMaster向ResourceManager申请Container，ApplicationMaster通知NodeManager在获得的Container中启动excutor进程；

4) sparkContext分配Task给excutor，excutor发送运行状态给driver。

### 3.3 yarn-cluster与yarn-client的区别：

它们的区别就是ApplicationMaster的区别，yarn-cluster中ApplicationMaster不仅负责申请资源，并负责监控Task的运行状况，因此可以关掉client；
而yarn-client中ApplicationMaster仅负责申请资源，由client中的driver来监控调度Task的运行，因此不能关掉client。

## 四、Spark on Yarn 日志

来源： https://blog.csdn.net/u011878191/article/details/45894167

## 4.1 终端查看log

一直以来都是在UI界面上查看Spark日志的，在终端里面查看某个job的日志该怎么看呢？今天特地查了下资料，找到如下命令：

**1、查看某个job的日志**

```bash
yarn logs -applicationId application_1503298640770_0230
```

**2、查看某个job的状态**

```bash
yarn application -status application_1503298640770_0230
```

**3、kill掉某个job**

直接在UI界面或者是终端kill掉任务都是不对的，该任务可能还会继续执行下去，所以要用如下命令才算完全停止该job的执行

```bash
yarn application -kill application_1503298640770_0230
```

### 4.2 log位置

#### 4.2.1 spark的应用程序运行结果输出日志在yarn和standalone模式下<font color="red">位置不一样</font>

1) 在yarn模式下，日志输出位置在**NodeManager**节点(**ResourceManager**节点下**没输出**)的hadoop安装目录/logs/userlogs/目录下

2) Standalone模式，对应的日志输出在NodeManager节点的spark安装目录 **/work/** 目录下。可以通过参数**SPARK_WORKER_DIR**修改。

#### 4.2.2 应用程序的日志输出没了

`yarn.nodemanager.log.retain-seconds`参数

这个参数指的是应用程序输出日志保存的时间，默认是10800，单位是s，也就是3个小时。<font color="orange">**这个参数在没有开启日志聚合的时候有效**</font>。

#### 4.2.3 日志聚合功能

yarn资源管理器模式提供了日志聚合功能，通过参数yarn.log-aggregation-enable来开启，这个参数默认是false。如果开启了，你可以在yarn模式下在命令行中使用yarn logs -applicationId 来查看你的应用程序的运行日志。但必须保证:

1. 开启了该功能
2. 程序必须运行完

因为yarn要进行聚合。另外，如果开启了日志聚合，本地的日志文件就会删除，从而腾出更多空间。


# Spark on Yarn的配置流程

来源：https://blog.csdn.net/qq_23330633/article/details/52216155

## 1. 前期环境

### 1.1 JDK配置

```bash
# tar xvzf jdk-7u45-linux-x64.tar.gz -C/usr/local 
# cd /usr/local 
# ln -s jdk1.7.0_45 java
```

vim /etc/profile 加入以下内容

```bash
export JAVA_HOME=/usr/local/java   
export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib   
export PATH=$PATH:$JAVA_HOME/bin
```

<font color="red">**source /etc/profile**</font>

### 1.2 Scala安装

```bash
# tar xvzf scala-2.10.3.tgz -C/usr/local 

# cd /usr/local 

# ln -s scala-2.10.3 scala
```

vim /etc/profile 加入以下内容


```bash
export SCALA_HOME=/usr/local/scala   
export PATH=$PATH:$SCALA_HOME/bin   
```

<font color="red">**source /etc/profile**</font>

### 1.3 SSH免登录配置

1. ssh-keygen

在node1下生成的密钥对：id_rsa和id_rsa.pub，默认存储在"~/.ssh"目录下，包括两个文件，id_rsa和id_rsa.pub，分别为私钥和公钥

2. 将公钥写入信任文件中

```bash
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```


3. 然后修改authorized_keys文件的权限

```bash
chmod 644 ~/.ssh/authorized_keys
```

4. node1中的authorized_keys拷贝至其余节点的~/.ssh目录下

5. 重启SSH服务

### 1.4 主机名设置

1) vim /etc/sysconfig/network

```bash
HOSTNAME=linux01/linux02/linux03
```

2) vim /etc/hosts

```bash
192.168.134.101   linux01  
192.168.134.102   linux02  
192.168.134.103   linux03 
```

### 1.5 Zookeeper安装

#### 1.5.1 建立日志及存储目录

```bash
mkdir –p /data/hadoop/zookeeper/{data,logs}
```

<font color="red">**两个文件夹都需要预先建立好，否则会运行时会报错**</font>

#### 1.5.2 修改配置文件

vim /usr/local/zookeeper/conf/zoo.cfg

```bash
tickTime=2000
initLimit=10
syncLimit=5
 
dataDir=/data/hadoop/zookeeper/data
clientPort=2181
 
server.1=192.168.134.101:2888:3888
server.2=192.168.134.102:2888:3888
server.3=192.168.134.103:2888:3888
```

#### 1.5.3 建立myid文件

在`/data/hadoop/zookeeper/data`下分别建立名为myid文件，文件内容为上述zoo.cfg中IP地址对应`server.[number]`中的number

```bash
node1 : echo 1 > /data/hadoop/zookeeper/data/myid
node2 : echo 2 > /data/hadoop/zookeeper/data/myid
node3 : echo 3 > /data/hadoop/zookeeper/data/myid
```

#### 1.5.4 启动zookeeper

分别在linux01，linux02，linux03 执行`zkServer.sh  start`，然后通过`zkServer.sh status`查看状态，如果发现每个node当前状态标记为follower或者leader，那么测试通过

## 2 集群部署

### 2.1 Hadoop（HDFS HA）集群部署

#### 2.1.1 修改配置文件

##### 2.1.1.1 vim /etc/profile

```bash
export HADOOP_HOME=/usr/local/hadoop  
export HADOOP_PID_DIR=/data/hadoop/pids  
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native  
export HADOOP_OPTS="$HADOOP_OPTS-Djava.library.path=$HADOOP_HOME/lib/native"  
export HADOOP_MAPRED_HOME=$HADOOP_HOME  
export HADOOP_COMMON_HOME=$HADOOP_HOME  
export HADOOP_HDFS_HOME=$HADOOP_HOME  
export YARN_HOME=$HADOOP_HOME  
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop  
export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop  
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop  
export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native  
export PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin 
```

<font color="red">**source /etc/profile**</font>

接下来有8个配置文件需要修改，配置文件均在$HADOOP_HOME/etc/hadoop/目录下

`hadoop-env.sh`， `mapred-env.sh`，`yarn-env.sh`中加入以下内容

```bash
export JAVA_HOME=/usr/local/java
export CLASS_PATH=$JAVA_HOME/lib:$JAVA_HOME/jre/lib 
  
export HADOOP_HOME=/usr/local/hadoop 
export HADOOP_PID_DIR=/data/hadoop/pids 
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native 
export HADOOP_OPTS="$HADOOP_OPTS-Djava.library.path=$HADOOP_HOME/lib/native" 
  
export HADOOP_PREFIX=$HADOOP_HOME 
  
export HADOOP_MAPRED_HOME=$HADOOP_HOME 
export HADOOP_COMMON_HOME=$HADOOP_HOME 
export HADOOP_HDFS_HOME=$HADOOP_HOME 
export YARN_HOME=$HADOOP_HOME 
  
export HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop 
export HDFS_CONF_DIR=$HADOOP_HOME/etc/hadoop 
export YARN_CONF_DIR=$HADOOP_HOME/etc/hadoop 
  
export JAVA_LIBRARY_PATH=$HADOOP_HOME/lib/native 
  
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```

##### 2.2.1.2 core-site.xml

```xml
<configuration>
    <property> 
        <name>fs.defaultFS</name> 
        <value>hdfs://myha</value> <!--此处不能有‘-’符号-->
    </property> 

    <property> 
        <name>io.file.buffer.size</name> 
        <value>131072</value> 
    </property> 

    <property> 
        <name>hadoop.tmp.dir</name> 
        <value>file:/data/hadoop/storage/tmp</value> 
    </property> 

    <property> 
        <name>ha.zookeeper.quorum</name> 
        <value>linux01:2181,linux02:2181,linux03:2181</value> 
    </property> 
 
    <property> 
        <name>ha.zookeeper.session-timeout.ms</name> 
        <value>2000</value> 
    </property> 

    <property> 
        <name>fs.trash.interval</name> 
        <value>4320</value> 
    </property> 

    <property> 
         <name>hadoop.http.staticuser.use</name> 
         <value>root</value> 
    </property> 

    <property> 
        <name>hadoop.proxyuser.hadoop.hosts</name> 
        <value>*</value> 
    </property> 

    <property> 
        <name>hadoop.proxyuser.hadoop.groups</name> 
        <value>*</value> 
    </property> 

    <property> 
        <name>hadoop.native.lib</name> 
        <value>true</value> 
    </property> 
</configuration>
```

##### 2.2.1.3 hdfs-site.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<!--
  Licensed under the Apache License, Version 2.0 (the "License");
  you may not use this file except in compliance with the License.
  You may obtain a copy of the License at
    http://www.apache.org/licenses/LICENSE-2.0
  Unless required by applicable law or agreed to in writing, software
  distributed under the License is distributed on an "AS IS" BASIS,
  WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
  See the License for the specific language governing permissions and
  limitations under the License. See accompanying LICENSE file.
-->
 
<!-- Put site-specific property overrides in this file. -->
 
<configuration>
    <property> 
        <name>dfs.namenode.name.dir</name> 
        <value>file:/data/hadoop/storage/hdfs/name</value> 
    </property> 

    <property> 
        <name>dfs.datanode.data.dir</name> 
        <value>file:/data/hadoop/storage/hdfs/data</value> 
    </property> 

    <property> 
        <name>dfs.replication</name> 
        <value>2</value> 
    </property> 

    <property> 
        <name>dfs.blocksize</name> 
        <value>67108864</value> 
    </property> 

    <property> 
        <name>dfs.datanode.du.reserved</name> 
        <value>10737418240</value> 
    </property> 

    <property> 
        <name>dfs.webhdfs.enabled</name> 
        <value>true</value> 
    </property> 

    <property> 
        <name>dfs.permissions</name> 
        <value>true</value> 
    </property> 

    <property> 
        <name>dfs.permissions.enabled</name> 
        <value>true</value> 
    </property> 

    <property> 
        <name>dfs.nameservices</name> 
        <value>myha</value> 
    </property> 

    <property> 
        <name>dfs.ha.namenodes.myha</name> 
        <value>nn1,nn2</value> 
    </property> 

    <property> 
        <name>dfs.namenode.rpc-address.myha.nn1</name> 
        <value>linux01:8020</value> 
    </property> 
   
    <property> 
        <name>dfs.namenode.rpc-address.myha.nn2</name> 
        <value>linux02:8020</value> 
    </property> 

    <property> 
        <name>dfs.namenode.servicerpc-address.myha.nn1</name> 
        <value>linux01:53310</value> 
    </property> 

    <property> 
        <name>dfs.namenode.servicerpc-address.myha.nn2</name> 
        <value>linux02:53310</value> 
    </property> 

    <property> 
        <name>dfs.namenode.http-address.myha.nn1</name> 
        <value>linux01:50070</value> <!-- 该处不建议占掉8080端口，很多教程上都直接8080-->
    </property> 

    <property> 
        <name>dfs.namenode.http-address.myha.nn2</name> 
        <value>linux02:50070</value> 
    </property> 

    <property> 
        <name>dfs.datanode.http.address</name> 
        <value>0.0.0.0:50070</value> 
    </property> 

    <property> 
        <name>dfs.namenode.shared.edits.dir</name> 
        <value>qjournal://linux01:8485;linux02:8485;linux03:8485/myha</value>
    </property> 

    <property> 
        <name>dfs.client.failover.proxy.provider.myha</name> 
        <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value> 
    </property> 

    <property> 
        <name>dfs.ha.fencing.methods</name> 
        <value>sshfence</value> 
    </property> 

    <property> 
        <name>dfs.ha.fencing.ssh.private-key-files</name> 
        <value>/root/.ssh/id_rsa</value> 
    </property> 

    <property> 
        <name>dfs.ha.fencing.ssh.connect-timeout</name> 
        <value>30000</value> 
    </property> 

    <property> 
        <name>dfs.journalnode.edits.dir</name> 
        <value>/data/hadoop/storage/hdfs/journal</value> 
    </property> 

    <property> 
        <name>dfs.ha.automatic-failover.enabled</name> 
        <value>true</value> 
    </property> 

    <property> 
        <name>ha.failover-controller.cli-check.rpc-timeout.ms</name> 
        <value>60000</value> 
    </property> 

    <property> 
        <name>ipc.client.connect.timeout</name> 
        <value>60000</value> 
    </property> 

    <property> 
        <name>dfs.image.transfer.bandwidthPerSec</name> 
        <value>41943040</value> 
    </property> 

    <property> 
        <name>dfs.namenode.accesstime.precision</name> 
        <value>3600000</value> 
    </property> 

    <property> 
        <name>dfs.datanode.max.transfer.threads</name> 
        <value>4096</value> 
    </property> 
</configuration>
```

##### 2.2.1.4 mapred-site.xml

```xml
<configuration> 
    <property> 
        <name>mapreduce.framework.name</name> 
        <value>yarn</value> 
    </property> 

    <property> 
        <name>mapreduce.jobhistory.address</name> 
        <value>linux01:10020</value> 
    </property> 

    <property> 
        <name>mapreduce.jobhistory.webapp.address</name> 
        <value>linux01:19888</value> 
    </property>

	<property>
		<name>mapreduce.application.classpath</name>
		<value>
		/usr/local/hadoop/etc/hadoop,
		/usr/local/hadoop/share/hadoop/common/*,
		/usr/local/hadoop/share/hadoop/common/lib/*,
		/usr/local/hadoop/share/hadoop/hdfs/*,
		/usr/local/hadoop/share/hadoop/hdfs/lib/*,
		/usr/local/hadoop/share/hadoop/mapreduce/*,
		/usr/local/hadoop/share/hadoop/mapreduce/lib/*,
		/usr/local/hadoop/share/hadoop/yarn/*,
		/usr/local/hadoop/share/hadoop/yarn/lib/*
		</value>
	</property>
</configuration>
```

##### 2.2.1.5 yarn-site.xml

```xml
<configuration>
 
<!-- Site specific YARN configuration properties -->
    <property> 
        <name>yarn.nodemanager.aux-services</name> 
        <value>mapreduce_shuffle</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name> 
        <value>org.apache.hadoop.mapred.ShuffleHandler</value> 
    </property> 

    <property> 
        <name>yarn.resourcemanager.scheduler.address</name> 
        <value>linux01:8030</value> 
    </property> 

    <property> 
        <name>yarn.resourcemanager.resource-tracker.address</name> 
        <value>linux01:8031</value>
    </property> 

    <property> 
        <name>yarn.resourcemanager.address</name> 
        <value>linux01:8032</value> 
    </property> 

    <property> 
        <name>yarn.resourcemanager.admin.address</name> 
        <value>linux01:8033</value> 
    </property>

    <property> 
        <name>yarn.resourcemanager.webapp.address</name> 
        <value>linux01:80</value> 
    </property> 
 
	<property>
        <name>yarn.nodemanager.hostname</name>
        <value>linux01/linux02/linux03</value> <!-- 每个slave应该对应自己的hostName-->
        <description>the nodemanagers bind to this port</description>
    </property>
	
	<property> 
        <name>yarn.nodemanager.webapp.address</name> 
        <value>${yarn.nodemanager.hostname}:80</value> 
    </property> 
	
    <property>
        <name>yarn.nodemanager.address</name>
        <value>${yarn.nodemanager.hostname}:8034</value>
        <description>the nodemanagers bind to this port</description>
    </property>

    <property> 
        <name>yarn.nodemanager.local-dirs</name> 
        <value>${hadoop.tmp.dir}/nodemanager/local</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.remote-app-log-dir</name> 
        <value>${hadoop.tmp.dir}/nodemanager/remote</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.log-dirs</name> 
        <value>${hadoop.tmp.dir}/nodemanager/logs</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.log.retain-seconds</name> 
        <value>604800</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.resource.cpu-vcores</name> 
        <value>2</value> 
    </property> 

    <property> 
        <name>yarn.nodemanager.resource.memory-mb</name> 
        <value>10240</value> 
    </property> 

    <property> 
        <name>yarn.scheduler.minimum-allocation-mb</name> 
        <value>256</value> 
    </property> 

    <property> 
        <name>yarn.scheduler.maximum-allocation-mb</name> 
        <value>40960</value> 
    </property> 

    <property> 
        <name>yarn.scheduler.minimum-allocation-vcores</name> 
        <value>1</value> 
    </property> 

    <property> 
        <name>yarn.scheduler.maximum-allocation-vcores</name> 
        <value>8</value> 
    </property> 
</configuration>
```

##### 2.2.1.6 slaves

(此处如果超过一个节点，不要填主机名，要填IP)

```bash
192.168.134.101
192.168.134.102
192.168.134.103
```

#### 2.1.2 后台执行命令

接下来依次执行以下命令：

**a) 在namenode1上执行，创建命名空间**

```bash
hdfs zkfc -formatZK
```

**b) 在对应的节点上启动日志程序journalnode**

```bash
cd /usr/local/hadoop 
sh /sbin/hadoop-daemon.sh start journalnode
```

**c) 格式化主NameNode节点（node1）**

```bash
# hdfs namenode -format
```

**d) 启动主NameNode节点**

```bash
cd /usr/local/hadoop 
sh sbin/hadoop-daemon.sh start namenode
```

**e) 格式备NameNode节点（node2）**

```bash
hdfs namenode -bootstrapStandby
```

**f) 启动备NameNode节点（node2）**

```bash
cd /usr/local/hadoop
sh sbin/hadoop-daemon.sh start namenode
```

**g) 在两个NameNode节点（node1、node2）上执行**

```bash
cd /usr/local/hadoop
sh sbin/hadoop-daemon.sh start zkfc
```

**h) 启动所有的DataNode节点（node3）**

```bash
cd /usr/local/hadoop
sh sbin/hadoop-daemon.sh start datanode
```

**i) 启动Yarn（node1）**

```bash
cd /usr/local/hadoop
sh sbin/start-yarn.sh
```

### 2.2 Spark安装与配置

#### 2.2.1 解压并创建链接

```bash
tar xvzf -C spark-0.9.0-incubating.tgz /usr/local

cd /usr/local

ln -s spark-0.9.0-incubating spark
```

#### 2.2.2 修改环境变量

vim /etc/profile

```bash
export SPARK_HOME=/usr/local/spark
export PATH=$PATH:$SPARK_HOME/bin
```

<font color="red">**source /etc/profile**</font>

#### 2.2.3 创建数据目录

```bash
mkdir -p /data/spark/tmp
```

#### 2.2.4 配置两个配置文件

文件在SPARK_HOME/conf/目录下

##### 2.2.4.1 spark-env .sh

```bash
export JAVA_HOME=/usr/local/java
export SCALA_HOME=/usr/local/scala
export HADOOP_HOME=/usr/local/hadoop
export SPARK_MASTER_PORT=7077
export SPARK_MASTER_WEBUI_PORT=7070
export SPARK_WORKER_CORES=2
export SPARK_WORKER_MEMORY=1024m
export SPARK_WORKER_INSTANCES=2

export SPARK_LOCAL_DIR="/data/spark/tmp"
export SPARK_JAVA_OPTS="-Dspark.storage.blockManagerHeartBeatMs=60000-Dspark.local.dir=$SPARK_LOCAL_DIR -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -Xloggc:$SPARK_HOME/logs/gc.log -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=60"
```

##### 2.2.4.2 slaves (多个节点不能是主机名)

```bash
192.168.134.101
192.168.134.102
192.168.134.103
```

**在主节点直接执行`sbin/start-all.sh`**，或分别进入每个节点的/usr/local/spark/sbin目录下，主节点执行`./start-master.sh`，子节点执行`./start-slaves.sh`

### 2.3 测试流程

#### 2.3.1 进程运行情况测试


在每个节点执行jps指令，若输出结果为以下内容，则测试通过，否则进入/usr/local/hadoop/logs或者/usr/local/spark/logs目录下查看log文件进行检查

- *Master和Worker ----- Spark相应进程*
- *DFSZKFailoverController ----- Zookeeper相应进程*
- *ResourceManager和NodeManager ----- Yarn相应进程*

#### 2.3.2 HDFS测试

在任意节点下执行

```bash
hdfs dfs -mkdir /test
hdfs dfs -ls /
hdfs dfs -put /test/test.txt
```

若不报错，则说明测试通过

#### 2.3.3 Spark测试

##### 2.3.3.1 Spark本地模式测试(Spark Standalone)

```bash
run-exampleorg.apache.spark.examples.SparkPi 100
spark-submit --class org.apache.spark.examples.JavaWordCount --master spark://linux01:6066 --deploy-mode client /usr/local/spark/lib/spark-examples-1.6.1-hadoop2.6.0.jar ./test.txt
```

在http://linux01:4040中观察输出结果

```bash
spark-submit --class org.apache.spark.examples.JavaWordCount --master spark://linux01:6066 --deploy-mode cluster /usr/local/spark/lib/spark-examples-1.6.1-hadoop2.6.0.jar hdfs://[hdfsnamespace]/test/test.txt
```

在http://linux01:4040中观察输出结果

##### 2.3.3.2 Spark集群模式测试(Spark on Yarn)

```bash
spark-submit --class org.apache.spark.examples.JavaWordCount --master yarn --deploy-mode client /usr/local/spark/lib/spark-examples-1.6.1-hadoop2.6.0.jar hdfs://[hdfsnamespace]/test/test.txt
```

```bash
spark-submit --class org.apache.spark.examples.JavaWordCount --master yarn --deploy-mode cluster /usr/local/spark/lib/spark-examples-1.6.1-hadoop2.6.0.jar hdfs://[hdfsnamespace]/test/test.txt
```

可通过http://linux01:80进入UI界面查看