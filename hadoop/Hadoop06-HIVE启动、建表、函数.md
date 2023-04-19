[TOC]
# 1. HIVE基本定义
HIVE可以让你把你的数据文件 “映射”成一个表，然后还可以让你输入SQL指令，它就能将你的SQL指令解析后生成mapreduce程序进行逻辑运算；

HIVE： 就是一个利用HDFS存储数据，利用mapreduce运算数据的数据仓库工具
## 1.1 hive基本思想
Hive是基于Hadoop的一个数据仓库工具(离线)，可以<font style="background-color: yellow;">将结构化的数据文件映射为一张数据库表，</font>并提供类SQL查询功能。
[![QiZNwt.png](https://www.z4a.net/images/2023/03/20/hive.png "HIVE的作用")](https://gd-hbimg.huaban.com/70f4748515e741ab5b4a36ec4418dfd8a22ce1e67e31-jUdEMj)

Hive构建在Hadoop之上:

1. HQL中对查询语句的解释、优化、生成查询计划是由Hive完成的
2. 所有的数据都是存储在Hadoop中
3. 查询计划被转化为MapReduce任务，在Hadoop中执行（有些查询没有MR任务，如：select * from table）
4. Hadoop和Hive都是用UTF-8编码的

**Hive是如何将SQL转化为MapReduce任务的**，整个编译过程分为六个阶段：

HiveSQL ->AST(抽象语法树) -> QB(查询块) ->OperatorTree（操作树）->优化后的操作树->mapreduce任务树->优化后的mapreduce任务树

1. Antlr定义SQL的语法规则，完成SQL词法，语法解析，将SQL转化为抽象语法树AST Tree
2. 遍历AST Tree，抽象出查询的基本组成单元QueryBlock
3. 遍历QueryBlock，翻译为执行操作树OperatorTree
4. 逻辑层优化器进行OperatorTree变换，合并不必要的ReduceSinkOperator，减少shuffle数据量
5. 遍历OperatorTree，翻译为MapReduce任务
6. 物理层优化器进行MapReduce任务的变换，生成最终的执行计划


## 1.2 为什么使用Hive

**(1) 直接使用hadoop所面临的问题**

- 人员学习成本太高
- 项目周期要求太短
- MapReduce实现复杂查询逻辑开发难度太大

**(2) 为什么要使用Hive**

- 操作接口采用类SQL语法，提供快速开发的能力。
- 避免了去写MapReduce，减少开发人员的学习成本。
- 功能扩展很方便。

## 1.3 Hive的特点
**(1) 可扩展**

Hive可以自由的扩展集群的规模，一般情况下不需要重启服务。

**(2) 延展性**

Hive支持用户自定义函数，用户可以根据自己的需求来实现自己的函数。

 

**(3) 容错**

良好的容错性，节点出现问题SQL仍可完成执行。



## 1.4 Hive和数据库异同
由于Hive采用了SQL的查询语言HQL，因此很容易将Hive理解为数据库。其实从结构上来看，Hive和数据库除了拥有类似的查询语言，再无类似之处。数据库可以用在Online的应用中，但是Hive是为数据仓库而设计的，清楚这一点，有助于从应用角度理解Hive的特性。

Hive和数据库的比较如下表：

|Hive|RDBMS|
|---|---|
|查询语言|HQL|
|数据存储|HDFS|
|数据格式|用户定义|
|数据更新|不支持|
|索引|无|
|执行|MapReduce|
|执行延迟|高|
|处理数据规模|大|
|可扩展性|高|


1. 查询语言。由于 SQL 被广泛的应用在数据仓库中，因此专门针对Hive的特性设计了类SQL的查询语言HQL。熟悉SQL开发的开发者可以很方便的使用Hive进行开发。

2. 数据存储位置。Hive是建立在Hadoop之上的，所有Hive的数据都是存储在HDFS中的。而数据库则可以将数据保存在块设备或者本地文件系统中。

3. 数据格式。Hive中没有定义专门的数据格式，数据格式可以由用户指定，用户定义数据格式需要指定三个属性：列分隔符（通常为空格、”\t”、”\x001″）、行分隔符（”\n”）以及读取文件数据的方法（Hive中默认有三个文件格式TextFile，SequenceFile以及RCFile）。由于在加载数据的过程中，不需要从用户数据格式到Hive定义的数据格式的转换，因此，Hive在加载的过程中不会对数据本身进行任何修改，而只是将数据内容复制或者移动到相应的HDFS目录中。而在数据库中，不同的数据库有不同的存储引擎，定义了自己的数据格式。所有数据都会按照一定的组织存储，因此，数据库加载数据的过程会比较耗时。

4. 数据更新。由于Hive是针对数据仓库应用设计的，而数据仓库的内容是读多写少的。因此，Hive中不支持对数据的改写和添加，所有的数据都是在加载的时候中确定好的。而数据库中的数据通常是需要经常进行修改的，因此可以使用INSERT INTO … VALUES添加数据，使用UPDATE … SET修改数据。

5. 索引。之前已经说过，Hive在加载数据的过程中不会对数据进行任何处理，甚至不会对数据进行扫描，因此也没有对数据中的某些Key建立索引。Hive要访问数据中满足条件的特定值时，需要暴力扫描整个数据，因此访问延迟较高。由于MapReduce的引入， Hive可以并行访问数据，因此即使没有索引，对于大数据量的访问，Hive仍然可以体现出优势。数据库中，通常会针对一个或者几个列建立索引，因此对于少量的特定条件的数据的访问，数据库可以有很高的效率，较低的延迟。由于数据的访问延迟较高，决定了Hive不适合在线数据查询。

6. 执行。Hive中大多数查询的执行是通过Hadoop提供的MapReduce来实现的（类似select * from tbl的查询不需要MapReduce）。而数据库通常有自己的执行引擎。

7. 执行延迟。之前提到，Hive在查询数据的时候，由于没有索引，需要扫描整个表，因此延迟较高。另外一个导致Hive执行延迟高的因素是MapReduce框架。由于MapReduce本身具有较高的延迟，因此在利用MapReduce执行Hive查询时，也会有较高的延迟。相对的，数据库的执行延迟较低。当然，这个低是有条件的，即数据规模较小，当数据规模大到超过数据库的处理能力的时候，Hive的并行计算显然能体现出优势。

8. 可扩展性。由于Hive是建立在Hadoop之上的，因此Hive的可扩展性是和Hadoop的可扩展性是一致的（世界上最大的Hadoop集群在Yahoo!，2009年的规模在4000台节点左右）。而数据库由于ACID语义的严格限制，扩展行非常有限。目前最先进的并行数据库Oracle在理论上的扩展能力也只有100台左右。

9. 数据规模。由于Hive建立在集群上并可以利用MapReduce进行并行计算，因此可以支持很大规模的数据；对应的，数据库可以支持的数据规模较小。

# 2. 使用
## 2.1 用bin/hive 启动一个交互式查询软件来使用

使用间接连接（像JDBC连接就是）或使用远程连接的话，则需要在对hive进行指令操作之前启动metastore或者hiveserver2才行！否则会报错
```bash
FAILED: HiveException java.lang.RuntimeException: Unable to instantiate org.apache.hadoop.hive.ql.metadata.SessionHiveMetaStoreClient

```

在hive服务端开启hive metastore服务


```bash
nohup hive --service metastore -p 9083 1>/dev/null 2>&1 &
```

如果你在hive-site.xml里指定了hive.metastore.uris的port

```xml
  <property>
          <name>hive.metastore.uris</name>
          <value>thrift://hadoop03:9083</value>
   </property>
```
就可以不指定端口启动了


```bash
nohup hive --service metastore 1>/dev/null 2>&1 &
```


[![hive查询](https://ftp.bmp.ovh/imgs/2019/11/e7377f975424b49d.png "hive查询")](https://ftp.bmp.ovh/imgs/2019/11/e7377f975424b49d.png "hive查询")

## 2.2 用bin/hiveserver2 启动一个hive的服务端软件来接收查询请求

补充：<font style="background-color: yellow;">后台运行程序</font>


```shell
1、在linux中，如果需要把一个程序真正运行在后台，可以这样启动
nohup sh a.sh 1>/dev/null 2>&1 &
// nohup：变为系统级进程，不会随着root用户退出而失效
// /dev/null linux中的黑洞目录，任何程序写入的信息都会消失

2、查询后台进程
jobs

3、放到前台
fg [任务号]

4、其他后台运行方式
(1)sh a.sh & //会干扰前台
(2)sh s.sh 1>/root/sh.log
           2>/root/sh.err &
```

### 2.2.1 启动hiveserver服务软件


```shell
bin/hiveserver2   ## 在前台运行
nohup bin/hiveserver2 1>/dev/null 2>&1 &  ## 在后台运行
```


别管怎么运行，可以用netstat -nltp | grep 10000 来检查是否已经启动完成

**为什么需要配置metastore服务？既然直接连接可以直接连上操作，那为什么还需要间接连接呢？hiveserver2是什么**？

参考：[Hive服务启动之metastore配置和hiveserver2](https://blog.csdn.net/qq_48784015/article/details/109016876)

> 先上结论：远程连接必须配置metastore服务，配置metastore服务或启动hiveserver2后才允许多台客户端访问。hiveserver2直白来说就是用于hive客户端连接hive服务端（远程连接）的，hiveserver2启动时会自动启动metastore服务。
> 
> 原因如下：众所周知，hive既是客户端又是服务端而且Hive是基于Hadoop分布式文件系统，数据存储于Hadoop分布式文件系统中。元数据主要包含Hive建表信息。元数据存储在关系型数据库如Derby、MySQL中，通常这些数据库不会暴露太多外部接口，所以远程连接或多台客户端连接都需要一个中间者！我们所配置的metastore参数就是起到这个作用。
>
> 而且非服务端的hive 配置中只需要配置metastore参数就能连上。而且这些客户端不需要知道MySQL数据库的用户名和密码，只需要连接metastore 服务即可

### 2.2.2 启动beeline客户端软件连接hiveserver2


```shell
bin/beeline -u jdbc:hive2://linux01:10000 -n root
```
[![beeline启动hive](https://gd-hbimg.huaban.com/009bb36e334d2834008f1708f3a82f121839aa067375-CCElhM "beeline启动hive")](https://ftp.bmp.ovh/imgs/2019/11/7f48d6719d935276.png "beeline启动hive")

**通过zookeeper连接ambari上的hive**

```bash
beeline
// 启动beeline，账号和密码默认为空
!connect jdbc:hive2://flpt02:2181,flpt03:2181,flpt01:2181/;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=hiveserver2
```

[![DxbF2.png](https://gd-hbimg.huaban.com/fc7aaa39fb50c93750382aa7bac7a1de7a3cc3bca693-Sg0tgE)](https://z3.ax1x.com/2021/05/13/gBLpeP.png)

##  2.3 用一次性命令来运行hive的sql

```shell
bin/hive -e "sql语句……""
```
有了这种方式，就可以将大量的sql语句写在一个脚本中，<font style="color: red;">自动批量执行</font>

```shell
#!/bin/bash
bin/hive -e "create table t_u_2 as select uid,uname from t_user"
bin/hive -e "create table t_u_distinct as select distinct uid from t_u_2"
bin/hive -e "select count(1) from t_u_distinct"
```

## 2.4 hive查看版本号
方法1：`hive --version`

```Java
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=512M; support was removed in 8.0
SLF4J: Class path contains multiple SLF4J bindings.

……

Hive 1.1.0-cdh5.13.1
Subversion file:///data/jenkins/workspace/generic-package-centos64-7-0/topdir/BUILD/hive-1.1.0-cdh5.13.1 -r Unknown
Compiled by jenkins on Thu Nov 9 08:36:37 PST 2017
From source with checksum a07ce805880f1ad3d4f92b623df89745
```

方法2：通过jar包查看

在文件系统中，定位运行hive的jar文件，通常在/usr/lib/hive/lib或类似的路径下。jar文件的名称类似于hive-[version]-[distribution].jar

1. `whereis hive` 获取 hive位置
2. 查看hive的jar包版本

# 3.  建库、建表
## 3.1 HIVE的建库

```sql
CREATE DATABASE db_name;
```

建库的实质：

HIVE 会记住关于库定义的信息(库名叫什么)

HIVE会在HDFS上创建一个库目录：

/user/hive/warehouse/db_name

## 3.2 HIVE的建表
参考：Hive数据加载(内部表，外部表，分区表)

https://blog.csdn.net/scgaliguodong123_/article/details/46906427
### 3.2.1. 内部表建表语句

使用建立的db_name库

```sql
use db_name;
```
建表

```sql
CREATE TABLE t_name(filed1 type,field2 type,field3 type)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;
```

建表的实质：

1. HIVE 会记住关于表定义的信息（表名叫什么、有哪些字段、数据文件的分隔符？）
2. HIVE会在HDFS上创建一个表数据文件存储目录：
/user/hive/warehouse/db_name/t_name

### 3.2.2. 外部表建表语句

```sql
CREATE EXTERNAL TABLE t_name(filed1 type,field2 type,field3 type)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/aa/bb/' ;
```

注意：

当drop一个**内部表**时，hive会清除这个表的元数据，并**删除**这个表的**数据目录**；

当drop一个**外部表**时，hive会清除这个表的元数据，但**不会删**它的**数据目录**；

通常，外部表用于映射最开始的数据文件（一般是由别的系统所生成的）

> **Hive内部表和外部表的区别：**
> 
> 1) 内部表由hive自身管理，外部表数据由HDFS管理；
> 2) 内部表数据存储的位置是`hive.metastore.warehouse.dir`（默认：`/user/hive/warehouse`），外部表数据的存储位置由自己制定（如果没有LOCATION，Hive将在HDFS 上的`/user/hive/warehouse` 文件夹下以外部表的表名创建一个文件夹，并将数据这个表的数据存放在这里）；
> 3) 删除内部表会直接删除元数据（metadata）及存储数据；删除外部表仅仅会删除元数据，HDFS 上的文件并不会被删除；对内部表的修改会直接同步给元数据，而对外部表的表结构和分区进行修改，在需要修复（mask repair table tablename）


### 2.2.3 分区表建表语句

**分区表：会在表数据存储目录中，允许有子目录（分区）**

```sql
CREATE TABLE t_access(ip string,url string)
PARTITIONED BY (day string)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',';
```

1. **建表时**，只是指定这个表可以按day变量的具体值建子目录；所以建表时，**不会生成子目录**；
2. **导入数据**到该表时，就需要指定一个具体的**day**变量的值，hive就会用这个值**建一个子目录**，并将**数据文件放入该子目录**；


导入数据语句：

```sql
LOAD DATA LOCAL INPATH '/root/access.1' INTO TABLE t_access PARTITION(day=’2017-11-25’);
LOAD DATA LOCAL INPATH '/root/access.2' INTO TABLE t_access PARTITION(day=’2017-11-26’);
```
格式：LOAD DATA 【LOCAL】 INPATH '....' 【OVERWRITE】 INTO TABLE t1 【PARTITION (...)】



以上内容中，方括号中的表示可选部分。分别解释以下：

- LOCAL 表示从本地加载数据，即Linux文件系统，这时数据会复制到HDFS的Hive仓库中；如果不使用LOCAL，指的是数据从HDFS的某个路径移动到Hive仓库中。
- OVERWRITE指的是否覆盖原始数据。如果不使用OVERWRITE，但是已经导入过这批数据，那么新的导入依然能够成功,即产生两份，而不是覆盖掉原来的那一份
- PARTITION指的是导入到指定分区表中。
- **使用load data操作的时候，不管是外部表还是内部表，<font style="color: red;">如果源数据存在于HDFS层，都是数据的移动。</font>**
即源数据从HDFS存储路径移动到HIVE数据仓库默认路径。
- alter table是重新指定的HDFS文件location路径，与默认的HIVE路径不一样

样例：ALTER table BraS ADD PARTITION (province="XXX",partitionDate="20190101") LOCATION "hdfs://目录XXXX/省份/20190101";

参考：HIVE中处理外部表和内部表时alter table和load data区别
https://blog.csdn.net/henrrywan/article/details/90612741

[![hive存储结构](https://ftp.bmp.ovh/imgs/2019/11/2b53717dab616543.png "hive存储结构")](https://ftp.bmp.ovh/imgs/2019/11/2b53717dab616543.png "hive存储结构")

**<font style="color: red;">分区表的查询特性</font>**：

**1、可以进行全表查询**

select count(1) from t_access;

 

**2、可以针对具体的分区查询**

select count(1) from t_access where day=’2017-11-25’;  

// 把分区变量day当成了一个字段


# 4. hive语法
## <font style="color: red;">4.1 分析函数-分组TOPN</font>
**(1) 需求:**

有如下数据：
> 1,18,a,male
> 
> 2,19,b,male
> 
> 3,22,c,female
> 
> 4,16,d,female
> 
> 5,30,e,male
> 
> 6,26,f,female

需要查询出每种性别中年龄最大的2条数据

 

**(2) 实现：**

### 4.1.1 row_number() over() <font style="color: red;">★</font>

使用row_number函数，对表中的数据按照性别分组，按照年龄倒序排序并进行标记

hql代码：

```sql
select id,age,name,sex,
row_number() over(partition by sex order by age desc) as rank
from t_rownumber
```

产生结果：

[![QiZDSg.png](https://s2.ax1x.com/2019/11/28/QiZDSg.png "分组TopN")](https://s2.ax1x.com/2019/11/28/QiZDSg.png)

然后，利用上面的结果，查询出rank<=2的即为最终需求


```sql
select id,age,name,sex
from
(select id,age,name,sex,
row_number() over(partition by sex order by age desc) as rank
from t_rownumber) tmp
where rank<=2;
```

### 4.1.2 Rank() over()


```sql
SELECT
  SalesOrderID,
  CustomerID,
  RANK() OVER (ORDER BY CustomerID) AS Rank
 FROM Sales.SalesOrderHeader
```

结果集：

[![QiZyOs.png](https://s2.ax1x.com/2019/11/28/QiZyOs.png "Rank() over")](https://s2.ax1x.com/2019/11/28/QiZyOs.png)

**区别：**

- rank over 可并列排名，不能跟partition by
- row_number() over() 不会出现并列排名，可以跟partition by

**distribute by 和 partition by 的区别**
> ```bash
> row_number() over( partition by 分组的字段order by 排序的字段) as rank(rank
> 可随意定义表示排序的标识)
> 
> row_number() over( distribute by 分组的字段sort by 排序的字段) as rank(rank
> 可随意定义表示排序的标识)
> ```
> 
> 注意：<br>
> `partition by` 只能和`order by` 组合使用<br>
> `distribute by` 只能和`sort by` 使用

## 4.2 表生成函数
### 4.2.1 行转列函数：explode()

假如有以下数据：
> 1,zhangsan,化学:物理:数学:语文
> 
> 2,lisi,化学:数学:生物:生理:卫生
> 
> 3,wangwu,化学:语文:英语:体育:生物

映射成一张表：

```sql
create table t_stu_subject(id int,name string,subjects array<string>)
row format delimited fields terminated by ','
collection items terminated by ':';
```

使用explode()对数组字段“炸裂”

[![QiZUTP.png](https://s2.ax1x.com/2019/11/28/QiZUTP.png "explode “炸裂函数")](https://s2.ax1x.com/2019/11/28/QiZUTP.png)

然后，我们利用这个explode的结果，来求去重的课程：

```sql
select distinct tmp.sub
from
(select explode(subjects) as sub from t_stu_subject) tmp;
```

### 4.2.2 表生成函数lateral view

```sql
select id,name,tmp.sub
from t_stu_subject lateral view explode(subjects) tmp as sub;
```

view explode() <font style="color: red;">~~**as**~~</font> tmp之间不能加 as

[![QiZwY8.png](https://s2.ax1x.com/2019/11/28/QiZwY8.png "lateral view")](https://s2.ax1x.com/2019/11/28/QiZwY8.png)

理解：lateral view 相当于**两个表在join**

左表：是原表

右表：是explode(某个集合字段)之后产生的表

而且：这个join**只在同一行的数据间**进行


---

那样，可以方便做更多的查询：

比如，查询选修了生物课的同学

```sql
select a.id,a.name,a.sub from 
(select id,name,tmp.sub as sub from t_stu_subject lateral view explode(subjects) tmp as sub) a
where sub='生物';
```

## 4.3 排序
- `order by`是全局排序，`sort by`是分区内排序(每个reduce内)
- `partition by`只能和`order by`组合使用<br>
 `distribute by`只能和`sort by`使用

**distribute by 和partition by 的区别**
> ```bash
> row_number() over( partition by 分组的字段order by 排序的字段) as rank(rank
> 可随意定义表示排序的标识)
> 
> row_number() over( distribute by 分组的字段sort by 排序的字段) as rank(rank
> 可随意定义表示排序的标识)
> ```
> 
> 注意：<br>
> `partition by` 只能和`order by` 组合使用<br>
> `distribute by` 只能和`sort by` 使用


## 4.4 union和union all的区别

- union：对两个结果集进行并集操作，不包括重复行，同时进行默认规则的排序；
- union all：对两个结果集进行并集操作，包括重复行，不进行排序；
# 5. Hive的Shell操作
## 5.1 Hive支持的一些命令
- Command Description
- quit Use quit or exit to leave the interactive shell.
- set key=value Use this to set value of particular configuration variable. One thing to note here is that if you misspell the variable name, cli will not show an error.
- set This will print a list of configuration variables that are overridden by user or hive.
- set -v This will print all hadoop and hive configuration variables.
- add FILE [file] [file]* Adds a file to the list of resources
- add jar jarname
- list FILE list all the files added to the distributed cache
- list FILE [file]* Check if given resources are already added to distributed cache
- ! [cmd] Executes a shell command from the hive shell
- dfs [dfs cmd] Executes a dfs command from the hive shell
- [query] Executes a hive query and prints results to standard out
- source FILE Used to execute a script file inside the CLI.

## 5.2 语法结构

```hive
hive [-hiveconf x=y]* [<-i filename>]* [<-f filename>|<-e query-string>] [-S]
```

说明：
1. -i 从文件初始化 HQL
2. -e 从命令行执行指定的 HQL
3. -f 执行 HQL 脚本
4. -v 输出执行的 HQL 语句到控制台
5. -p connect to Hive Server on port number
6. -hiveconf x=y（Use this to set hive/hadoop configuration variables）
7. -S：表示以不打印日志的形式执行命名操作


## 5.4 Hive指定队列

```bash
# 老版本
set mapred.job.queue.name=queueName;
set mapred.queue.names=queueName;

# 新版本
set mapreduce.job.queuename=queueName;
```

## 5.5 sql 语句的执行顺序

sql语句
```sql
SELECT 查询列表.
FROM 表 1                                      
【连接类型】 JOIN 表2                           
ON 连接条件
WHERE 筛选条件
GROUP BY 分组列表
HAVING 分组后的筛选条件
ORDER BY 排序的字段
LIMIT 起始的条目索引，条目数;
```

```sql
SQL语句书写顺序
select -> from -> join -> on ->  where -> group by -> having -> order by -> limit

SQL语句执行顺序
from -> join -> on -> where -> group by -> having -> select -> order by -> limit
```

### 5.5.1 SQL中where与having的区别

**where:**<br>
- where是一个约束声明,使用where来约束来自数据库的数据;
- where是在结果返回之前起作用的;
- where中不能使用聚合函数。

**having:**<br>
- having是一个过滤声明;
- 在查询返回结果集以后，对查询结果进行的过滤操作;
- 在having中可以使用聚合函数

## 5.6 示例
**（1）运行一个查询**

```bash
[hadoop@hadoop3 ~]$ hive -e "select * from cookie.cookie1;"
```

[![QiZdFf.png](https://s2.ax1x.com/2019/11/28/QiZdFf.png "hive 查询")](https://s2.ax1x.com/2019/11/28/QiZdFf.png)

**（2）运行一个文件**

编写hive.sql文件<br>
[![QiZGyd.png](https://s2.ax1x.com/2019/11/28/QiZGyd.png "编写hive.sql文件")](https://s2.ax1x.com/2019/11/28/QiZGyd.png)

运行编写的文件<br>
[![QiZJOA.png](https://s2.ax1x.com/2019/11/28/QiZJOA.png "hive运行编写的文件")](https://s2.ax1x.com/2019/11/28/QiZJOA.png)

**（3）运行参数文件**

从配置文件启动 hive，并加载配置文件当中的配置参数<br>
[![QiZrlQ.png](https://s2.ax1x.com/2019/11/28/QiZrlQ.png "hive运行参数文件")](https://s2.ax1x.com/2019/11/28/QiZrlQ.png)


# 6. UDF

只需要继承`org.apache.hadoop.hive.ql.exec.UDF`，并定义
`public Object evaluate(Object args) {}`方法即可。

如下例子是一个传入string参数，调另一个接口，返回新的string的udf：

```java
public class MyUDF extends UDF{
            public String evaluate(String phone) {
                String res="phoneNumber = " + phone;
                return res;
            }
    }
```

将上面代码打包，hive_udf.jar

## 6.1 发布

**一、临时函数**<br>
只对当前会话窗口有效

临时函数，jar可以放在Linux本地
```
1) add jar /home/hadoop/data/hive/hive_udf.jar;  ---添加jar
2) create temporary function MyUDF as 'com.ctsi.udf.MyUDF';----创建函数
3) show functions  查看可以看到MyUDF
```

==注意：如果jar包是上传到$HIVE_HOME/lib/目录以下，就不需要执行add命令了==

**二、永久函数**<br>
创建的永久函数可以在任何一个窗口使用，重新启动函数也照样可以使用

永久函数，jar需放在hdfs
```
1) hdfs dfs -put -f hive-udf.jar /user/udfs/hive-udf.jar   ---jar放到hdfs
2) create function xxx as 'xxx' using jar 'hdfs://user/udfs/hive-udf.jar';----创建函数
3) show functions  查看
```

检查mysql中的元数据，测试函数的信息已经注册到了元数据中

```sql
select * from funcs;
```


## 6.2 测试

```
hive>select MyUDF('12315');

OK

phoneNumber = 12315
```

## 6.3 删除


```
drop temporary function if exists MyUDF;
```

