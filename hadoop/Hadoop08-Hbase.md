[TOC]
# 1. 安装hbase
## 1.1 环境配置
<font style="background-color:#D3FF93;color: red;">**注意：hbase集群的服务器时间同步要求非常严格，机器之间的时间差不能超过30s**</font>

设置linux系统时间的命令：

```bash
date -s '2019-11-08 16:00:30'

wclock -w
```


如果能访问网络，可用时间同步服务：

```bash
yum install ntpdate -y

ntpdate 0.asia.pool.ntp.org
```

## 1.2 修改配置文件
解压hbase安装包

**(1) 修改hbase-env.sh**


```bash
export JAVA_HOME=/root/apps/jdk1.7.0_67

export HBASE_MANAGES_ZK=false
```

**(2) 修改hbase-site.xml**

```bash
<configuration>

<!-- 指定hbase在HDFS上存储的路径 -->

  <property>

    <name>hbase.rootdir</name>

    <value>hdfs://linux01:9000/hbase</value>

  </property>

<!-- 指定hbase是分布式的 -->

  <property>

    <name>hbase.cluster.distributed</name>

    <value>true</value>

  </property>

<!-- 指定zk的地址，多个用“,”分割 -->

  <property>

    <name>hbase.zookeeper.quorum</name>

    <value>linux01:2181,linux02:2181,linux03:2181</value>

  </property>

</configuration>
```

**(3) 修改regionservers**

```bash
linux01
linux02
linux03
linux04
```

## 1.3 启动hbase集群

```bash
bin/start-hbase.sh
```

启动完后，还可以在集群中找任意一台机器启动一个**备用的master**

```bash
bin/hbase-daemon.sh start master
```

<font color="red">**新启的这个master会处于backup状态**</font>


```bash
单独启动一个HMaster进程：

bin/hbase-daemon.sh start master

单独停止一个HMaster进程：

bin/hbase-daemon.sh stop master

单独启动一个HRegionServer进程：

bin/hbase-daemon.sh start regionserver

单独停止一个HRegionServer进程：

bin/hbase-daemon.sh stop regionserver
```



## 1.4 启动hbase客户端

```bash
bin/hbase shell

Hbase> list     // 查看表

Hbase> status   // 查看集群状态

Hbase> version  // 查看集群版本
```

# 2. 命令行操作示范
## 2.1 增
**1、建表**

```bash
create 't_user','base_info','extra_info'
```

**2、插入数据**

```bash
put 't_user','rk001','base_info:name','zhangsan'

put 't_user','rk001','extra_info:married','false'
```

**3、增加某个列族**

<font color="red">先禁用，再修改表结构</font>

```bash
disable 't_user'
desc 't_user' #查看表结构

alter 't_user', {NAME => 'other_info', VERSIONS => 1}

desc 't_user'
enable 't_user' #启动表
```

## 2.2 查
**1、查看所有的表**

```bash
list
```

**2、查看表结构**

`describe(desc) <table>`

```bash
desc 't_user'
```


**3、全表扫描**

```bash
scan 't_user'
```

**4、范围扫描(包头不包尾))**

结果包括STARTROW 本身，不包括ENDROW本身，使用JAVA API 也是一样逻辑。 

```bash
scan  't_user',{STARTROW =>'rk001',ENDROW =>'rk010'}
```

row key在rk001* 到rk090* 之间的所有数据

> rk001
>
> rk0010
>
> rk00101
>
> rk0020……
>
> rk003……
>
> rk009……

**5、单行获取**

```bash
get 't_user','rk001'
```

## 2.3 改

1、**修改数据**

<font color="red">就是put插入后覆盖</font>

## 2.4 删

**1、删除一个kv**

语法：`delete <table>,<rowkey>,<family:column>,<timestamp>`,必须指定列名

```bash
delete 't_user','rk001','base_info:name'
```

**2、删掉整行**

```bash
deleteall 't_user','rk001'
```

**3、清空整个表**

```bash
truncate 't_user'
```

**4、删除整个表**

<font color="red">先禁用，再drop</font>

```bash
disable 't_user'

drop 't_user'
```

**5、删除某个列族**

<font color="red">先禁用，再修改表结构</font>

```bash
disable 't_user'
desc 't_user' #查看表结构

alter 't_user', {NAME => 'extra_info', METHOD => 'delete'}

desc 't_user'
enable 't_user' #启动表
```