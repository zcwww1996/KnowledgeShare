[TOC]

参考：https://mp.weixin.qq.com/s/p8XXV9FghfLYH3CCQScEEw

# 1.文档编写目的

本文描述了一次因为Zookeeper的异常导致ResourceManager卡住，从而导致集群所有作业无法提交的问题分析和处理。


*   生产环境

1.CDH and CM version：CDH5.15.1 and CM5.15.1

2.集群启用:Kerbeos+OpenLDAP+Sentry

# 2.异常描述

9月16日17:00左右业务反应hive job无法提交，然后在beeline里面进行如下简单测试，发现卡在以下过程：

1) 第一个query：此query的application id  是application\_1600160921573\_0026

```
select count(*) from co_ft_in_out_bus_dtl;
```

![](https://i.loli.net/2021/03/01/qbfeV9G7THtaomD.png)

2) 第二个query：此query在以下过程等待很久没有产生application\_id,

```
select count(*) from xft_sheet1;
```

beeline 输出卡在以下流程

```
Submitting tokens for job: job_1600165270401_0014INFO  : Kind: HDFS_DELEGATION_TOKEN, Service: ha-hdfs:nameservice1, Ident: (token for hive: HDFS_DELEGATION_TOKEN owner=hive/cmskdc001.cmbcloud.cmbchina.net@CMBCHINA.NET, renewer=yarn, realUser=, issueDate=1600165500389, maxDate=1600770300389, sequenceNumber=20818157, masterKeyId=641)
```

![](https://i.loli.net/2021/03/01/WpljHwPNDXtQkyi.jpg)

3) 此时执行pyspark也慢，但是向HDFS put 大文件的时间长短和之前集群正常状态下没有明显差别，说明HDFS没有变慢。

![](https://i.loli.net/2021/03/01/eCdVsFWDvMluxa8.png)

4) 查看 ResourceManager图表出现 GC 

![](https://i.loli.net/2021/03/01/LDwKnr91buWOlvM.jpg)

问题发生时候 ResourceManager GC time很长，达到9s。

![](https://i.loli.net/2021/03/01/C1LGlqF39YhU4Oe.jpg)

发现9月10号GC time也很长，达到快80s，但是集群没有出现异常，

![](https://i.loli.net/2021/03/01/JdSn7lHwWBUm5X8.jpg)

5) 查看当时 ResourceManager 的JVM 使用不大。

![](https://i.loli.net/2021/03/01/DFo3pbYqXJsTRui.png)

过去7天 ResourceManager 的JVM使用也比较恒定，没有达到ResourceManager JVM配置的4GB峰值。

![](https://i.loli.net/2021/03/01/k4MWEAe6pCfwrD2.jpg)

# 3.异常分析

1) 为了尽快恢复业务，尝试多次滚动重启ResourceManager，发现异常还是无法得到解决。于是查看ResourceManager的日志，并且结合前面测试提交的application\_1600160921573\_0026进行排查问题 。在ResourceManager日志可以看到提交的这个 Job 一直在重复 Recovering【1】。

【1】

```java
2020-09-15 18:09:46,009 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: Recovering app: application_1600162721525_0026 with 0 attempts and final state = NONE...2020-09-15 18:21:31,592 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: Recovering app: application_1600162721525_0026 with 0 attempts and final state = NONE...2020-09-15 18:33:44,648 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: Recovering app: application_1600162721525_0026 with 0 attempts and final state = NONE...2020-09-15 18:45:31,393 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: Recovering app: application_1600162721525_0026 with 0 attempts and final state = NONE...2020-09-15 18:55:21,618 INFO org.apache.hadoop.yarn.server.resourcemanager.rmapp.RMAppImpl: Updating application application_1600162721525_0026 with final state: FAILED
```

![](https://i.loli.net/2021/03/01/3ZMncRIYpkD2QWo.jpg)

2) 同时在Active ResourceManager（cmsnn002）日志中看到如下与Zookeeper相关的报错,通过以下日志我们可以看到由于 Zookeeper 的连接异常导致 Active  ResourceManager进入 Standby 状态【2】:

```java
2020-09-15 16:36:00,882 INFO org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore: Maxed out ZK retries. Giving up!2020-09-15 16:36:00,882 INFO org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore: Watcher event type: None with state:Disconnected for path:null for Service org.apache.hadoop.yarn.server.resourcemanager.recovery.RMStateStore in state org.apache.hadoop.yarn.server.resourcemanager.recovery.RMStateStore: STARTED2020-09-15 16:36:00,882 INFO org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore: ZKRMStateStore Session disconnected2020-09-15 16:36:00,883 WARN org.apache.hadoop.yarn.server.resourcemanager.ResourceManager: Transitioning the resource manager to standby.2020-09-15 16:36:00,921 INFO org.apache.hadoop.ipc.Server: Stopping server on 8032
```

![](https://i.loli.net/2021/03/01/QjOpnhJwL4tsoAc.jpg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnx67xbY28J877CGIHT1U9RHwVjKX3pfG7Ds9IZ22dLGNxiawyObBlJxag/640?wx_fmt=jpeg)

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnxWjfSu261UERMaB66VEmAqEH9BBs63iaF6tashVBb2dB6M4kSos93z4A/640?wx_fmt=jpeg)

3) 但糟糕的是, 当天 16:36 左右, 另一个 ResourceManager（cmsnn001）似乎一直无法进入 Active 状态, (但是由于相关日志缺失, 无法确认具体的原因, 不过很可能也是因为 Zookeeper 失联导致)。这导致本 ResourceManager（cmsnn002）再次成为 Active, 但是已经过了 10 分钟, 相当于这 10 分钟 ResourceManager宕机【3】, 所有任务无法提交。

```
2020-09-15 16:47:09,713 INFO org.apache.hadoop.ha.ActiveStandbyElector: Writing znode /yarn-leader-election/yarnRM/ActiveBreadCrumb to indicate that the local node is the most recent active...2020-09-15 16:47:09,718 INFO org.apache.hadoop.yarn.server.resourcemanager.ResourceManager: Transitioning to active state2020-09-15 16:47:22,879 INFO org.apache.hadoop.yarn.server.resourcemanager.RMAppManager: Recovering 10176 applications
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnxayamVa2PSkia71zwb2ozIMAoQhoqj218EusP0BxRH3ADoQGeBm00rww/640?wx_fmt=jpeg)

4) 结合前面9月10日也看到ResourceManager的GC time异常增大的现象，于是尝试结合9月10日的ResourceManager 日志提取更多有效信息。查看9月15日问题发生时间点和9月10日ResourceManager GC time异常增大时间点时候ResourceManager的日志， 发现都有如下异常【4】，此异常说明ResourceManager  尝试和Zookeeper建立连接，但是进行了1000次（1-1000）次后还是没法和Zookeeper建立连接，于是分别在每个Zookeeper节点进入死循环连接。

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnxaErVIIt4nvkTBgXl1TH7USbOtGkll4nhyvmexAlwInnz9SF31APaBw/640?wx_fmt=jpeg)

5) 于是去查看Zookeeper日志，尝试查找ResourceManager 一直连接不上Zookeeper的原因。在Zookeeper日志中查看到问题发生时间点有如下异常信息【5】：

```
2020-09-15 16:25:43,431 WARN org.apache.zookeeper.server.NIOServerCnxn: Exception causing close of session 0x17187f84f281336 due to java.io.IOException: Len error 14307055
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnxCjibj5iaxLPmfvKSwflkTW8E4jlSxbYNSe7FWllPwAqtztXLk2OtVptw/640?wx_fmt=jpeg)

6) 通过查找资料，在Cloudera官网中有一个Knowledge Base提到了Zookeeper中相似的问题【6】，里面说到此问题和Zookeeper 的Jute Max Buffer参数配置的大小有关。

```
https://my.cloudera.com/knowledge/Active-ResourceManager-Crashes-while-Executing-a-ZooKeeper?id=75670
```

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnx5k3Ma3AAJbSU0NNesudiaueyTF7ibibEdqa2EtIMIkrh7nBo20laYbYEA/640?wx_fmt=png)

7) 通过Knowledge Base的介绍，查看现在集群Zookeeper的Jute Max Buffer值为4MB，此值默认为4MB【7】,而我们在Zookeeper日志中看到Len error 14307055 (≈14MB)的异常,说明集群中Zookeeper接受的数据片段已经远远大于默认的4MB，导致Zookeeper的负载增大，其中在某一时刻导致Active ResourceManager与Zookeeper断开连接后，再也无法再次重新连接上。

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnx2HmibtibhBVoSGibqMGqVSoSvg46O7BbIMWRRTbmBefnq6KfoiblVXwdnA/640?wx_fmt=png)

# 4.异常解决

1) 按照如下操作把Zookeeper的Jute Max Buffer 值由默认的4MB增大到32MB。

(1) 打开Cloudera Manager > Zookeeper > Configuration；

(2) 搜索Jute Max Buffer，把jute.maxbuffer数值由4MB修改为32MB；

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnx24DEiaF5duxNiazJ3JWPQunw2rPfjWZoeQTqicMGZwyz3OvbD47Uk2Bqw/640?wx_fmt=png)

(3) 保存设置，滚动重启Zookeeper服务。

2) 再次在beeline测试发现所有任务都可以正常提交，之前待定的application也逐渐恢复运行，待定container逐渐降低，至此问题解决。

# 5.问题总结

1) 此次故障的原因还是与当时的集群负载有关，因为Zookeeper 的jute.maxbuffer 溢出，导致Zookeeper服务端出错，从而关闭与ResourceManager的连接【1】。而Len error并不是 ResourceManager发送的, 而是 Zookeeper Server 端的报错,因为Zookeeper要返回给 ResourceManager的数据量超过了默认的 4MB 限制所导致，具体的数据量应该与对应的 Znode 的当时情况有关。

```
2020-09-15 16:25:43,431 WARN org.apache.zookeeper.server.NIOServerCnxn: Exception causing close of session 0x17187f84f281336 due to java.io.IOException: Len error 14307055
```

![](https://mmbiz.qpic.cn/mmbiz_jpg/THumz4762QCibbh3RO2ub1GRV3YSGjfnxCjibj5iaxLPmfvKSwflkTW8E4jlSxbYNSe7FWllPwAqtztXLk2OtVptw/640?wx_fmt=jpeg)

2) 一般这个问题可能与正在运行的作业数量、 以及作业的 Attempts数量、应用负载增加,、集群扩容后都可能出现。我们查看此次问题发生时间点和9月10日YARN都有7万多待定container【2】，这对于当时的YARN和Zookeeper负载都是比较大的，这也是导致ResourceManager和Zookeeper断连的原因。

9月15日YARN待定container有7万多：

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnxEZbG9yLJqdxGLoqshMkv4DK71xf583gz4eIVwArwQkLsSJogbJeKKQ/640?wx_fmt=png)

9月10日YARN待定container也有7万多：

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnxQ237IMUzwzRCibzFeCiakibo35INB6H8jpd5j9U64jdibvd3F5F1GgE7eA/640?wx_fmt=png)

过去30天YARN待定container：

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnxedmib3qrfxYCJwwicEHypqPmMUX7pqlHn0BglbJID10UicV4r6jhtZTaQ/640?wx_fmt=png)

3) 通过进一步的调查发现，此故障发生的Lenerror异常其实是Zookeeper的一个bug（ZOOKEEPER-706)【1】导致。YARN 往 Zookeeper中写入的数据既包括application的属性信息，也包括application的 DelegationToken 和每个 attempt 的 diagnostic info等运行时信息，所以大负载的集群很容易出现往 Zookeeper写入大量数据的情况.YARN 在这一方面已经做了重要的修复，具体见YARN-3469、YARN-2962、YARN-7262、YARN-6967【2】。我们的集群是CDH5.15.1，通过调查发现， ResourceManager的这个问题在 CDH5上并没有修复, 虽然 CDH5 已经包括了YARN-3469, 但是根本上解决这个问题至少需要YARN-2962, YARN-7262 和 YARN-6967, 通过优化 znode 的结构来防止大量 znode 平铺导致Zookeeper服务的Len error。所有这些都已经进入 CDH6和后续版本, 但是不会放入 CDH5。另外, 虽然社区已经改进了 YARN, 但由于 YARN 的新功能也在不断加入, Zookeeper的存储需求也在增加, 即使升级到了 CDH6, 还是有可能出现 YARN 到 Zookeeper之间大数据量写入的问题。

【1】

```java
https://my.cloudera.com/knowledge/ERROR-javaioIOException-Len-errorquot-in-Zookeeper-causing?id=275334https://issues.apache.org/jira/browse/ZOOKEEPER-706
```

【2】

```java
https://issues.apache.org/jira/browse/YARN-3469https://issues.apache.org/jira/browse/YARN-2962https://issues.apache.org/jira/browse/YARN-7262http://mail-archives.apache.org/mod_mbox/hadoop-yarn-issues/201708.mbox/%3CJIRA.13093146.1502193666000.123395.1502242800181@Atlassian.JIRA%3E
```

# 6.总结

1) ResourceManager连接 Zookeeper主要是更新它的State-store, 用来存储 YARN 集群的持久状态, 包括所有运行中的 application 和 attempts。这个 state-store 保存在 Zookeeper的/rmstore目录中，我们可以通过 zookeeper-client 访问和读取此数据。当 Zookeeper发生 failover 的时候, 被选举为 Active 的 Zookeeper会读取/rmstore中的数据然后恢复整个 YARN 集群的运行状态；

2) 问题发生当时有看到两个ResourceManager在不断尝试主备切换，但是都没有主备切换成功，因为ResourceManager没有等到任何一个Zookeeper的响应。

(1) 具体就是ACTIVE ResourceManager（cmsnn002）在不断的想连接其中一个Zookeeper,通过ResourceManager中的日志org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore: Retrying operation on ZK. Retry no. 999可以看到其中一个ResourceManager尝试连接了1000次第一个Zookeeper，没得到Zookeeper的响应，然后继续从1开始尝试连接第二个Zookeeper，到了1000次还是没有得到第二个Zookeeper的响应，我们有五个Zookeeper，它依次在五个Zookeeper上不断的重复遍历1000次连接，最后都没有得到Zookeeper的响应。这个过程中我们尝试滚动重启ResourceManager还是没有恢复，相当于ACTIVE  ResourceManager陷入了无法工作的死循环连接Zookeeper状态。

(2) ResourceManager从Zookeeper读取数据的次数，在每个Zookeeper默认读取是1000，只要有一次读取到了Zookeeper数据都能完成主备切换。默认控制ResourceManager从Zookeeper读取数据次数的参数是yarn.resourcemanager.zk-num-retries，默认控制每次的读取时间参数是yarn.resourcemanager.zk-retry-interval-ms。我们可以通过CM的  ResourceManager Advanced Configuration Snippet (Safety Valve) for yarn-site.xml 进行配置修改。但是这两个值的修改对本故障是没有帮助的，因为ZOOKEEPER-706的bug如果不修复的话, 还是有可能出现ResourceManager即便连上了 Zookeeper也是无法读取数据的。

3) 此次故障我们通过把Zookeeper的jute maxbuffer 改为32MB来解决此问题。关于此值的配置大小，我们可以在Zookeeper日志中找到Len error最大的值, 如果小于 32MB, 那么统一建议 32MB。但是此值不能配置得过大，毕竟Zookeeper并不是为了大片段数据存储设计的. 如果应用(或者服务)持续大量往 Zookeeper写数据, 会对磁盘以及 Zookeeper本身之间的同步造成压力, 容易使依赖 Zookeeper的应用(服务)不稳定. 比如说此次的 YARN 故障, 本质上来说也是因为这个原因导致的。  

4) 下次如果再遇到这个问题, 我们可以从CM -> Zookeeper 页面确定当前的 Zookeeper Leader, 然后提取该主机上的Zookeeper日志目录（比如/var/log/zookeeper/）和数据目录(比如/var/lib/zookeeper/version-2)目录进行分析。可以通分析 Zookeeper的数据存储, 然后进一步确定是哪一块数据写入导致了Zookeeper Len error这个问题。通过LogFormatter，来查看zookeeper当时写入了什么数据。具体方法可以参考如下链接【1】。

【1】

```
http://www.openkb.info/2017/01/how-to-read-or-dump-zookeeper.html
```

(1) Zookeeper 的数据先写入数据目录(比如/var/lib/zookeeper/version-2)的transaction log，当总量达到 10 万条记录的时候会自动做快照(snapshot)【2】。所以数据文件不是根据时间来保存的，没法设置数据保存日期。CDH 默认设置是每天清理这个目录, 并且只保留 5 个快照. 如果想需要保留更多的数据, 可以在  CM -> Zookeeper -> Configuration -> Auto Purge Snapshots Retain Count 【3】中设置保存更多的快照，不过这对于定位 Zookeeper 导致的问题可能帮助也不是很大, 因为快照生成的时间不一定是发生问题的时候. 所以如果问题再现的话, 建议直接把当时的整个数据目录打包来分析。

【2】

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnxKCSVw0bzvSHAl3DvmwycSialTEibHXPd3IRhNTzviayk2TWXiaqD9mevbg/640?wx_fmt=png)

【3】

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QCibbh3RO2ub1GRV3YSGjfnxKuxUlibX58I85vOG5JUP1goze0xNuHlMNCdkwFR0mV2BhNPxyf2ujAQ/640?wx_fmt=png)