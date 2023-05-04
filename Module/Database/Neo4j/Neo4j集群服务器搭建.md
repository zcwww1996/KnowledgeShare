[TOC]

参考：<br>
https://blog.csdn.net/thinkercode/article/details/46472209

# 1. 什么是图形数据库

图形数据库（graphic database）是利用计算机将点、线、画霹图形基本元素按一定数据结同灶行存储的数据集合。  

图形数据库将地图与其它类型的平面图中的图形描述为点、线、面等基本元素，并将这些图形元素按一定数据结构（通常为拓扑数据结构）建立起来的数据集合。包括两个层次：第一层次为拓扑编码的数据集合，由描述点、线、面等图形元素间关系的数据文件组成，包括多边形文件、线段文件、结点文件等。文件间通过关联数据项相互联系；第二层次为坐标编码数据集合，由描述各图形元素空间位置的坐标文件组成。图形数据库是地理信息系统中对矢量结构地图数字化数据进行组织的主要形式。  

Neo4j是一个用Java实现的、高性能的、NoSQL图形数据库。Neo4j 使用图（graph）相关的概念来描述数据模型，通过图中的节点和节点的关系来建模。Neo4j完全兼容ACID的事务性。Neo4j以“节点空间”来表达领域数据，相对于传统的关系型数据库的表、行和列来说，节点空间可以更好地存储由节点关系和属性构成的网络，如社交网络，朋友圈等。  

由Neo4j构建“图”模型，也可以准确表达 数据库模型，key-value模型，文档模型的数据关系。

# 2. 高可用架构

高可用性（High Availability）特征只能在Neo4j 企业版中可用，Neo4j High Availability 或者 Neo4j HA 提供以下两点主要特征：  
1) 可以是一个使用多台neo4j从数据库设置可以替代单台neo4j主数据库的容错架构数据库，这可以在硬件设备损坏的情况下使数据库具备完善的功能和读写操作的能力。  
2) 系统具有比单台neo4j数据库处理更多的读取负载处理的横向扫描主读架构。

Neo4j HA 设计的目的是为了从一台到多台机器transition的操作简单，而不需要在已存在的应用中做任何更改。当neo4j数据库以HA模式运行时，总是会有一个单台的主机（master），零个或者更多的从机（slave）。与其他主从复制设置相对比，Neo4j可以从从节点（slave）处理写入，所以不需要从主节点直接写入。为保持数据一致性，从节点（slave）将于主节点（master）同步处理写入。然而更新操作最终从主节点（master）传播到从节点（slave），所以一个从节点的写入在其他所有从节点上不是立即可见的。这是多台设备与单台设备在Neo4j HA模式下运行操作的唯一区别，所有其他ACID特征都一样。从节点写入数据同步效率极低，不建议直接往从节点写入，自己在测试期间也发现很多问题。

[![Neo4j HA](https://www.z4a.net/images/2023/05/04/20150612152352236.jpg)](https://img-blog.csdn.net/20150612152352236)


# 3. 系统环境

操作系统：
- CentOS-6.5-x86\_64  
- JDK 1.8.0  
- Neo4j enterprise-2.2.2  
  - Neo4j-1 192.168.10.133  
  - Neo4j-2 192.168.10.134  
  - Neo4j-3 192.168.10.135  
  - Neo4j-4 192.168.10.136  
- Haproxy 192.168.10.141  

部署方式：先以Neo4j-1、Neo4j-2、Neo4j-3搭建集群，之后不重启加入Neo4j-4

# 3. 服务搭建

## 4.1 4台服务器全部安装JDK

Neo4对应jdk版本
- Neo4j 3.5.X
  - Jdk 8
- Neo4j 4.X.Y
  - Jdk 14


## 4.2 安装Neo4j

```bash
[root@CentOS ~]# yum install -y lsof
[root@CentOS ~]# tar xzf neo4j-enterprise-2.2.2-unix.tar.gz
[root@CentOS ~]# mv neo4j-enterprise-2.2.2 neo4j
[root@CentOS bin]# chmod +x neo4j/bin/*
```

## 4.3 服务配置

> ==**如果在已有的服务器上配置集群，只能有一条服务器上可以有数据，其他服务器的data/graph.db目录必须清空**==  
> 调整Neo4j-1的neo4j-server.properties配置文件

```bash
dbms.security.auth_enabled=false
org.neo4j.server.database.mode=HA
org.neo4j.server.webserver.address=0.0.0.0
```

调整Neo4j-1的neo4j.properties配置文件

```bash
remote_shell_enabled=true
remote_shell_host=127.0.0.1
remote_shell_port=1337
ha.server_id=1
ha.initial_hosts=192.168.10.133:5001,192.168.10.134:5001,192.168.10.135:5001
ha.cluster_server=192.168.10.133:5001
ha.server=192.168.10.133:6001
ha.pull_interval=10
```

调整Neo4j-2的neo4j-server.properties配置文件

```bash
dbms.security.auth_enabled=false
org.neo4j.server.database.mode=HA
org.neo4j.server.webserver.address=0.0.0.0
```

调整Neo4j-2的neo4j.properties配置文件

```bash
remote_shell_enabled=true
remote_shell_host=127.0.0.1
remote_shell_port=1337
ha.server_id=2
ha.initial_hosts=192.168.10.133:5001,192.168.10.134:5001,192.168.10.135:5001
ha.cluster_server=192.168.10.134:5001
ha.server=192.168.10.134:6001
ha.pull_interval=10
```

调整Neo4j-3的neo4j-server.properties配置文件

```bash
dbms.security.auth_enabled=false
org.neo4j.server.database.mode=HA
org.neo4j.server.webserver.address=0.0.0.0
```

调整Neo4j-3的neo4j.properties配置文件

```bash
remote_shell_enabled=true
remote_shell_host=127.0.0.1
remote_shell_port=1337
ha.server_id=3
ha.initial_hosts=192.168.10.133:5001,192.168.10.134:5001,192.168.10.135:5001
ha.cluster_server=192.168.10.135:5001
ha.server=192.168.10.135:6001
ha.pull_interval=10
```

## 4.4 启动服务

```bash
Neo4j-1[root@CentOS ~]# neo4j/bin/neo4j start
Neo4j-2[root@CentOS ~]# neo4j/bin/neo4j start
Neo4j-3[root@CentOS ~]# neo4j/bin/neo4j start
```

> 如果无法启动服务就把iptables和selinux关闭，或者设置防火请规则。

## 4.5 动态添加一台服务器到集群  
调整Neo4j-4的neo4j-server.properties配置文件

```bash
dbms.security.auth_enabled=false
org.neo4j.server.database.mode=HA
org.neo4j.server.webserver.address=0.0.0.0
```

调整Neo4j-4的neo4j.properties配置文件

```bash
remote_shell_enabled=true
remote_shell_host=127.0.0.1
remote_shell_port=1337
ha.server_id=4
ha.initial_hosts=192.168.10.133:5001,192.168.10.134:5001,192.168.10.135:5001,192.168.10.136:5001
ha.cluster_server=192.168.10.136:5001
ha.server=192.168.10.136:6001
ha.pull_interval=10
```

## 4.6 启动Neo4j-4服务

```bash
bin/neo4j start
```

> 此时集群中已经有了四台服务器，其中一台主库，3台从库

![](https://img-blog.csdn.net/20150612152431752)

# 5. 一主多重模式

## 5.1 Haproxy下载与安装

```bash
[root@localhost ~]# wget http://haproxy.1wt.eu/download/1.4/src/haproxy-1.4.20.tar.gz
[root@localhost ~]# tar xzvf haproxy-1.4.20.tar.gz
[root@localhost ~]# cd haproxy-1.4.20
[root@localhost ~]# uname -a
[root@localhost ~]# make TARGET=linux26
[root@localhost ~]# make install
```

## 5.2 配置Haproxy  
把所有从库配置在Haproxy中，通过程序区分主从服务

```bash
vim /usr/local/sbin/haproxy.cfg

global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:7474
    default_backend neo4j-slaves

backend neo4j-slaves
    option httpchk GET /db/manage/server/ha/slave
    server s1 192.168.10.134:7474 maxconn 32 check
    server s2 192.168.10.135:7474 maxconn 32 check
    server s3 192.168.10.136:7474 maxconn 32 check
```

## 5.3 启动服务

```bash
/usr/local/sbin/haproxy -f /usr/local/sbin/haproxy.cfg &
```

> 应用程序在写数据时连接192.168.10.133:7474服务器，再读数据时连接192.168.10.141:7474

# 6. master自动选举模式

> Neo4j支持自动推荐master，当master出现故障的时候，会自动选举一台slave为master，由于Neo4j并没有MongoDB的mongos路由服务，导致这种高可用模式只能通过负载均衡器来实现。上述方式当master挂了，Neo4j会自动推举一台slave为master，但是应用程序并不知道，所以并没有用到该特性。实现该特性需要利用两个负载均衡器，一个发送请求到master，一个发送请求到slave，请求类型的区分就要在应用程序端来实现。这里的取舍，虽然Neo4j允许在slave写入，这种间接关系增加了不必要的写请求延迟，通过这种模式可以区分读写，且能实现自动推举master功能。

## 6.1 配置Haproxy

```bash
global
    daemon
    maxconn 256

defaults
    mode http
    timeout connect 5000ms
    timeout client 50000ms
    timeout server 50000ms

frontend http-in
    bind *:7474
    default_backend neo4j-master

backend neo4j-master
    option httpchk GET /db/manage/server/ha/master
    server s1 192.168.10.134:7474 maxconn 32 check
    server s2 192.168.10.135:7474 maxconn 32 check
    server s3 192.168.10.136:7474 maxconn 32 check
    server s4 192.168.10.133:7474 maxconn 32 check

frontend http-in-slaves
    bind *:7475
    default_backend neo4j-slaves

backend neo4j-slaves
    option httpchk GET /db/manage/server/ha/slave
    server s1 192.168.10.134:7474 maxconn 32 check
    server s2 192.168.10.135:7474 maxconn 32 check
    server s3 192.168.10.136:7474 maxconn 32 check
    server s4 192.168.10.133:7474 maxconn 32 check
```

## 6.2 启动服务

```bash
/usr/local/sbin/haproxy -f /usr/local/sbin/haproxy.cfg &
```

> 应用程序在写数据时连接192.168.10.141:7474服务器，再读数据时连接192.168.10.141:7475

**最新的neo4j选举模式(1.9 版本)使用使用paxos 取代了zookeeper**，采用自动选举的机制，在一个master宕机了，会自动选举下一个master作为替代。且这种模式无需进行文件配置，比起zookeeper要简便，也可以指定mater就是初始启动的时候id最小的就是master。