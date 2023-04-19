[TOC]
# 1. elasticsearch简介

ElasticSearch是一个基于Lucene的搜索服务器。它提供了一个分布式多用户能力的全文搜索引擎，基于RESTful web接口。Elasticsearch是用Java开发的，并作为Apache许可条款下的开放源码发布，是当前流行的企业级搜索引擎。设计用于云计算中，能够达到实时搜索，稳定，可靠，快速，安装使用方便。

**来源**： http://blog.java1234.com/blog/articles/341.html

Elasticsearch是一个基于Lucene构建的开源、分布式、RESTful的搜索引擎，能够实现近实时（NRT）搜索，稳定、可靠、安装方便。

Elasticsearch是一个近实时的系统，从你写入数据到数据可以被检索到，一般会有1秒钟的延时

## 1.1 Lucene
Lucene是apache软件基金会4 jakarta项目组的一个子项目，是一个开放源代码的全文检索引擎工具包，但它不是一个完整的全文检索引擎，而是一个全文检索引擎的架构，提供了完整的查询引擎和索引引擎，部分文本分析引擎。Lucene的目的是为软件开发人员提供一个简单易用的工具包，以方便的在目标系统中实现全文检索的功能，或者是以此为基础建立起完整的全文检索引擎。
## 1.2 名词解释

|常用术语|关系型数据库|Elasticsearch|
|:---:|:---:|:---:|
|数据库|database|index|
表|table|type|
|行 |row|document|
|列|column|field|


（1）Cluster：集群。
Index：索引，Index相当于关系型数据库的**DataBase**。

（2）Type：相当于关系型数据库的<font style="background-color: #FFFF00
;">table</font>

（3）Document：文档，Json结构，这点跟MongoDB差不多。

（4）Shard、Replica：分片，副本。

分片的好处，一个是可以水平扩展，另一个是可以并发提高性能。

副本：ES支持把分片拷贝出一份或者多份，称为副本分片，简称副本。

副本的好处，一个是实现高可用（HA，High Availability），另一个是利用副本提高并发检索性能。

==分片和副本的数量可以在创建index的时候指定，index创建之后，只能修改副本数量，不能修改分==片
## 1.2 集群健康状态
安装了head插件之后，可以在web上看到集群健康状态，集群处于绿色表示当前一切正常，集群处于黄色表示当前有些副本不正常，集群处于红色表示部分数据无法正常提供。绿色和黄色状态下，集群都是能提供完整数据的，红色状态下集群提供的数据是有缺失的。

# 2. Elasticsearch集群安装方法
## 2.1 安装前提
### 2.1.1 明确安装环境
准备几台机器，明确ip，用户密码等信息
```bash
192.168.111.101  Linux01
192.168.111.102  Linux02
192.168.111.103  Linux03
```
### 2.1.2 各个机器节点可以ping通。
### 2.1.3 每个节点均安装x-pack。
## 2.2 安装方法
以上述机器地址为例，进入Linux01, 然后执行以下操作
### 2.2.1 新建一个用户elk
1. 新建elk用户并设置密码
```bash
useradd elk & passwd elk
```

2. 切换成elk
```bash
su elk
cd /home/elk  显示结果为：drwx------. 2 elk elk 4096 Dec 10 18:49 elk
```
注意：在启动elasticsearch的之前，需要切换到elk用户，elk用户使用完之后要记得退出
```bash
exit
```
### 2.2.2 下载elasticsearch-7.3.2-linux-x86_64.tar.gz 包 
下载网址为：https://www.elastic.co/cn/downloads/past-releases#elasticsearch

1. 解压tar包
```bash
tar -zxvf elasticsearch-7.3.2-linux-x86_64.tar.gz
```
2. 新建数据存储目录
```bash
mkdir -p  /usr/local/elasticsearch-7.3.2/data
```
3. 新建日志存储目录,使用默认
```bash
/usr/local/elasticsearch-7.3.2/logs
```
### 2.2.3 编辑配置文件（config/elasticsearch.yml）
- 集群名称
```bash
cluster.name: elasticsearch_cluster
```

- 节点名称
```bash
node.name: "Linux01"
```
- 是否有master资格
```bash
node.master: true
```
- 是否存储数据
```bash
node.data: true
```
- 数据存储目录
```bash
path.data: /usr/local/elasticsearch-7.3.2/data
```
- 日志目录
```bash
path.logs: /usr/local/elasticsearch-7.3.2/logs
```
- 设置节点域名
```bash
network.host: 172.16.100.221
```
- 对外提供服务的http端口，默认为9200
```bash
http.port: 9200
```
- 设置集群其它节点地址
```bash
discovery.zen.ping.unicast.hosts: ["192.168.111.101", "192.168.111.102", "192.168.111.103"]
```

- 下面这个参数控制的是，一个节点需要看到的具有master节点资格的最小数量，然后才能在集群中做操作。官方推荐值是(N/2)+1；其中N是具有master资格的节点的数量（我们的情况是3，因此这个参数设置为2)但是：但对于只有2个节点的情况，设置为2就有些问题了，一个节点DOWN掉后，肯定连不上2台服务器了，这点需要注意
```bash
discovery.zen.minimum_master_nodes: 2
```
- ES默认开启了内存地址锁定，为了避免内存交换提高性能。但是Centos6不支持SecComp功能，启动会报错，所以需要将其设置为false
```bash
bootstrap.memory_lock: false
bootstrap.system_call_filter: false
```
**注意： 如果需要设置es占用的内存资源, 去config目录下jvm.options文件中配置**

### 2.2.4 修改文件所有者
```bash
chown -R elk:elk /usr/local/elasticsearch-7.3.2/*
```
### 2.2.5 配置完毕后, 把elasticsearch目录scp至各个节点中, 并修改node.name和network.host这两个选项
```bash
cd /usr/local/
scp -r elasticsearch-7.3.2 Linux02:$PWD
scp -r elasticsearch-7.3.2 Linux02:$PWD
```
### 2.2.6 分别去各个节点的bin目录中执行文件elasticsearch
**用nohup执行**
```bash
nohup ./elasticsearch >/usr/local/elasticsearch-7.3.2/logs/es.log 2>&1 &
```
### 2.2.7 验证：http:IP地址//:9200/

##  2.3 遇到问题的解决方法
### 2.3.1 max number of threads [1024] for user [elasticsearch] is too low, increase to at least [4096]
用root权限修改其他用户的内存限制。不一定是90-nproc.conf，是“数字-nproc.conf”
```bash
vim /etc/security/limits.d/90-nproc.conf
```
把*对应的限制大小1024 改为 unlimited
### 2.3.2 虚拟内存大小不够
max virtual memory areas vm.max_map_count [65530] likely too low, increase to at least [262144]
增加虚拟内存的大小
```bash
echo "vm.max_map_count = 262144" >> /etc/sysctl.conf & sysctl -p
```

### 2.3.3 max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
每个进程最大同时打开文件数太小，可通过下面2个命令查看当前数量
```bash
ulimit -Hn
ulimit -Sn
```
修改/etc/security/limits.conf文件，增加配置，用户退出后重新登录生效:
vi /etc/security/limits.conf
添加：
```bash
*               soft    nofile          65536
*               hard    nofile          65536
```

### 2.3.4 Creating mailbox file: File exists
```bash
rm -rf /var/spool/mail/用户名
userdel -r 用户名
```

### 2.3.5 内存不是root用户被限制
https://blog.csdn.net/fanshukui/article/details/100932732