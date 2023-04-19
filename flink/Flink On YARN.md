[TOC]

参考：<br>
[Flink 1.10 Run 命令详解](https://blog.csdn.net/Zsigner/article/details/107787344)

[Flink On YARN](https://blog.csdn.net/lingeio/article/details/103833711)

# 1. 任务提交模式

Flink多种提交方式对比

常用提交方式分为local，standalone，yarn三种。

- local：本地提交项目，可纯粹的在本地单节点运行，也可以将本地代码提交到远端flink集群运行。
- standalone：flink集群自己完成资源调度，不依赖于其他资源调度器，需要手动启动flink集群。
- yarn：依赖于hadoop yarn资源调度器，由yarn负责资源调度，不需要手动启动flink集群。需要先启动yarn和hdfs。又分为yarn-session和yarn-cluster两种方式。提交Flink任务时，所在机器必须要至少设置环境变量`YARN_CONF_DIR`、`HADOOP_CONF_DIR`、`HADOOP_CONF_PATH`中的一个，才能读取YARN和HDFS的配置信息（会按三者**从左到右**的顺序读取，只要发现一个就开始读取。如果没有正确设置，会尝试使用HADOOP_HOME/etc/hadoop），否则提交任务会失败。

## 1.1 local模式

local即本地模式，可以不依赖hadoop，可以不搭建flink集群。一般在开发时调试时使用。
### 1.1.1 纯粹的local模式运行

就是直接运行项目中的代码的方式，例如直接在idea中运行。创建ExecutionEnvironment的方式如下：

```java
// getExecutionEnvironment()方法可以根据flink运用程序如何提交判断出是那种模式提交
//ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
ExecutionEnvironment env = ExecutionEnvironment.createLocalEnvironment();
```

### 1.1.2 local使用remote的方式运行

这种方式可以将本地代码提交给远端flink集群运行，需要指定集群的master地址。在flink集群的web ui会存在Running Job/Compaleted Jobs的记录。

```java
public class TestLocal {
    public static void main(String[] args) throws Exception {

        ExecutionEnvironment env = ExecutionEnvironment.createRemoteEnvironment("remote_ip", 8090, "D:\\code\\flink-local.jar");

        System.out.println(env.getParallelism());

        env.readTextFile("hdfs://remote_ip:9000/tmp/test.txt")
                .print();
    }
}
```

### 1.1.3 本地提交到remote集群

例如有如下项目代码：

```java
public class TestLocal {
    public static void main(String[] args) throws Exception {
        ExecutionEnvironment env = ExecutionEnvironment.getExecutionEnvironment();
        env.readTextFile("hdfs://remote_ip:9000/tmp/test.txt")
                .print();
    }
}

```

将项目打成jar包，在本机上使用flink run 指定集群的模式提交（这个机器可以不是flink集群的节点，但需要在local机器上有flink提交命令，）。如：

```bash
./flink run -m remote_ip:8090 -p 1 -c com.test.TestLocal /home/hdp/flink-local.jar
✦ -m flink 集群地址
✦ -p 配置job的并行度
✦ -c Job执行的main class
```


也会在flink web ui界面显示结果。
## 1.2 standalone模式

上面讲了flink在local机器上进行提交，需要指定flink的master信息。
standalone模式提交也是类似，不过可以不用指定master节点，还有个区别就是，提交是在flink集群的机器节点上。
两个提交命令示例：

```Java
# 前台提交
./flink run -p 1 -c com.test.TestLocal /home/hdp/flink-local.jar
# 通过-d后台提交
./flink run -p 1 -c com.test.TestLocal -d /home/hdp/flink-local.jar
```

## 1.3 yarn模式
yarn模式必须确保本机配置了`HADOOP_HOME`环境变量，flink会尝试通过`HADOOP_HOME/etc/hadoop`目录下的配置文件连接yarn。

flink on yarn 有两种提交方式：

- yarn-session：启动一个YARN session(Start a long-running Flink cluster on YARN)
- yarn-cluster：直接在YARN上提交运行Flink作业(Run a Flink job on YARN)

**两者区别**<br>
- 一种是yarn-session，就是把首先启动一个yarn-session当成了一个flink容器，官方说法是flink服务，然后我们提交到yarn上面的全部flink任务全部都是提交到这个服务，也就是容器里面进行运行的。flink任务之间也是独立的，但是都存在于flink服务即容器里面，yarn上只能监测到一个flink服务即容器，无法监测到flink单个任务，需要进入flink服务即容器内部，才可以看到。
- 另一种是yarn-cluster，就是每个把flink任务当成了一个application，就是一个job，在yarn上可以管理，flink任务之间互相是独立的。

### 1.3.1 共用一个 yarn-session

在 YARN 中初始化一个 Flink 集群，初始化好资源，提交的任务都在这个集群执行，共用集群的资源。

这个 Flink集群常驻在 YARN 集群中，要关闭可以手动停止。

### 1.3.2 每个Job启动一个集群

每次提交都会创建一个新的 Flink 集群，Job之间是互相独立的。任务执行完之后集群会注销。

# 2. yarn-session 模式

**`yarn-session.sh + flink run`**
## 2.1 启动常驻的Flink集群


```bash
yarn-session.sh -n 2 -s 4 -qu flink-queue -jm 1024 -tm 4096 -d
```
> 备注："-n"参数在flink1.10中已经不再支持，多余的"-n"参数，会导致后面其他参数没有生效，均使用默认值启动了yarn-session。

## 2.2 查看集群


```bash
yarn application -list
yarn-session.sh -id application_id
```


## 2.3 提交任务

```bash
离线
flink run $FLINK_HOME/examples/batch/WordCount.jar \
-output hdfs:///data/wordcount-result.txt

实时
flink run $FLINK_HOME/examples/streaming/SocketWindowWordCount.jar --hostname 192.168.134.111 --port 9000
```

> 从1.5版本开始，Flink on YARN时的容器数量——即TaskManager数量——将由程序的并行度自动推算，也就是说flink run脚本的-yn/--yarncontainer参数不起作用了。
> 
> 自动推算：Job的最大并行度除以每个TaskManager分配的任务槽数。

## 2.4 关闭集群


```bash
查询
yarn application -list -appStates RUNNING

yarn application -kill Application-Id
```


## 2.5 参数

yarn-session参数

|参数|说明|
|---|---|
|~~-n~~|~~分配多少个Container(taskmanager数量)，必选~~|
|-s|每个TaskManager中Slot的数量|
|-j|Flinkjar文件的路径|
|-jm|JobManager容器的内存（默认值：MB）|
|-tm|每个TaskManager容器的内存（默认值：MB）|
|-nm|在YARN上为应用程序设置自定义名称|
|-d|以分离模式运行，后台执行|
|-D|以配置文件中的设置加载资源|
|-id|指定yarn的任务ID|
|-nl|为YARN应用程序指定YARN节点标签|
|-q|显示可用的YARN资源（内存，内核）|
|-qu|指定YARN队列|
|~~-st~~|~~以流模式启动Flink~~|
|-z|命名空间，用于为HA模式创建Zookeeper子路径|


# 3. yarn-cluster 模式

**`flink run -m yarn-cluster`**

## 3.1 启动集群，提交任务


```bash
离线
flink run -m yarn-cluster -ys 4 -yjm 1024 -ytm 1024 $FLINK_HOME/examples/batch/WordCount.jar

实时
flink run -m yarn-cluster -ys 4 -yqu flink-queue $FLINK_HOME/examples/streaming/SocketWindowWordCount.jar --hostname 192.168.134.111 --port 9000
```

## 3.2 参数

yarn-cluster 参数
|参数|说明|
|---|---|
|-m|指定资源调度系统|
|~~-yn~~|~~container容器个数，TaskManager的数量~~|
|-ys|每个TaskManager中Slot的数量|
|-yjm|JobManager容器的内存(默认值：MB)|
|-ytm|每个TaskManager容器的内存(默认值：MB)|
|-ynm|在YARN上为应用程序设置自定义名称|
|-d|以分离模式运行，后台执行|
|-yD|以配置文件中的设置加载资源|
|-yid|指定yarn的任务ID|
|-ynl|为YARN应用程序指定YARN节点标签|
|-yq|显示可用的YARN资源(内存，内核)|
|-yqu|指定YARN队列|
|~~-yst~~|~~以流模式启动Flink~~|
|-z|命名空间，用于为高可用性模式创建Zookeeper子路径|
|-p|指定运行并行度，即slot数量，可覆盖文件中的配置|
|-c|指定入口类|


# 4. Flink1.10新特性

[Flink1.10新特性解读-(集群和部署)](https://zhuanlan.zhihu.com/p/140447269)

## 4.1 使用`-yarnship`参数时，资源目录和 jar 文件会被添加到classpath中
## 4.2 删除`-yn / -yarncontainer`命令行选项
Flink CLI不再支持已弃用的命令行选项-yn/--yarncontainer，该选项用于指定从YARN启动时的container数量。自从引入FLIP-6以来，这个选项就被弃用了。建议所有Flink用户删除此命令行选项。

从1.5版本开始，Flink on YARN时的容器数量——即TaskManager数量——将由程序的并行度自动推算。

自动推算：Job的最大并行度除以每个TaskManager分配的任务槽数。

## 4.3 删除`-yst / -yarnstreaming`命令行选项
Flink CLI不再支持命令行选项-yst/--yarnstreaming，该选项用于禁用预先分配的内存。建议所有Flink用户删除此命令行选项。
## 4.4 高可用存储目录的修改
高可用目录现在存储在`HA_STORAGE_DIR/HA_CLUSTER_ID`下。

`HA_STORAGE_DIR`路径由`high-availability.storageDir` 参数配置，`HA_CLUSTER_ID`由 `high-availability.cluster-id`配置

> 之前如果启用高可用，ha的存储路径如下，submittedJobGraph和completedCheckpoint直接存储在ha存储路径下。当flink cluster正常完成时是合理的。但是，当YARN application failed或killed时，submittedJobGraph和completedCheckpoint将永远存在。我们甚至不知道它们属于哪个flink cluster(yarn application)。所以将它们移到application子目录中。可以使用一些外部工具来清理这些残留的文件。
> 
> 此外，我们需要在flink cluster完成之前进行清理。
> 
> 之前高可用存储目录结构
> 
> ```bash
> └── <high-availability.storageDir>
>     ├── submittedJobGraph
>     ├                  ├ <jobgraph1>(random named)
>     ├                  ├ <jobgraph2>(random named)
>     ├── completedCheckpoint
>     ├              ├ <checkpoint1>(random named)
>     ├              ├ <checkpoint2>(random named)
>     ├              ├ <checkpoint3>(random named)
>     ├── <high-availability.cluster-id>
>            ├── blob
>                   ├── <blob1>(named as [no_job|job_<job-id>]/blob_<blob-key>)
> ```
> 
> 
> 新高可用存储目录结构
> 
> 
> ```bash
> └── <high-availability.storageDir>
>     ├── <high-availability.cluster-id>
>               ├── submittedJobGraph
>               ├                  ├ <jobgraph1>(random named)
>               ├                  ├ <jobgraph2>(random named)
>               ├── completedCheckpoint
>               ├               ├ <checkpoint1>(random named)
>               ├               ├ <checkpoint2>(random named)
>               ├               ├ <checkpoint1>(random named)
>               ├── blob
>                      ├── <blob1>(named as [no_job|job_<job-id>]/blob_<blob-key>)
> ```

## 4.5 Flink客户端根据配置的类加载策略加载

可配置`parent-first` 或`child-first`类加载。以前，只有集群组件(如task manager或job manager)支持此设置。用户需要显式地配置类加载策略

## 4.6 允许任务在所有taskmanager上均匀分布

当FLIP6(资源调度模型重构)在Flink 1.5.0被推出时，改变了从TaskManagers (TMs)分配slots的方式。不是从所有已注册的TM中平均分配slot，而是倾向于在使用另一个TM之前耗尽一个TM。如果需要使用一种更类似于FLIP6之前的调度策略，其中Flink试图将工作负载分散到所有当前可用的TMs上，参考以下方式配置。

在`flink-conf.yaml`文件中设置`cluster.evenly-spread-out-slots: true`


# 5. 遇到的问题
## 5.1 采用yarn-session方式运行过项目后，再以standalone模式运行项目报错

问题描述：搭建好一个全新flink集群后，在没有运行过flink on yarn之前，直接采用standalone模式运行项目能够成功。命令如下：


```bash
./bin/flink run ./examples/streaming/SocketWindowWordCount.jar --hostname hadoop04 --port 9090
```

但是在运行过一次yarn-session过后，再次用standalone模式以同样的命令运行项目，就会报错：

[![1.png](https://p.pstatp.com/origin/fe460001fcef56b55de6)](https://s1.ax1x.com/2020/09/09/w1JosU.png)

我们知道standalone模式不需要启动yarn，也不会去连接yarn，那么日志显示：发现了yarn的配置文件，flink尝试去连接yarn。由于yarn并没有启动，所以一直在重试。出于好奇，我启动了yarn，再次提交项目，又报了以下错误：

[![2.png](https://p.pstatp.com/origin/fec10002fed98623c92b)](https://s1.ax1x.com/2020/09/09/w1YgOO.png)

日志显示无法找到`application：ApplicationNotFoundException: Application with id 'application_1580903213963_0001' doesn't exist in RM`，这里怎么会有application id呢？百思不得其解。根据日志我输出了`/tmp/.yarn-properties-hdp`文件的内容：

[![3.png](https://p.pstatp.com/origin/1379600021464e67fcf00)](https://s1.ax1x.com/2020/09/09/w1J4zV.png)

发现日志中打印的application id与文件中记录的id相同。我猜测是由于运行过一次yarn-session过后，会在本地留下一个yarn的配置文件`/tmp/.yarn-properties-hdp`，再次以standalone模式提交任务时，flink发现了这个配置文件，便会尝试去连接yarn。
那么如果将这个文件删除，是否就可以再以standalone模式提交任务了呢？
将这个文件删除，关闭yarn，再次以同样的命令提交任务，果然成功了！！！

[![4.png](https://p.pstatp.com/origin/1380400011d0166dc45e7)](https://s1.ax1x.com/2020/09/09/w1JhR0.png)