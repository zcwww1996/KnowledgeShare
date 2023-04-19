[TOC]

文章链接: https://www.hnbian.cn/posts/a1601de4.html

# 1. 测试版本
- flink版本为1.10.3
- ambari版本为2.7.x

# 2. 添加流程
## 2.1 安装 git

```bash
# 检查是否安装 git
[root@node1 ~]# git --help
-bash: git: 未找到命令

# 如果没有安装 git 则需要先安装git
[root@node1 ~]# yum install git
已加载插件：fastestmirror
Repository base is listed more than once in the configuration
Determining fastest mirrors
 * base: mirrors.aliyun.com
 * extras: mirrors.aliyun.com
 * updates: mirrors.aliyun.com
......
软件包 git-1.8.3.1-23.el7_8.x86_64 已安装并且是最新版本
无须任何处理
```

## 2.2 设置 version 变量


```bash
VERSION=`hdp-select status hadoop-client | sed 's/hadoop-client - \([0-9]\.[0-9]\).*/\1/'` 
[root@node1 opt]# echo $VERSION
3.1
```

## 2.3 下载ambari-flink-service服务

```bash
# 下载ambari-flink-service服务到 ambari-server 资源目录下
[root@node1 opt]# sudo git clone https://github.com/abajwa-hw/ambari-flink-service.git   /var/lib/ambari-server/resources/stacks/HDP/$VERSION/services/FLINK
正克隆到 '/var/lib/ambari-server/resources/stacks/HDP/3.1/services/FLINK'...
remote: Enumerating objects: 192, done.
remote: Total 192 (delta 0), reused 0 (delta 0), pack-reused 192
接收对象中: 100% (192/192), 2.08 MiB | 35.00 KiB/s, done.
处理 delta 中: 100% (89/89), done.

[root@node1 opt]# cd /var/lib/ambari-server/resources/stacks/HDP/3.1/services/FLINK/
[root@node1 FLINK]# ll
总用量 20
drwxr-xr-x 2 root root   58 2月  23 16:16 configuration
-rw-r--r-- 1 root root  223 2月  23 16:16 kerberos.json
-rw-r--r-- 1 root root 1777 2月  23 16:16 metainfo.xml
drwxr-xr-x 3 root root   21 2月  23 16:16 package
-rwxr-xr-x 1 root root 8114 2月  23 16:16 README.md
-rw-r--r-- 1 root root  125 2月  23 16:16 role_command_order.json
drwxr-xr-x 2 root root  236 2月  23 16:16 screenshots
```

## 2.4 修改配置文件

1. 编辑 metainfo.xml 将安装的版本修改为 1.10.3

```bash
 [root@node1 FLINK]# vim metainfo.xml

<name>FLINK</name>
<displayName>Flink</displayName>
<comment>Apache Flink is a streaming ...</comment>
<version>1.10.3</version>
```

2. 配置 Flink on yarn 故障转移方式

设置完，重启yarn集群


flink在yarn上可以直接运行起来
```xml
<property>
    <name>yarn.client.failover-proxy-provider</name>
    <value>org.apache.hadoop.yarn.client.ConfiguredRMFailoverProxyProvider</value>
</property>
```

flink在yarn上无法运行
```xml
<property>
    <name>yarn.client.failover-proxy-provider</name>
    <value>org.apache.hadoop.yarn.client.RequestHedgingRMFailoverProxyProvider</value>
</property>
```


3. 编辑configuration/flink-ambari-config.xml修改下载地址


```xml
 [root@node1 FLINK]# vim configuration/flink-ambari-config.xml

<property>
    <name>flink_download_url</name>
    <!--<value>http://www.us.apache.org/dist/flink/flink-1.9.1/flink-1.9.1-bin-scala_2.11.tgz</value>-->
    <!--<value>https://downloads.apache.org/flink/flink-1.10.3/flink-1.10.3-bin-scala_2.11.tgz</value>-->
    <value>http://192.169.12.1/Package/flink-1.10.3-bin-scala_2.11.tgz</value>
    <description>Snapshot download location. Downloaded when setup_prebuilt is true</description>
  </property>
```

`http://192.169.12.1/Package/flink-1.9.0-bin-scala_2.11.tgz`  这里我使用的是本地路径, 推荐使用[https://archive.apache.org/dist/](https://archive.apache.org/dist/),选择的自己的版本将连接填充完整，其他两个国内可以访问的有[http://www.us.apache.org/dist/flink/](http://www.us.apache.org/dist/flink/) ， [https://archive.apache.org/dist/](https://archive.apache.org/dist/)

## 2.5 创建 Flink 用户组与用户

```bash
# 添加用户组
[root@node1 FLINK]# groupadd flink
# 添加用户
[root@node1 FLINK]# useradd  -d /home/flink  -g flink flink
```

## 2.6 重启 ambari-server 并检查可用服务列表


```bash
[root@node1 ~]# ambari-server restart
Using python  /usr/bin/python
Restarting ambari-server
Waiting for server stop...
Ambari Server stopped
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start.............................................
Server started listening on 8080
```

*   查看安装服务中是否有 Flink

**Flink 已经添加到可用服务列表**<br>
![](https://images.hnbian.cn/Ftw8NjJE4q1C0ZzShzs7ZVLFWvIi)


## 2.7 安装-flink

**1.选择 Flink 添加服务**<br>
![](https://images.hnbian.cn/Fksq7hs8NlBm_OLVB6wTvwHQdPyE)


**2.确认依赖服务是否安装**<br>
![](https://images.hnbian.cn/FsM_6t2O9UCeL0V57QuIJm6SDR3f)


**3.选择安装服务到哪台服务器**<br>
![](https://images.hnbian.cn/Frgk2_MnHWiabOHQtLcArAkK_ovD)


**4.修改 flink安装路径**<br>
![](https://images.hnbian.cn/FoeP9IjnwHbqv3ggcNcw31HY2Ql6)


**5.安装与测试**<br>
![](https://images.hnbian.cn/FnoOc_6vFXNnE888gRcw8m5DbqZl)


**6.安装完成**<br>
![](https://images.hnbian.cn/FhdDHFX4yKyEcX5gC9r7l8bogMQb)


**7.摘要**<br>
![](https://images.hnbian.cn/FtHIlum6Le84QomnKst5nlJSQ1uP)


**8.查看 ambari 中安装好的flink**<br>
![](https://images.hnbian.cn/Ftlrn82KZzuIBldLI2pK8FDxhCYL)


**9.点击上图的配置按钮，修改 Javahome 路径**<br>
![](https://images.hnbian.cn/FqjT7S90W5mI4W1bnVA6o0NkPsS_)


**10.增加堆内存**
![](https://images.hnbian.cn/FrDr8O2Et4c9LTJowb7-sujQ3AdJ)

**11.配置hadodp_conf_dir**
![yarn配置](https://img-blog.csdnimg.cn/2020012011303253.png)

`javahome` 与`/etc/prfile`一致即可，`hadodp_conf_dir`使用hdp的生成dir即可

## 2.8 启动 flink 服务

**启动 flink 服务**<br>
![](https://images.hnbian.cn/FosKLtIFht9fgpZgnO0VwXq7LJJ9)

**yarn 中运行的 Flink yarn session**<br>
![](https://images.hnbian.cn/FgcAYlHU0rJBNVzfYrUPVaALCOIV)

**Flink web UI**<br>
![](https://images.hnbian.cn/FqLl2u5DvnA1v2L9znoZVkJxVZLP)

## 2.9 测试 flink


*   提交任务

```bash
[hdfs@node1 flink-1.10.2]$ bin/flink run ./examples/batch/WordCount.jar # 提交 flink 自带的测试任务
Setting HADOOP_CONF_DIR=/etc/hadoop/conf because no HADOOP_CONF_DIR was set.
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/opt/flink-1.10.2/lib/slf4j-log4j12-
...
# 执行结果
(a,5)
(action,1)
(after,1)
(against,1)
(all,2)
(and,12)
(arms,1)
(arrows,1)
(awry,1)
(ay,1)
(bare,1)
(be,4)
(bear,3)
...
```

*   查看 Flink webUI 中的任务情况

**提交任务之后马上进Flink webUI 能看到 Wordcount 任务正在运行**<br>
![](https://images.hnbian.cn/FisyxUz1XvX9YPflTVmZ0nYyfD2O)


**查看任务运行流程**<br>
![查看任务运行流程](https://images.hnbian.cn/FsVdDr-EvUFnt1KVkDu_bEeXwTq_)


# 3. 遇到的问题

## 3.1 启动时权限问题

1. **hdfs权限没有读写权限**

```bash
Caused by: org.apache.hadoop.security.AccessControlException: Permission denied: user=flink, access=EXECUTE, inode="/user/flink/.flink/application_1579166472849_0003":root:hdfs:drwxrwx---
```
![5KwHc4.png](https://z3.ax1x.com/2021/10/13/5KwHc4.png)

解决：

使用hdfs用户，/etc/profile添加：`export HADOOP_USER_NAME=hdfs`（hdfs为最高权限）

`source /etc/profile`（记得执行，以保证立即生效）

也可以执行 `sed -i '$a export HADOOP_USER_NAME=hdfs'` ,记得也要`source /etc/profile`一下


2. **本地没有读写权限**

```bash
Execution of 'yarn application -list 2>/dev/null | awk '/flinkapp-from-ambari/ {print $1}' | head -n1 > /var/run/flink/flink.pid' returned 1. -bash: /var/run/flink/flink.pid: Permission denied
awk: (FILENAME=- FNR=4) warning: error writing standard output (Broken pipe)
```

![5Kw73F.png](https://z3.ax1x.com/2021/10/13/5Kw73F.png)

解决：

使用root用户，修改 `flink.py`


```bash
# flink.py路径
pwd
/var/lib/ambari-server/resources/stacks/HDP/2.6/services/FLINK/package/scripts

vim flink.py
Execute("yarn application -list 2>/dev/null | awk '/" + params.flink_appname + "/ {print $1}' | head -n1 > " + status_params.flink_pid_file, user='root')

```


## 3.2 启动时 java: 未找到命令

```java
resource_management.core.exceptions.ExecutionFailed: Execution of 'export ...9.3.22.v20171030.jar:/usr/hdp/3.1.0.0-78/tez/lib/jsr305-3.0.0.jar:/usr/hdp/3.1.0.0-78/tez/lib/metrics-core-3.1.0.jar:/usr/hdp/3.1.0.0-78/tez/lib/protobuf-java-2.5.0.jar:/usr/hdp/3.1.0.0-78/tez/lib/servlet-api-2.5.jar:/usr/hdp/3.1.0.0-78/tez/lib/slf4j-api-1.7.10.jar:/usr/hdp/3.1.0.0-78/tez/lib/tez.tar.gz; /usr/hdp/3.1.0.0-78/flink/bin/yarn-session.sh -n 1 -s 1 -jm 768 -tm
-qu default -nm flinkapp-from-ambari -d >> /var/log/flink/flink-setup.log' returned 127. /usr/hdp/3.1.0.0-78/flink/bin/yarn-session.sh:行37: java: 未找到命令
```

解决办法：

![修改 Javahome 路径](https://images.hnbian.cn/FqjT7S90W5mI4W1bnVA6o0NkPsS_)

## 3.3 测试时 IllegalConfigurationException:

```java
org.apache.flink.configuration.IllegalConfigurationException: Sum of configured Framework Heap Memory (128.000mb (134217728 bytes)), Framework Off-Heap Memory (128.000mb (134217728 bytes)), Task Off-Heap Memory (0 bytes), Managed Memory (25.600mb (26843546 bytes)) and Network Memory (64.000mb (67108864 bytes)) exceed configured Total Flink Memory (64.000mb (67108864 bytes)).
    at org.apache.flink.runtime.clusterframework.TaskExecutorProcessUtils.deriveInternalMemoryFromTotalFlinkMemory(TaskExecutorProcessUtils.java:320)
    at org.apache.flink.runtime.clusterframework.TaskExecutorProcessUtils.deriveProcessSpecWithTotalProcessMemory(TaskExecutorProcessUtils.java:248)
    at org.apache.flink.runtime.clusterframework.TaskExecutorProcessUtils.processSpecFromConfig(TaskExecutorProcessUtils.java:147)
    at org.apache.flink.client.deployment.AbstractContainerizedClusterClientFactory.getClusterSpecification(AbstractContainerizedClusterClientFactory.java:44)
    at org.apache.flink.yarn.cli.FlinkYarnSessionCli.run(FlinkYarnSessionCli.java:547)
    at org.apache.flink.yarn.cli.FlinkYarnSessionCli.lambda$main$5(FlinkYarnSessionCli.java:786)
    at java.security.AccessController.doPrivileged(Native Method)
    at javax.security.auth.Subject.doAs(Subject.java:422)
    at org.apache.hadoop.security.UserGroupInformation.doAs(UserGroupInformation.java:1730)
    at org.apache.flink.runtime.security.HadoopSecurityContext.runSecured(HadoopSecurityContext.java:41)
    at org.apache.flink.yarn.cli.FlinkYarnSessionCli.main(FlinkYarnSessionCli.java:786)
```

解决办法：

![增加堆内存](https://images.hnbian.cn/FrDr8O2Et4c9LTJowb7-sujQ3AdJ)


## 3.4 测试时FeaturesAndProperties

```bash
Caused by: java.lang.ClassNotFoundException: com.sun.jersey.core.util.FeaturesAndProperties
```

解决:

修改yarn-site.xml中的`yarn.timeline-service.enabled`设置为false，修改后如图

![image](https://img-blog.csdnimg.cn/20200119175021139.png)

## 3.5 页面显示异常

从页面启动 flink之后，Flink-yarn-session 启动成功，但是 ambari 进度不到 100%

## 3.6 页面没有 Flink web ui 链接地址

