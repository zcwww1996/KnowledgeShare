[TOC]

参考：https://docs.ksyun.com/documents/1278

https://docs.microsoft.com/zh-cn/azure/hdinsight/hdinsight-hadoop-manage-ambari
# 一、Dashboard（仪表盘，总览页面）


## 1.1 总览

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095029416-61569670.png)

【集群操作】

![image](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095037810-1543829694.png)

【配置文件下载】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095043498-1834888134.png)


【图表操作】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095051299-605339532.png)

【图表时间配置】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095103859-923481780.png)


【集群总体监控图表】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095205395-174578598.png)

- Memory Usage：整个集群的内存使用情况，包括 cached，swapped，used，和shared。

- Network usage：整个就群的网络流量，包括上行和下行；

- CPU Usage：集群的CPU使用情况；

- Cluster Load：集群整体加载信息，包括节点数目，总CPU个数，正在运行的进程
  
## 1.2 HDFS层面

【HDFS Disk Usage】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095309005-401466350.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095252203-1748852856.png)



左图：整个集群的磁盘使用情况。

右图：DFS的使用情况；non DFS的使用情况；磁盘实际剩余空间。

**总共：100G空间。**

**如果配置了dfs.datanode.du.reserved = 30G。**

**那么，HDFS可以理所应当的占据70GB的空间。**

**这个时候，如果系统文件或者其他文件已经使用了40GB。**

**那么就意味着，最多给HDFS的空间只剩下60GB了！**

**本来讲道理，HDFS有70GB的空间可以挥霍，但是现在空间只有60GB。**

**是不是说，有10GB应当给HDFS用的空间，却被其他东西使用了？**

**这个10GB的空间，就是Non - DFS！**

**如果dfs.datanode.du.reserved配置了0GB。**

**那么就意味着，只要不是HDFS使用的空间，都是NonDFS！！**

【NameNode Heap】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095356893-1496145624.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095359835-1939621746.png)



NameNode的JVM堆使用情况。

【NameNode CPU WIO】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095407758-848410340.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095413959-402392457.png)



NameNode节点的CPU WIO。表示CPU空闲等待IO的情况，参数越高，说明CPU在长时间等待磁盘、网络等IO的操作而空闲。IO瓶颈较大。

【NameNode RPC】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095419530-1714138437.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095422263-1085122343.png)

RPC请求在队列中的平均滞留时间。

【NameNode Uptime】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095433573-152991745.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095436824-192997290.png)

NameNode累计上线时间，以及上线时间点。

【DataNodes Live】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095523163-151897891.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095525939-129591831.png)


DataNode的状态。

【HDFS Links】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095540651-1884245351.png)


HDFS相关页面的快速链接。 

## 1.3 Yarn 层面

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095557842-503874907.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095601933-1877899800.png)


YARN Memory：Yarn集群的内存使用率。

【ResourceManager Heap】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095615616-996858889.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095619940-607554452.png)


RM的JVM堆使用情况。

【ResourceManager Uptime】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095628479-1567925995.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095630887-141490428.png)

RM累计上线时间，以及上线时间点。

【NodeManagers Live】

NM的节点状态监控。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095646027-1849855433.png)![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095648393-569281510.png)

【节点热力图】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095654160-978781059.png)


【服务参数版本管理】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095702097-1327013493.png)


## 1.4 查看操作

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095712957-852095265.jpg)


【查看告警】


![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095720940-1810961288.png)

# 二、服务面板


下面是HDFS的主面板，其他的类似。


![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095732789-485688063.png)

# 三、参数配置、组、版本


![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095756141-1280588521.png)

【服务配置版本与组的时间上关系】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095805105-1424588027.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095812482-928380221.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095816613-1579590261.png)

**可以把Default理解为主版本（master版本），默认所有的节点配置都是按照这个来。**

**可以对这个主版本创建一个分支，也就是创建一个group。group中存储额外override覆盖的参数。**

**group中的参数会在哪个节点中生效取决于该group中配置了哪些host。**

在默认的Default组的config面板中，参数都可以直接修改，这里改的是master主版本的配置。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095844184-485582964.png)

核心参数不允许Override。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095852356-1981029207.png)

也可以Override这个参数，一旦点击，就会提示说在哪个group中改这个参数。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095913281-2046584187.png)

在分支组中的配置面板如下：<br>
![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816095925105-535840138.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100001932-1289197579.png)


# 四、Host主机管理

主机列表视图：

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100030032-1796278264.png)

主机视图：

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100037312-1076381391.png)

# 五、告警管理

告警列表视图：

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100050327-1312842892.png)

告警详情：

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100100171-803041635.png)

# 六、Ambari管理

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100114173-1534210682.png)  
总体界面：


![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816100120957-1167921743.png)

【自定义页面管理】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161449669-903921957.png)

【用户和用户组角色分配】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161520871-268560918.png)

【角色权限列表】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161536205-1392686891.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161541505-607235318.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161547295-1488251951.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161612310-800031514.png)

# 七、扩展页面

【Yarn队列管理】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161702070-1500174599.png)

【HDFS】文件管理

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161719943-1824749597.png)

# 七、AMS（Ambari Metrics System）

AMS包括4个部分：

**Metrics Monitors：** 在各个节点中收集系统级别的度量参数，然后推送给Metrics Collector。

**Hadoop Sinks：** 内嵌在Hadoop的各个组件中，将Hadoop的度量参数推送给Metrics Collector。

**Metrics Collector：** 一个守护进程，运行在特定的节点中，用来接收已经注册的“Publisher”的数据。

**Grafana：** 开源的度量分析和可视化套件。数据源为Collector。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161835603-1932081881.png)

【AMS架构图】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161855615-2018003559.jpg)

【访问Grafana界面】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161927636-1581455242.png)

默认端口号是3000。

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161940908-225624661.png)

【Grafana简单操作】

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816161951009-286505754.png)

![](https://images2018.cnblogs.com/blog/1076786/201808/1076786-20180816162000466-771419297.png)

