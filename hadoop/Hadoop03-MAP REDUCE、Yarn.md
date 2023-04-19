[TOC]
# 1. MAP REDUCE

mapreduce运行机制全解析--清明上河图
[![清明上河图.png](https://ae01.alicdn.com/kf/H51d6c9094e0946828032d4a47738f65am.jpg)](https://i0.wp.com/i.loli.net/2019/11/25/f7w6U3sbeJCYMrD.png)

一个分布式运算程序，往往需要分成多个阶段，最基本应该分2个阶段：

阶段1：读数据，拆分数据

阶段2：聚合数据

## 1.1 什么是MAP REDUCE
两个角度：

**角度1：MAP REDUCE是一种分布式运算编程模型**

它将整个运算逻辑分成两个阶段：

1. 阶段1：将数据映射成key:value形式
2. 阶段2：将key:value按相同key分组，对每组聚合运算

之所以这么设计，是因为可以方便地把整个运算过程实现成分布式并行计算的过程

 

**角度2：MAP REDUCE 是hadoop提供的一个对上述编程思想实现好的一套编程框架**

它为map阶段实现了一个程序：map task

它为reduce阶段实现了一个程序： reduce task

maptask 和reducetask都可以在多台机器上并行运行；（分布式并行运行）

它的这些程序已经完成了整个分布式运算过程中的绝大部分流程、工作；只需要用户提供两个阶段中的数据处理逻辑：

1. map阶段：映射逻辑（就是实现框架所提供的一个接口Mapper）
2. reduce阶段：聚合逻辑（就是实现框架所提供的一个接口：Reducer）

## 1.2 wordcount代码开发

### 1.2.1 需求：

hdfs的/wordcount/input/目录中有大量文本文件，需要统计所有文件中，每个单词出现的总次数；

### 1.2.2 实现思路：

可以采用hadoop所提供的mapreduce编程框架来写分布式计算程序进行单词统计；

### 1.2.3 实现步骤：

1) 建一个工程（导入mapreduce框架的所有jar包）
2) 新建一个类，实现Mapper接口，写我们自己的映射逻辑
3) 新建一个类，实现Reducer接口，写我们自己的聚合逻辑
4) 新建一个类，写一个提交job到yarn去运行的main方法
5) 再将工程打成jar包；把jar包上传到hadoop集群中的任意一台linux上；
6) 然后用命令启动jar包中的Driver类： hadoop jar wc.jar top.ganhoo.mr.wordcount.Driver

### 1.2.4 code
#### 1.2.4.1 WordcountMapper

```java
/**
 * KEYIN: 是maptask读取到的数据中的key（一行起始偏移量）的类型：Long
 * VALUEIN:是maptask读取到的数据中的value（一行的内容）类型：String
 * 
 * KEYOUT:是我们的映射逻辑所产生的key（单词）的类型：String
 * VALUEOUT:是我们的映射逻辑所产生的value（1）的类型：Integer
 * 
 * 
 * 但是，HADOOP中这些key：value数据需要经常进行磁盘序列化，网络传输，这些key-value数据需要实现序列化机制；
 * 但是jdk中的Long、String、Integer实现的是jdk自己的serializable序列化接口，这种序列化机制把一个对象序列化成字节时，体积臃肿
 * 所以，hadoop自己开发了一套序列化机制：Writable接口
 * 那么，在mapreduce编程中涉及到的需要序列化的数据都需要实现Writable接口
 * 所以，String应该替换成hadoop中已经实现了Writable接口的类型：Text
 * Long应该替换成：LongWritable类型
 * Integer应该替换成：IntWritable类型
 * Float应该替换成：FloatWritable类型
 * Double应该替换成：DoubleWritable类型
 * .....
 * 
 * 如果key或者value数据是用户自定义的类，那么这个类也得实现Writable接口
 * 
 * @author hunter.ganhoo
 *
 */



public class WordcountMapper extends Mapper<LongWritable, Text, Text, IntWritable>{
	
	/**
	 * 本方法是由maptask来调用的
	 * maptask每读取一行数据，就会调用一次，而且会将该行的起始偏移量传入参数key，该行的内容传入参数value
	 * context是提供给我们来返回映射结果数据的
	 */
	@Override
	protected void map(LongWritable key, Text value, Context context)
			throws IOException, InterruptedException {
		
		// 将该行切单词
		String line = value.toString();
		String[] words = line.split(" ");
		// 将每一个单词映射成 <单词：1>，并通过context返回给maptask
		for (String word : words) {
			context.write(new Text(word), new IntWritable(1));
		}
	}
}
```

#### 1.2.4.2 WordcountReducer
```java
package top.ganhoo.mr.wordcount;

import java.io.IOException;
import java.util.Iterator;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;


/**
 * KEYIN: reduce task传入的key的类型，（其实就是mapper阶段产生的key的类型）
 * VALUEIN:reduce task传入的value的类型（其实就是mapper阶段产生的value的类型）
 * 
 * KEYOUT: 是我们的聚合逻辑所产生的结果的key的类型
 * VALUEOUT:是我们的聚合逻辑所产生的结果的value的类型
 * @author hunter.ganhoo
 *
 */
public class WordcountReducer extends Reducer<Text, IntWritable, Text, IntWritable>{
	
	/**
	 * reduce方法由 reduce task程序调用
	 * reduce task 对每一组数据调用一次reduce方法
	 * 
	 * 那么，reduce task就得想办法把一组数据传入我们的reduce方法
	 * 方法中: 参数key就是这一组数据的key
	 *  参数values是这一组数据的一个迭代器，可以迭代这一组数据中所有的value
	 */
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,Context context) throws IOException, InterruptedException {

		Iterator<IntWritable> iterator = values.iterator();
		
		int count = 0;
		while(iterator.hasNext()) {
			IntWritable value = iterator.next();
			count += value.get();
		}
		
		context.write(key, new IntWritable(count));
	}
}
```

#### 1.2.4.3 Driver

```java
package top.ganhoo.mr.wordcount;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

/**
 * 用于设置job参数 并提交mr程序相关资源（jar包，配置参数等）给yarn去运行
 * 
 * @author hunter.ganhoo
 *
 */
public class Driver {

	public static void main(String[] args) throws Exception {
		System.setProperty("HADOOP_USER_NAME", "root");
		Configuration conf = new Configuration();

		conf.set("mapreduce.framework.name", "yarn");
		conf.set("yarn.resourcemanager.hostname", "hdp26-01");
		conf.set("fs.defaultFS", "hdfs://hdp26-01:9000");
		conf.set("mapreduce.app-submission.cross-platform", "true");

		conf.set("mapreduce.input.fileinputformat.split.maxsize", "134217728");

		Job job = Job.getInstance(conf);

		// 告知job对象，我们的mr程序jar所在的本地路径
		job.setJar("d:/wc.jar");
		// job.setJarByClass(Driver.class);

		// 设置本次job所使用的mapper类和reducer类
		job.setMapperClass(WordcountMapper.class);
		job.setReducerClass(WordcountReducer.class);

		// 设置本次job的map阶段输出的key-value类型
		job.setMapOutputKeyClass(Text.class);
		job.setMapOutputValueClass(IntWritable.class);

		// 设置本次job的reduce阶段输出的key-value类型
		job.setOutputKeyClass(Text.class);
		job.setOutputValueClass(IntWritable.class);

		// 设置本次job的输入数据所在路径
		FileInputFormat.setInputPaths(job, new Path(args[0]));

		// 设置本次job的输出结果文件所在路径
		FileOutputFormat.setOutputPath(job, new Path(args[1]));

		// 设置本次job的reduce task运行实例数
		// maptask的数量不能这样直接设置，mr框架内有一个固定的机制来计算需要启动多少个maptask
		job.setNumReduceTasks(2);

		// 将本次job的jar包和设置的上述信息提交给yarn
		// job.submit();
		boolean res = job.waitForCompletion(true);
		System.out.println(res ? 数据处理完毕" : "程序错误，已退出");
	}
}
```

# 2. Yarn命令
来源： https://blog.csdn.net/qianshangding0708/article/details/47395783

启动单个进程命令

```bash
yarn-daemon.sh start resourcemanager
yarn-daemon.sh start nodemanager
```

## 2.1 概述
mapreduce在yarn上分布式运行全生命周期示意图--乾坤大挪移

[![乾坤大挪移.png](https://pic.imgdb.cn/item/5f0ea16f14195aa59428bede.png)](https://images.weserv.nl/?url=https://files.catbox.moe/ozp7ev.png)

YARN命令是调用bin/yarn脚本文件，如果运行yarn脚本没有带任何参数，则会打印yarn所有命令的描述。

使用: 

```shell
yarn [--config confdir] COMMAND [--loglevel loglevel] [GENERIC_OPTIONS] [COMMAND_OPTIONS]
```

YARN有一个参数解析框架，采用解析泛型参数以及运行类。

|命令参数|描述|
|---|---|
|--config confdir|指定一个默认的配置文件目录，默认值是： ${HADOOP_PREFIX}/conf.|
|--loglevel loglevel|重载Log级别。有效的日志级别包含：FATAL, ERROR, WARN, INFO, DEBUG, and TRACE。默认是INFO。|
|GENERIC_OPTIONS|YARN支持<font color="red">表A</font>的通用命令项。|
|COMMAND COMMAND_OPTIONS|YARN分为用户命令和管理员命令。|

**表A：**

|通用项|Description|
|---|---|
|-archives <comma separated list of archives>|	用逗号分隔计算中未归档的文件。 仅仅针对JOB。|
|-conf <configuration file>|	制定应用程序的配置文件。|
|-D <property>=<value>|使用给定的属性值。|
|-files <comma separated list of files>|	用逗号分隔的文件,拷贝到Map reduce机器，仅仅针对JOB|
|-jt <local> or <resourcemanager:port>|	指定一个ResourceManager. 仅仅针对JOB。|
|-libjars <comma seperated list of jars>|	将用逗号分隔的jar路径包含到classpath中去，仅仅针对JOB。|

## 2.2 用户命令：

对于Hadoop集群用户很有用的命令：
### 2.2.1 application|app

使用: 

```shell
yarn application [options]
yarn app [options]
```

|命令选项|描述|
|---|:---|
| \-appId <ApplicationId> | 指定要操作的应用程序 Id |
| **\-list** | 从RM返回的应用程序列表。支持可选使用 -appTypes 来过滤基于应用程序类型的应用程序，-appStates 来过滤基于应用程序状态的应用程序，-appTags 来过滤基于应用程序标签的应用程序。 |
| **\-kill <Application ID>** | 杀死该应用程序。可以提供与空间分隔的应用程序集 |
| \-appStates <States> | 使用-list命令，基于应用程序的状态来过滤应用程序，如果应用程序的状态有多个，用逗号分隔。有效的应用程序状态可以是以下状态之一：ALL, NEW, NEW\_SAVING, SUBMITTED, ACCEPTED, RUNNING, FINISHED, FAILED, KILLED |
| \-appTags <Tags> | 与 -list 一起工作过滤基于输入逗号分隔的应用程序标签列表的应用程序。 |
| \-appTypes <Types> | 使用 -list 根据输入逗号分隔的应用程序类型列表过滤应用程序。 |
| \-changeQueue <Queue Name> | 将应用程序移动到一个新队列。可以使用 “appId” 选项传递 ApplicationId。' movetoqueue '命令不赞成使用，这个新命令' changeQueue '执行相同的功能。 |
| \-component <Component Name> <Count> | 与 -flex 选项一起工作，以改变组件/容器的数量运行的应用程序/长期运行的服务。支持绝对或相对更改，如+1、2或-3。 |
| \-components <Components> | 使用 -upgrade 选项触发应用程序指定组件的升级。多个组件之间应该用逗号分隔。 |
| \-decommission <Application Name> | 为应用程序/长时间运行的服务退役组件实例。需要实例的选择。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-destroy <Application Name> | 销毁已保存的应用程序规范并永久删除所有应用程序数据。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-enableFastLaunch | 上传对 HDFS 的依赖，使未来的启动更快。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-flex <Application Name or ID> | 更改应用程序/长时间运行的服务的组件的运行容器的数量。需要分的选择。如果提供了名称，则必须提供 appType，除非它是默认的 yarn-service。如果提供了 ID，将查找 appType。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-help | 显示所有命令的帮助。 |
| \-instances <Component Instances> | 使用 -upgrade 选项触发应用程序指定组件实例的升级。还可以使用 -decommission 选项来退役指定的组件实例。多个实例之间应该用逗号分隔。 |
| \-launch <Application Name> <File Name> | 从规范文件启动应用程序(保存规范并启动应用程序)。可以指定选项 -updateLifetime 和 -changeQueue 来更改文件中提供的值。支持 -appTypes 选项指定要使用的客户端实现。 |
| ~~\-movetoqueue <Application ID>~~ | ~~将应用程序移动到另一个队列~~。弃用的命令。使用“changeQueue”代替。 |
| \-queue <Queue Name> | 使用 movetoqueue 命令指定要将应用程序移动到哪个队列。 |
| \-save <Application Name> <File Name> | 保存应用程序的规范文件。可以指定选项 -updateLifetime 和 -changeQueue 来更改文件中提供的值。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-start <Application Name> | 启动先前保存的应用程序。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-status <ApplicationId or ApplicationName> | 打印应用程序的状态。如果提供了app ID，则打印通用 yarn 应用状态。如果提供了名称，它将根据应用程序自己的实现打印应用程序特定的状态，并且必须指定 -appTypes 选项，除非它是默认的 yarn-service 类型。 |
| \-stop <Application Name or ID> | 优雅地停止应用程序(可能稍后再次启动)。如果提供了名称，则必须提供 appType，除非它是默认的 yarn-service。如果提供了ID，将查找 appType。支持 -appTypes 选项指定要使用的客户端实现。 |
| \-updateLifetime <Timeout> | 从现在开始更新应用程序的超时。可以使用“appId”选项传递 ApplicationId。超时值的单位是秒。 |
| \-updatePriority <Priority> | 应用程序的更新优先级。可以使用“appId”选项传递 ApplicationId。 |
```bash
yarn application -list

yarn app -kill application_1637552234049_5254
```

### 2.2.2 applicationattempt

使用:

```shell
yarn applicationattempt [options]
```

|命令选项|描述|
|---|:---|
|-help|帮助|
|-list <ApplicationId>|	获取到应用程序尝试的列表，<font color="red">其返回值ApplicationAttempt-Id等于 <Application Attempt Id></font>|
|-status <Application Attempt Id>|	打印应用程序尝试的状态。|

打印应用程序尝试的报告。

示例1：

```
[hadoop@hadoop01 bin]$ yarn applicationattempt -list application_1437364567082_0106  
    15/08/10 20:58:28 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032  
    Total number of application attempts :1  
    ApplicationAttempt-Id                  State    AM-Container-Id                        Tracking-URL  
    appattempt_1437364567082_0106_000001   RUNNING  container_1437364567082_0106_01_000001 http://hadoop01:8088/proxy/application_1437364567082_0106/
```

示例2：

```
[hadoop@hadoop01 bin]$ yarn applicationattempt -status appattempt_1437364567082_0106_000001  
15/08/10 21:01:41 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032  
Application Attempt Report :   
    ApplicationAttempt-Id : appattempt_1437364567082_0106_000001  
    State : FINISHED  
    AMContainer : container_1437364567082_0106_01_000001  
    Tracking-URL : http://hadoop01:8088/proxy/application_1437364567082_0106/jobhistory/job/job_1437364567082_0106  
    RPC Port : 51911  
    AM Host : hadoop01  
    Diagnostics :
```

### 2.2.3 node

使用: 

```shell
yarn node [options]
```


|命令选项|描述|
|---|---|
|-all|所有的节点，不管是什么状态的。|
|-list|列出所有RUNNING状态的节点。支持-states选项过滤指定的状态，节点的状态包含：NEW，RUNNING，UNHEALTHY，DECOMMISSIONED，LOST，REBO|OTED。支持--all显示所有的节点。|
|-states <States>|和-list配合使用，用逗号分隔节点状态，只显示这些状态的节点信息。|
|-status <NodeId>|打印指定节点的状态。|

示例1：

```shell
yarn@hadoop01 ~]$ yarn node -list -all
19/11/15 15:57:05 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032
Total Nodes:3
         Node-Id	     Node-State	   Node-Http-Address	Number-of-Running-Containers
     hadoop01:45454	        RUNNING	       hadoop01:8042                               2
     hadoop02:45454	        UNHEALTHY      hadoop02:8042                               0
     hadoop03:45454	        RUNNING	       hadoop03:8042                               2
     
```

示例2：

```
[yarn@sd178 ~]$ yarn node -list -states RUNNING
19/11/15 15:59:33 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032
Total Nodes:2
         Node-Id	     Node-State	   Node-Http-Address	Number-of-Running-Containers
     hadoop01:45454	        RUNNING	       hadoop01:8042                               2
     hadoop03:45454	        RUNNING	       hadoop03:8042                               2

# 列出不健康的 nodemanager
yarn node -list -states UNHEALTHY
# 列出丢失心跳的 nodemanager
yarn node -list -all|grep LOST

```

示例3：打印节点的报告。

```shell
[yarn@hadoop01 ~]$ yarn node -status hadoop01:45454
19/11/15 16:03:32 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032
Node Report : 
	Node-Id : hadoop01:45454
	Rack : /default-rack
	Node-State : RUNNING
	Node-Http-Address : hadoop01:8042
	Last-Health-Update : 星期五 15/十一月/19 04:01:51:454CST
	Health-Report : 
	Containers : 2
	Memory-Used : 2048MB
	Memory-Capacity : 122880MB
	CPU-Used : 2 vcores
	CPU-Capacity : 32 vcores
	Node-Labels : 
```

### 2.2.4 classpath
使用: 
```shell
yarn classpath
```

|命令选项|描述|
|---|---|
|--glob	|扩大通配符|
|--jar path	|在 jar 命名路径中编写类路径作为清单|
|-h, --help	|print help|

打印需要得到Hadoop的jar和所需要的lib包路径

```shell
[hadoop@sd178 bin]$ yarn classpath  
/home/hadoop/apache/hadoop-2.4.1/etc/hadoop:
/home/hadoop/apache/hadoop-2.4.1/etc/hadoop:
/home/hadoop/apache/hadoop-2.4.1/etc/hadoop:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/common/lib/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/common/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/hdfs:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/hdfs/lib/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/hdfs/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/yarn/lib/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/yarn/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/mapreduce/lib/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/mapreduce/*:
/home/hadoop/apache/hadoop-2.4.1/contrib/capacity-scheduler/*.jar:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/yarn/*:
/home/hadoop/apache/hadoop-2.4.1/share/hadoop/yarn/lib/*  
```

### 2.2.5 container

使用:
```shell
yarn container [options]
```

|命令选项|描述|
|---|---|
|-help|帮助|
|-list|<Application Attempt Id>|	|应用程序尝试的Containers列表|
|-status <ContainerId>|打印Container的状态|

打印container(s)的报告

示例1：

```shell
[hadoop@hadoop01 bin]$ yarn container -list appattempt_1437364567082_0106_01   
15/08/10 20:45:45 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032  
Total number of containers :6  
             Container-Id                         Start Time                Finish Time              State       Host                                LOG-URL  
container_1437364567082_0106_01_000028         1439210458659                       0                 RUNNING    hadoop01:37140   //hadoopcluster83:8042/node/containerlogs/container_1437364567082_0106_01_000028/hadoop  
container_1437364567082_0106_01_000016         1439210314436                       0                 RUNNING    hadoop02:43818   //hadoopcluster84:8042/node/containerlogs/container_1437364567082_0106_01_000016/hadoop  
container_1437364567082_0106_01_000019         1439210338598                       0                 RUNNING    hadoop01:37140   //hadoopcluster83:8042/node/containerlogs/container_1437364567082_0106_01_000019/hadoop  
container_1437364567082_0106_01_000004         1439210314130                       0                 RUNNING    hadoop03:48622   //hadoopcluster82:8042/node/containerlogs/container_1437364567082_0106_01_000004/hadoop  
container_1437364567082_0106_01_000008         1439210314130                       0                 RUNNING    hadoop03:48622   //hadoopcluster82:8042/node/containerlogs/container_1437364567082_0106_01_000008/hadoop  
container_1437364567082_0106_01_000031         1439210718604                       0                 RUNNING    hadoop01:37140   //hadoopcluster83:8042/node/containerlogs/container_1437364567082_0106_01_000031/hadoop
```

示例2：

```
[hadoop@hadoop01 bin]$ yarn container -status container_1437364567082_0105_01_000020
15/08/10 20:28:00 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.68.1.101:8032
Container Report :
    Container-Id : container_1437364567082_0105_01_000020
    Start-Time : 1439208779842
    Finish-Time : 0
    State : RUNNING
    LOG-URL : //hadoop01:8042/node/containerlogs/container_1437364567082_0105_01_000020/hadoop
    Host : hadoop01:37140
    Diagnostics : null
```

### 2.2.6 jar

使用: 
```shell
yarn jar <jar> [mainClass] args...
```

运行jar文件，用户可以将写好的YARN代码打包成jar文件，用这个命令去运行它。

### 2.2.7 logs

使用:
```shell
yarn logs -applicationId <application ID> [options]
```

注：<font color="red">应用程序没有完成，该命令是不能打印日志的。</font>

|命令选项|描述|
|---|---|
|-applicationId <application ID>|	指定应用程序ID，应用程序的ID可以在yarn.resourcemanager.webapp.address配置的路径查看(即：ID)|
|-appOwner <AppOwner>|	应用的所有者(如果没有指定就是当前用户)应用程序的ID可以在yarn.resourcemanager.webapp.address配置的路径查看(即：User)|
|-containerId <ContainerId>|Container Id|
|-help|帮助|
|-nodeAddress <NodeAddress>|节点地址的格式:nodename:port(端口是配置文件中:yarn.nodemanager.webapp.address参数指定)|

转存container的日志。
示例：

```shell
[hadoop@hadoop01 bin]$ yarn logs -applicationId application_1437364567082_0104  -appOwner hadoop  
15/08/10 17:59:19 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032  
  
  
Container: container_1437364567082_0104_01_000003 on hadoopcluster82_48622  
============================================================================  
LogType: stderr  
LogLength: 0  
Log Contents:  
  
LogType: stdout  
LogLength: 0  
Log Contents:  
  
LogType: syslog  
LogLength: 3673  
Log Contents:  
2015-08-10 17:24:01,565 WARN [main] org.apache.hadoop.conf.Configuration: job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.retry.interval;  Ignoring.  
2015-08-10 17:24:01,580 WARN [main] org.apache.hadoop.conf.Configuration: job.xml:an attempt to override final parameter: mapreduce.job.end-notification.max.attempts;  Ignoring.  
。。。。。。此处省略N万个字符  
// 下面的命令，根据APP的所有者查看LOG日志，因为application_1437364567082_0104任务我是用hadoop用户启动的，所以打印的是如下信息：  
[hadoop@hadoop01 bin]$ yarn logs -applicationId application_1437364567082_0104  -appOwner root  
15/08/10 17:59:25 INFO client.RMProxy: Connecting to ResourceManager at hadoop01/192.168.1.101:8032  
Logs not available at /tmp/logs/root/logs/application_1437364567082_0104  
Log aggregation has not completed or is not enabled.
```

### 2.2.8 queue

使用: 
```shell
yarn queue [options]
```

|命令选项|描述|
|---|---|
|-help|帮助|
|-status <QueueName>|打印队列的状态|

打印队列信息。

### 2.2.9 version

使用: 
```shell
yarn version
```

打印hadoop的版本。


## 2.3 管理员命令：

下列这些命令对hadoop集群的管理员是非常有用的。

### 2.3.1 daemonlog

获取/设置每个守护进程的日志级别。

使用:

```shell
yarn daemonlog -getlevel <host:httpport> <classname> 
yarn daemonlog -setlevel <host:httpport> <classname> <level>
```

|参数选项|描述|
|---|---|
|-getlevel <host:httpport> <classname>|	打印运行在<host:port>的守护进程的日志级别。这个命令内部会连接http://<host:port>/logLevel?log=<name>|
|-setlevel <host:httpport> <classname> <level>|	设置运行在<host:port>的守护进程的日志级别。这个命令内部会连接http://<host:port>/logLevel?log=<name>|

针对指定的守护进程，获取/设置日志级别.

示例1:

```shell
[root@hadoop01 ~]# hadoop daemonlog -getlevel hadoop02:50075 org.apache.hadoop.hdfs.server.datanode.DataNode  
Connecting to http://hadoop02:50075/logLevel?log=org.apache.hadoop.hdfs.server.datanode.DataNode  
Submitted Log Name: org.apache.hadoop.hdfs.server.datanode.DataNode  
Log Class: org.apache.commons.logging.impl.Log4JLogger  
Effective level: INFO  
  
[root@hadoop01 ~]# yarn daemonlog -getlevel hadoop02:8088 org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl  
Connecting to http://hadoop02:8088/logLevel?log=org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl  
Submitted Log Name: org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl  
Log Class: org.apache.commons.logging.impl.Log4JLogger  
Effective level: INFO  
  
[root@hadoop01 ~]# yarn daemonlog -getlevel hadoop01:19888 org.apache.hadoop.mapreduce.v2.hs.JobHistory  
Connecting to http://hadoop01:19888/logLevel?log=org.apache.hadoop.mapreduce.v2.hs.JobHistory  
Submitted Log Name: org.apache.hadoop.mapreduce.v2.hs.JobHistory  
Log Class: org.apache.commons.logging.impl.Log4JLogger  
Effective level: INFO
```

### 2.3.2 proxyserver（代理服务）

使用: 
```shell
yarn proxyserver
```


启动web proxy server

### 2.3.3 nodemanager（节点管理）
使用: 
```shell
yarn nodemanager
```


启动NodeManager



### 2.3.4 resourcemanager（资源管理）

使用: 
```shell
yarn resourcemanager [-format-state-store]
```

|参数选项|描述|
|---|---|
|-format-state-store|RMStateStore的格式. 如果过去的应用程序不再需要，则清理RMStateStore， RMStateStore仅仅在ResourceManager没有运行的时候，才运行RMStateStore|

启动ResourceManager

### 2.3.5 rmadmin

运行ResourceManager管理客户端
使用:

```shell
yarn rmadmin [-refreshQueues]
             [-refreshNodes]
             [-refreshUserToGroupsMapping] 
             [-refreshSuperUserGroupsConfiguration]
             [-refreshAdminAcls] 
             [-refreshServiceAcl]
             [-getGroups [username]]
             [-transitionToActive [--forceactive] [--forcemanual] <serviceId>]
             [-transitionToStandby [--forcemanual] <serviceId>]
             [-failover [--forcefence] [--forceactive] <serviceId1> <serviceId2>]
             [-getServiceState <serviceId>]
             [-checkHealth <serviceId>]
             [-help [cmd]]
```

|参数选项|描述|
|---|---|
|-refreshQueues|	重载队列的ACL，状态和调度器特定的属性，ResourceManager将重载mapred-queues配置文件|
|-refreshNodes|动态刷新dfs.hosts和dfs.hosts.exclude配置，无需重启NameNode。|
|dfs.hosts|列出了允许连入NameNode的datanode清单(IP或者机器名)|
|dfs.hosts.exclude|列出了禁止连入NameNode的datanode清单(IP或者机器名) 重新读取hosts和exclude文件，更新允许连到Namenode的或那些需要退出或入编的Datanode的集合。|
|-refreshUserToGroupsMappings|刷新用户到组的映射。|
|-refreshSuperUserGroupsConfiguration|刷新用户组的配置|
|-refreshAdminAcls|刷新ResourceManager的ACL管理|
|-refreshServiceAcl|ResourceManager重载服务级别的授权文件。|
|-getGroups [username]|获取指定用户所属的组。|
|-transitionToActive [–forceactive] [–forcemanual] <serviceId>|尝试将目标服务转为 Active 状态。如果使用了–forceactive选项，不需要核对非Active节点。如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual选项，你需要谨慎。|
|-transitionToStandby [–forcemanual] <serviceId>|	将服务转为 Standby 状态. 如果采用了自动故障转移，这个命令不能使用。虽然你可以重写–forcemanual选项，你需要谨慎。|
|-failover [–forceactive] <serviceId1> <serviceId2>|	启动从serviceId1 到 serviceId2的故障转移。如果使用了-forceactive选项，即使服务没有准备，也会尝试故障转移到目标服务。如果采用了自动故障转移，这个命令不能使用。|
|-getServiceState <serviceId>	|返回服务的状态。(注：ResourceManager不是HA的时候，时不能运行该命令的)|
|-checkHealth <serviceId>|请求服务器执行健康检查，如果检查失败，RMAdmin将用一个非零标示退出。(注：ResourceManager不是HA的时候，时不能运行该命令的)|
|-help [cmd]|显示指定命令的帮助，如果没有指定，则显示命令的帮助。|



```bash
# 查看哪个 rm 处于 active 的状态
yarn rmadmin -getAllServiceState
```


### 2.3.6 scmadmin

使用: 
```shell
yarn scmadmin [options]
```

|参数选项|描述|
|---|---|
|-help|Help|
|-runCleanerTask|Runs the cleaner task|

Runs Shared Cache Manager admin client



### 2.3.7 sharedcachemanager

使用: 
```shell
yarn sharedcachemanager
```

启动Shared Cache Manager


## 2.4 Yarn资源分配性能调优
来源:https://blog.csdn.net/lively1982/article/details/50598847

日志：

> Container [pid=134663,containerID=container_1430287094897_0049_02_067966] is running beyond physical memory limits. Current usage: 1.0 GB of 1 GB physical memory used; 1.5 GB of 10 GB virtual memory used. Killing container. Dump of the process-tree for
> 
> Error: Java heap space

问题1：Container xxx is running beyond physical memory limits

问题2：java heap space

优化前：

```shell
yarn.nodemanager.resource.memory-mb   8 GB
yarn.nodemanager.resource.cpu-vcores  32 core
```

```
pre Mapper 
CPU:1   [mapreduce.map.cpu.vcores ]
MEM:1G  [mapreduce.map.memory.mb ]
===> 8 map slot / node
 
pre Reducer
CPU:1   [mapreduce.reduce.cpu.vcores]
MEM:1G  [mapreduce.reduce.memory.mb]
===> 8 reduce slot / node 【有8G内存，实际有CPU 32个，所以只能启动8个reduce在每个node上】
```

- map slot / reduce slot 由nodemanager的内存/CPU core上限与客户端设置的单mapper, reducer内存/CPU使用值决定
- heapsize（ java.opts中的-Xmx）应根据单mapper, reducer内存进行调整，而与slot个数无关 => heapsize不能大于memory.mb值,一般设置为memory.mb的85%左右

### 2.4.1 OOM
**(1) 内存、Heap**

需要设置：
- 内存：mapreduce.map.memory.mb
- Heap Size：-Xmx在mapreduce.map.java.opts做相同调整
- 内存：mapreduce.reduce.memory.mb
- Heap Size：-Xmx在mapreduce.reduce.java.opts做相同调整

**(2) Container 超过了虚拟内存的使用限制**
> Container XXX is running beyond virtual memory limits

NodeManager端设置，类似系统层面的overcommit问题

- yarn.nodemanager.vmem-pmem-ratio
- 【默认2.1，我们的做法呢【物理内存和虚拟内存比率】值为了15，yarn-site.xml中修改】


```xml
<property>
    <name>yarn.nodemanager.vmem-pmem-ratio</name>
        <value>10</value>
    </property>
```

或者yarn.nodemanager.vmem-check-enabled，false掉
```xml
    <property>
        <name>yarn.nodemanager.vmem-check-enabled</name>
        <value>false</value>
    </property>
```

调优后：
- mapreduce.map.java.opts, mapreduce.map.java.opts.max.heap=1.6G
- mapreduce.reduce.java.opts,mapreduce.reduce.java.opts.max.heap=3.3G

注意上面两个参数和下面的mapper,reducer的内存有关系,是下面mem的0.85倍！

- yarn.nodemanager.resource.memory-mb=32GB
- yarn.nodemanager.resource.cpu-vcores=32core


```shell
pre Mapper 
CPU:2   [mapreduce.map.cpu.vcores ]
MEM:2G  [mapreduce.map.memory.mb ]
===> 16 map slot / node
 
pre Reducer
CPU:4   [mapreduce.reduce.cpu.vcores]
MEM:4G  [mapreduce.reduce.memory.mb]
==> 8 reduce slot / node
```

### 2.4.2 shuffle.parallelcopies如何计算?
（reduce.shuffle并行执行的副本数,最大线程数-sqrt(节点数 map slot数) 与 (节点数 map slot数)/2 之间 ==>结果：{12-72}

mapreduce.reduce.shuffle.parallelcopies=68

> 排序文件时要合并的流的数量。也就是说，在 reducer 端合并排序期间要使用的排序头
> 数量。此设置决定打开文件句柄数。并行合并更多文件可减少合并排序迭代次数并通过消
> 除磁盘 I/O 提高运行时间。注意：并行合并更多文件会使用更多的内存。如 `io.sort.
> factor` 设置太高或最大 JVM 堆栈设置太低，会产生过多地垃圾回收。Hadoop 默认值为 
> 10，但 Cloudera 建议使用更高值。将是生成的客户端配置的一部分。

mapreduce.task.io.sort.factor=64

xml配置
yarn.nodemanager.vmem-pmem-ratio=10 # yarn-site.xml的 YARN 客户端高级配置

mapreduce.task.timeout=1800000

### 2.4.3 impala调优
Impala 暂存目录：需要注意此目录磁盘空间问题！最好在单独的一个挂载点！

**1、内存**

```
服务器端(impalad)
Mem：default_query_options MEM_LIMIT=128g
```

**2、并发查询**

```
queue
  queue_wait_timeout_ms默认只有60s
    queue_wait_timeout_ms=600000
  default pool设置
```

**3、资源管理**

```
Dynamic Resource Pools
   并发控制：max running queries
```

**4、yarn资源隔离**

http://hadoop.apache.org/docs/r2.7.1/hadoop-yarn/hadoop-yarn-site/NodeManagerCgroups.html