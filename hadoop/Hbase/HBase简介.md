[TOC]

参考：https://www.cnblogs.com/tgzhu/p/5857035.html

# 1 HBase简介
## 1.1 什么是HBase
HBASE是一个高可靠性、高性能、面向列、可伸缩的分布式存储系统，利用HBASE技术可在廉价PC Server上搭建起大规模结构化存储集群。

HBASE的目标是存储并处理大型的数据，更具体来说是仅需使用普通的硬件配置，就能够处理由成千上万的行和列所组成的大型数据。

HBASE是Google Bigtable的开源实现，但是也有很多不同之处。比如：Google Bigtable使用GFS作为其文件存储系统，HBASE利用Hadoop HDFS作为其文件存储系统；Google运行MAPREDUCE来处理Bigtable中的海量数据，HBASE同样利用Hadoop MapReduce来处理HBASE中的海量数据；Google Bigtable利用Chubby作为协同服务，HBASE利用Zookeeper作为协同服务。

## 1.2 与传统数据库的对比
1、传统数据库遇到的问题：

1) 数据量很大的时候无法存储；
2) 没有很好的备份机制；
3) 数据达到一定数量开始缓慢，很大的话基本无法支撑；

2、HBASE优势：

1) 线性扩展，随着数据量增多可以通过节点扩展进行支撑；
2) 数据存储在hdfs上，备份机制健全；
3) 通过zookeeper协调查找数据，访问速度快。

## 1.3 HBase集群中的角色
1. 一个或者多个主节点，Hmaster；
2. 多个从节点，HregionServer；
3. HBase依赖项，zookeeper；

# 2. HBase数据模型
[![lzcvTA.png](https://s2.ax1x.com/2020/01/17/lzcvTA.png)](https://s2.ax1x.com/2020/01/17/lzcvTA)
## 2.1 HBase的存储机制
HBase是一个面向列的数据库，在表中它由行排序。表模式定义只能列族，也就是键值对。一个表有多个列族以及每一个列族可以有任意数量的列。后续列的值连续存储在磁盘上。表中的每个单元格值都具有时间戳。总之，在一个HBase：

- 表是行的集合。
- 行是列族的集合。
- 列族是列的集合。
- 列是键值对的集合。

这里的列式存储或者说面向列，其实说的是列族存储，HBase是根据列族来存储数据的。列族下面可以有非常多的列，列族在创建表的时候就必须指定。

<font style="font-size:18px;color: red;">**HBase 和 RDBMS的比较**</font>

[![lzczFI.png](https://s2.ax1x.com/2020/01/17/lzczFI.png)](https://s2.ax1x.com/2020/01/17/lzczFI.png)

**RDBMS的表**：</br>
[![lzgpfP.png](https://s2.ax1x.com/2020/01/17/lzgpfP.png)](https://s2.ax1x.com/2020/01/17/lzgpfP.png)

**Hbase的表**：</br>
[![lzcjwd.png](https://s2.ax1x.com/2020/01/17/lzcjwd.png)](https://s2.ax1x.com/2020/01/17/lzcjwd.png)

## 2.2 Row Key 行键
与nosql数据库一样，row key是用来表示唯一一行记录的主键，HBase的数据时按照RowKey的**字典顺序**进行全局排序的，所有的查询都只能依赖于这一个排序维度。访问HBASE table中的行，只有三种方式：

1. 通过单个row key访问；

2. 通过row key的range（正则）

3. 全表扫描

Row  key 行键（Row key）可以是任意字符串(最大长度是64KB，实际应用中长度一般为10-1000bytes)，在HBASE内部，row  key保存为字节数组。存储时，数据按照Row  key的字典序(byte  order)排序存储。设计key时，要充分排序存储这个特性，将经常一起读取的行存储放到一起。(位置相关性)

## 2.3 Columns Family 列族
列簇：HBASE表中的每个列，都归属于某个列族。列族是表的schema的一部分(而列不是)，必须在使用表之前定义。列名都以列族作为前缀。例如courses：history，courses：math 都属于courses这个列族。

## 2.4 Cell
由{row key，columnFamily，version} 唯一确定的单元。cell中的数据是没有类型的，全部是字节码形式存储。

关键字：无类型、字节码

## 2.5 Time Stamp 时间戳
HBASE中通过rowkey和columns确定的为一个存储单元称为cell。每个cell都保存着同一份数据的多个版本。版本通过时间戳来索引。时间戳的类型是64位整型。时间戳可以由HBASE(在数据写入时自动)赋值，此时时间戳是精确到毫秒的当前系统时间。时间戳也可以由客户显示赋值。如果应用程序要避免数据版本冲突，就必须自己生成具有唯一性的时间戳。每个cell中，不同版本的数据按照时间倒序排序，即最新的数据排在最前面。

为了避免数据存在过多版本造成的管理(包括存储和索引)负担，HBASE提供了两种数据版本回收方式。一是保存数据的最后n个版本，二是保存最近一段时间内的版本(比如最近7天)。用户可以针对每个列族进行设置。

# 3. HBase原理

## 3.1 HBase系统架构体系图

[![lz2OIA.png](https://s2.ax1x.com/2020/01/17/lz2OIA.png)](https://s2.ax1x.com/2020/01/17/lz2OIA.png)

组成部件说明：

**Client**：

使用HBase RPC机制与HMaster和HRegionServer进行通信
Client与HMaster进行管理类操作
Client与HRegionServer进行数据读写类操作

**Zookeeper**：

Zookeeper Quorum存储-ROOT-表地址、HMaster地址
HRegionServer把自己以Ephedral方式注册到Zookeeper中，HMaster随时感知各个HRegionServer的健康状况
Zookeeper避免HMaster单点问题

**HMaster**：

HMaster没有单点问题，HBase可以启动多个HMaster，通过Zookeeper的Master Election机制保证总有一个Master在运行
主要负责Table和Region的管理工作：
1. 管理用户对表的增删改查操作
2. 管理HRegionServer的负载均衡，调整Region分布
3. Region Split后，负责新Region的分布
4. 在HRegionServer停机后，负责失效HRegionServer上Region迁移

**HRegionServer**：

HBase中最核心的模块，主要负责响应用户I/O请求，向HDFS文件系统中读写
[![lz2jPI.png](https://s2.ax1x.com/2020/01/17/lz2jPI.png)](https://s2.ax1x.com/2020/01/17/lz2jPI.png)

[![lz2Lad.png](https://s2.ax1x.com/2020/01/17/lz2Lad.png)](https://s2.ax1x.com/2020/01/17/lz2Lad.png)

HRegionServer管理一系列HRegion对象；<br/>
每个HRegion对应Table中一个Region，HRegion由多个HStore组成；<br/>
每个HStore对应Table中一个Column Family的存储；<br/>
Column Family就是一个集中的存储单元，故将具有相同IO特性的Column放在一个Column Family会更高效。

**HStore**：

HBase存储的核心。由MemStore和StoreFile组成。MemStore是Stored Memory Buffer。

**HLog**：

引入HLog原因：在分布式系统环境中，无法避免系统出错或者宕机，一旦HRegionServer意外退出，MemStore中的内存数据就会丢失，引入HLog就是防止这种情况。

工作机制：<br/>
每个HRegionServer中都会有一个HLog对象，HLog是一个实现Write Ahead Log的类，每次用户操作写入MemStore的同时，也会写一份数据到HLog文件，HLog文件定期会滚动出新，并删除旧的文件(已持久化到StoreFile中的数据)。当HRegionServer意外终止后，HMaster会通过Zookeeper感知，HMaster首先处理遗留的HLog文件，将不同region的log数据拆分，分别放到相应region目录下，然后再将失效的region重新分配，领取到这些region的HRegionServer在Load Region的过程中，会发现有历史HLog需要处理，因此会Replay HLog中的数据到MemStore中，然后flush到StoreFiles，完成数据恢复。

## 3.2 HBase的存储格式
HBase中的所有数据文件都存储在Hadoop HDFS文件系统上，格式主要有两种：

1. HFile，HBase中Key-Value数据的存储格式，HFile是Hadoop的二进制格式文件，实际上StoreFile就是对HFile做了轻量级包装，即StoreFile底层就是HFile。

2. HLog File，HBase中WAL(Write Ahead Log)的存储格式，物理上是Hadoop的Sequence File

**HFile**

[![lzfP4s.png](https://s2.ax1x.com/2020/01/17/lzfP4s.png)](https://s2.ax1x.com/2020/01/17/lzfP4s.png)

图片解释：

HFile文件不定长，长度固定的块只有两个：Trailer和FileInfo

Trailer中指针指向其他数据块的起始点

File Info中记录了文件的一些Meta信息，例如：AVG_KEY_LEN, AVG_VALUE_LEN, LAST_KEY, COMPARATOR, MAX_SEQ_ID_KEY等

Data Index和Meta Index块记录了每个Data块和Meta块的起始点

Data Block是HBase I/O的基本单元，为了提高效率，HRegionServer中有基于LRU的Block Cache机制

每个Data块的大小可以在创建一个Table的时候通过参数指定，大号的Block有利于顺序Scan，小号Block利于随机查询 

每个Data块除了开头的Magic以外就是一个个KeyValue对拼接而成, Magic内容就是一些随机数字，目的是防止数据损坏

HFile里面的每个KeyValue对就是一个简单的byte数组。这个byte数组里面包含了很多项，并且有固定的结构。

[![lzfFCn.png](https://s2.ax1x.com/2020/01/17/lzfFCn.png)](https://s2.ax1x.com/2020/01/17/lzfFCn.png)

KeyLength和ValueLength：两个固定的长度，分别代表Key和Value的长度 

Key部分：Row Length是固定长度的数值，表示RowKey的长度，Row 就是RowKey 

Column Family Length是固定长度的数值，表示Family的长度 

接着就是Column Family，再接着是Qualifier，然后是两个固定长度的数值，表示Time Stamp和Key Type（Put/Delete） 

Value部分没有这么复杂的结构，就是纯粹的二进制数据

**HLog File**

[![lzfAg0.png](https://s2.ax1x.com/2020/01/17/lzfAg0.png)](https://s2.ax1x.com/2020/01/17/lzfAg0.png)

HLog中日志单元WALEntry表示一次行级更新的最小追加单元，由两部分组成：**`HLogKey`** 和 **`WALEdit`**。

- **HLogKey** 中包含多个属性信息，包含table name、region name、sequenceid等；
- **WALEdit** 用来表示一个事务中的写入/更新集合，一次行级事务可以原子操作同一行中的多个列。为了保证region级别事务的写入原子性，一次写入操作中所有KeyValue会构成一条WALEdit记录

HLog文件就是一个普通的Hadoop Sequence File，Sequence File 的Key是HLogKey对象，HLogKey中记录了写入数据的归属信息，除了table和region名字外，同时还包括 sequence number和timestamp，timestamp是“写入时间”，sequence number的起始值为0，或者是最近一次存入文件系统中sequence number。 

HLog Sequece File的Value是HBase的KeyValue对象，即对应HFile中的KeyValue

## 3.3 sequenceid

sequenceid是region级别一次行级事务的自增序号。需要关注的地方有三个：

1) sequenceid是自增序号。很好理解，就是随着时间推移不断自增，不会减小。
2) sequenceid是一次行级事务的自增序号。行级事务是什么？简单点说，就是更新一行中的多个列族、多个列，行级事务能够保证这次更新的原子性、一致性、持久性以及设置的隔离性，HBase会为一次行级事务分配一个自增序号。
3) sequenceid是region级别的自增序号。每个region都维护属于自己的sequenceid，不同region的sequenceid相互独立。

## 3.4 写流程
[![lzfCNj.png](https://s2.ax1x.com/2020/01/17/lzfCNj.png)](https://s2.ax1x.com/2020/01/17/lzfCNj.png)

1) Client通过Zookeeper的调度，向RegionServer发出写数据请求，在Region中写数据；

2) 数据被写入Region的MemStore，知道MemStore达到预设阀值(即MemStore满)；

3) MemStore中的数据被Flush成一个StoreFile；

4) 随着StoreFile文件的不断增多，当其数量增长到一定阀值后，触发Compact合并操作，将多个StoreFile合并成一个StoreFile，同时进行版本合并和数据删除；

5) StoreFiles通过不断的Compact合并操作，逐步形成越来越大的StoreFile；

6) 单个StoreFile大小超过一定阀值后，触发Split操作，把当前Region Split成2个新的Region。父Region会下线，新Split出的2个子Region会被HMaster分配到相应的RegionServer上，使得原先1个Region的压力得以分流到2个Region上。

可以看出HBase只有增添数据，所有的更新和删除操作都是在后续的Compact历程中举行的，使得用户的写操作只要进入内存就可以立刻返回，实现了HBase I/O的高性能。

## 3.5 读流程
1) Client访问Zookeeper，查找-ROOT-表，获取.META.表信息；

2) 从.META.表查找，获取存放目标数据的Region信息，从而找到对应的RegionServer；

3) 通过RegionServer获取需要查找的数据；

4) RegionServer的内存分为MemStore和BlockCache两部分，MemStore主要用于写数据，BlockCache主要用于读数据。读请求先到MemStore中查数据，查不到就到BlockCache中查，再查不到就会到StoreFile上读，并把读的结果放入BlockCache。

寻址过程：client—>Zookeeper—>ROOT表—>.META. 表—>RegionServer—>Region—>client

> **备注：**
> 
> 0.96+ 版本 删除了root 表，改为查询zookeeper里面的文件

# 4. HBASE命令
## 4.1 登陆客户端
1、hbase提供了一个shell的终端给用户交互

<font style="color: red;">*hbase shell*</font>

[![lzhldg.png](https://s2.ax1x.com/2020/01/17/lzhldg.png)](https://s2.ax1x.com/2020/01/17/lzhldg.png)

2、如果退出执行**quit**命令

[![lzhusf.png](https://s2.ax1x.com/2020/01/17/lzhusf.png)](https://s2.ax1x.com/2020/01/17/lzhusf.png)

## 4.2 hbase shell命令

|名称|命令表达式|
|---|---|
|查看hbase状态|status|
|创建表|create '表名','列族名1','列族名2','列族名N'|
|查看所有表|list|
|描述表|describe '表名'|
|判断表存在|exists '表名'|
|判断是否禁用启用表|is_enabled '表名'/is_disabled '表名'|
|添加记录|put '表名','rowkey','列族：列'，'值'|
|查看记录rowkey下的所有数据|get '表名','rowkey'|
|查看所有记录|scan '表名'|
|查看表中的记录总数|count '表名'|
|获取某个列族|get  '表名','rowkey','列族：列'|
|获取某个列族的某个列|get '表名','rowkey','列族：列'|
|删除记录|delete '表名','行名','列族：列'|
|删除整行|deleteall '表名','rowkey'|
|删除一张表(先要屏蔽该表，才能对该表进行删除)|第一步 disable '表名'，第二步 drop '表名'|
|清空表|truncate '表名'|
|查看某个表某个列中所有数据|scan '表名',{COLUMNS=>'列族名：列名'}|
|更新记录|就是重写一遍，进行覆盖，**hbase没有修改，都是追加**|
具体实例：

**1、查看HBase运行状态  status**

[![lzhQeS.png](https://s2.ax1x.com/2020/01/17/lzhQeS.png)](https://s2.ax1x.com/2020/01/17/lzhQeS.png)

**2、创建表`create <table>,{NAME => <family>, VERSIONS => <VERSIONS>}`**

创建一个User表，并且有一个info列族<br/>
[![lzhKL8.png](https://s2.ax1x.com/2020/01/17/lzhKL8.png)](https://s2.ax1x.com/2020/01/17/lzhKL8.png)

**3、查看所有表  list**

[![lzhnQP.png](https://s2.ax1x.com/2020/01/17/lzhnQP.png)](https://s2.ax1x.com/2020/01/17/lzhnQP.png)

**4、描述表详情  describe 'User'**

[![lz4d1I.png](https://s2.ax1x.com/2020/01/17/lz4d1I.png)](https://s2.ax1x.com/2020/01/17/lz4d1I.png)

**5、判断表是否存在 exists  'User'**

[![lz4a9A.png](https://s2.ax1x.com/2020/01/17/lz4a9A.png)](https://s2.ax1x.com/2020/01/17/lz4a9A.png)

**6、启用或禁用表 is_disabled 'User' / is_enabled 'User'**

[![lz4Nhd.png](https://s2.ax1x.com/2020/01/17/lz4Nhd.png)](https://s2.ax1x.com/2020/01/17/lz4Nhd.png)

**7、添加记录，即插入数据，语法：`put <table>,<rowkey>,<family:column>,<value>`**

[![lz4YAe.png](https://s2.ax1x.com/2020/01/17/lz4YAe.png)](https://s2.ax1x.com/2020/01/17/lz4YAe.png)

**8、根据rowKey查询某个记录，语法：`get <table>,<rowkey>,[<family:column>, ...]`**

[![lz4ttH.png](https://s2.ax1x.com/2020/01/17/lz4ttH.png)](https://s2.ax1x.com/2020/01/17/lz4ttH.png)

**9、查询所有记录，语法：`scan <table>,{COLUMNS  =>  [family:column, ...], LIMIT => num}`**

扫描所有记录<br/>
[![lz40jP.png](https://s2.ax1x.com/2020/01/17/lz40jP.png)](https://s2.ax1x.com/2020/01/17/lz40jP.png)

扫描前2条<br/>
[![lz4Dnf.png](https://s2.ax1x.com/2020/01/17/lz4Dnf.png)](https://s2.ax1x.com/2020/01/17/lz4Dnf.png)

范围查询<br/>
[![lz4rB8.png](https://s2.ax1x.com/2020/01/17/lz4rB8.png)](https://s2.ax1x.com/2020/01/17/lz4rB8.png)

另外，还可以添加TIMERANGE和FILTER等高级功能，<font style="background-color: Yellow
;">STARTROW、ENDROW必须**大写**，否则报错，查询结果**不包含等于ENDROW的结果集**。</font>

**10、统计表记录数，语法：`count <table>, {INTERVAL => intervalNum，CACHE => cacheNum}`**

 INTERVAL设置多少行显示一次及对应的rowkey，默认1000；CACHE每次去取的缓存区大小，默认是10，调整该参数可提高查询速度。
 
[![lz52VO.png](https://s2.ax1x.com/2020/01/17/lz52VO.png)](https://s2.ax1x.com/2020/01/17/lz52VO.png)

**11、删除**

删除列<br/>
[![lz5RaD.png](https://s2.ax1x.com/2020/01/17/lz5RaD.png)](https://s2.ax1x.com/2020/01/17/lz5RaD.png)

删除整行<br/>
[![lz5WIe.png](https://s2.ax1x.com/2020/01/17/lz5WIe.png)](https://s2.ax1x.com/2020/01/17/lz5WIe.png)

清空表中所有数据<br/>
[![lz5hPH.png](https://s2.ax1x.com/2020/01/17/lz5hPH.png)](https://s2.ax1x.com/2020/01/17/lz5hPH.png)

**12、禁用或启用表**

禁用表<br/>
[![lz5OIg.png](https://s2.ax1x.com/2020/01/17/lz5OIg.png)](https://s2.ax1x.com/2020/01/17/lz5OIg.png)

启用表<br/>
[![lzINQI.png](https://s2.ax1x.com/2020/01/17/lzINQI.png)](https://s2.ax1x.com/2020/01/17/lzINQI.png)

**12、删除表**

删除前，必须先disable<br/>
[![disable hbase表.png](https://i0.wp.com/i.loli.net/2020/01/17/8ZHfvaJNlqwj5We.png)](https://files.catbox.moe/c1skz9.png)

# 5. hbase 面试题
## 5.1 compact 用途是什么，什么时候触发，分为哪两种，有什么区别？

在hbase 中每当有memstore 数据flush到磁盘之后，就形成一个storefile，当storeFile
的数量达到一定程度后，就需要将storefile 文件来进行compaction 操作。

**Compact 的作用：**
- ① 合并文件
- ② 清除过期，多余版本的数据，提高读写数据的效率

HBase 中实现了两种compaction 的方式：**minor and major**。 这两种compaction 方式的区别是：

1) Minor 操作只用来做部分文件的合并操作以及包括minVersion=0 并且设置ttl 的过期版本清理，不做任何删除数据、多版本数据的清理工作。
2) Major 操作是对Region 下的HStore 下的所有StoreFile 执行合并操作，最终的结果是整理合并出一个文件。

## 5.2 百亿数据存入HBase，如何保证数据的存储正确性和时效性？

**需求分析：**<br>
1) 百亿数据：证明数据量非常大；
2) 存入HBase：证明是跟HBase 的写入数据有关；
3) 保证数据的正确：要设计正确的数据结构保证正确性；
4) 在规定时间内完成：对存入速度是有要求的。

**解决思路：**<br>
1) 数据量百亿条，什么概念呢？假设一整天60x60x24 = 86400 秒都在写入数据，那么每秒的写入条数高达100 万条，HBase 当然是支持不了每秒百万条数据的，所以这百亿条数据可能不是通过实时地写入，而是批量地导入。<br>
批量导入推荐使用BulkLoad 方式（推荐阅读：Spark 之读写HBase），性能是普通写入方式几倍以上；
2) 存入HBase：普通写入是用JavaAPI put 来实现，批量导入推荐使用BulkLoad；
3) 保证数据的正确：这里需要考虑RowKey 的设计、预建分区和列族设计等问题；
4) 在规定时间内完成也就是存入速度不能过慢，并且当然是越快越好，使用BulkLoad。


## 5.3 HBase 中scan 对象的setCache 和setBatch 方法的使用？
**setCache 用于设置缓存，即设置一次RPC 请求可以获取多行数据**。对于缓存操作，如果行的数据量非常大，多行数据有可能超过客户端进程的内存容量，由此引入批量处理这一解决方案。

**setBatch 用于设置批量处理，批量可以让用户选择每一次ResultScanner 实例的next 操作要取回多少列**，例如，在扫描中设置setBatch(5)，则一次next()返回的Result 实例会包括5 列。如果一行包括的列数超过了批量中设置的值，则可以将这一行分片，每次next 操作返回一片，当一行的列数不能被批量中设置的值整除时，最后一次返回的Result 实例会包含比较少的列

组合使用扫描器缓存和批量大小，可以让用户方便地控制扫描一个范围内的行键所需要的RPC调用次数。==**Cache** 设置了服务器一次返回的**行数**，而**Batch** 设置了服务器一次返回的**列数**==。

## 5.4 HBase 中RowFilter 和BloomFilter 原理？
1) **RowFilter 原理简析**

**RowFilter 顾名思义就是对rowkey 进行过滤**，那么rowkey 的过滤无非就是相等
（ EQUAL ）、大于(GREATER) 、小于(LESS) ， 大于等于(GREATER_OR_EQUAL) ， 小于等于
(LESS_OR_EQUAL)和不等于(NOT_EQUAL)几种过滤方式。Hbase 中的RowFilter 采用比较符结合
比较器的方式来进行过滤。

> 比较器的类型如下：
> - BinaryComparator
> - BinaryPrefixComparator
> - NullComparator
> - BitComparator
> - RegexStringComparator
> - SubStringComparator

例子：

```java
Filter rowFilter = new RowFilter(CompareFilter.CompareOp.EQUAL,
new BinaryComparator(Bytes.toBytes(rowKeyValue)));
Scan scan=new Scan();
scan.setFilter(rowFilter) ...
```

在上面例子中，比较符为EQUAL，比较器为BinaryComparator

2) **BloomFilter 原理简析**

- **主要功能：提供随机读的性能**
- 存储开销：BloomFilter 是列族级别的配置，一旦表格中开启BloomFilter，那么在生成StoreFile时同时会生成一份包含BloomFilter 结构的文件MetaBlock，所以会增加一定的存储开销和内存开销
- 粒度控制：ROW 和ROWCOL

> **简单说一下BloomFilter 原理**：<br>
> 1) 内部是一个bit 数组，初始值均为0
> 2) 插入元素时对元素进行hash 并且映射到数组中的某一个index，将其置为1，再进行多次不同的hash 算法，将映射到的index 置为1，同一个index 只需要置1 次。
> 3) 查询时使用跟插入时相同的hash 算法，如果在对应的index 的值都为1，那么就可以认为该元
> 素可能存在，注意，**只是可能存在**
> 4) 所以BlomFilter **==只能保证过滤掉不包含的元素，对于是否包含存在误判==**

设置：在建表时对某一列设置BloomFilter 即可

## 5.5 Region 如何预建分区？
预分区的目的主要是在创建表的时候指定分区数，提前规划表有多个分区，以及每个分区的区间范围，这样在存储的时候rowkey 按照分区的区间存储，**可以避免region 热点问题**。

通常有两种方案：

- 方案1:shell 方法<br>
`create 'tb_splits', {NAME => 'cf',VERSIONS=> 3},{SPLITS => ['10','20','30']}`
- 方案2: JAVA 程序控制<br>
  - 取样，先随机生成一定数量的rowkey,将取样数据按升序排序放到一个集合里；
  - 根据预分区的region 个数，对整个集合平均分割，即是相关的splitKeys；
  - `HBaseAdmin.createTable(HTableDescriptor tableDescriptor,byte[][]splitkeys)`可以指定预分区的
splitKey，即是指定region 间的rowkey 临界值。

## 5.6 HRegionServer 宕机如何处理？
1) ZooKeeper 会监控HRegionServer 的上下线情况，当ZK 发现某个HRegionServer 宕机之
后会通知HMaster 进行失效备援；
2) 该HRegionServer 会停止对外提供服务，就是它所负责的region 暂时停止对外提供服务；
3) HMaster 会将该HRegionServer 所负责的region 转移到其他HRegionServer 上，并且会对
HRegionServer 上存在memstore 中还未持久化到磁盘中的数据进行恢复；
4) 这个恢复的工作是由WAL 重播来完成，这个过程如下：
    - wal 实际上就是一个文件，存在/hbase/WAL/对应RegionServer 路径下。
    - 宕机发生时，读取该RegionServer 所对应的路径下的wal 文件，然后根据不同的
region 切分成不同的临时文件recover.edits。
    - 当region 被分配到新的RegionServer 中，RegionServer 读取region 时会进行是否存在recover.edits，如果有则进行恢复。

## 5.7 HTable API 有没有线程安全问题，在程序中是单例还是多例？
在单线程环境下使用hbase 的htable 是没有问题，但是突然高并发多线程情况下就可能出现问题。

hbase 官方文档建议我们：**HTable 不是线程安全的**。建议使用同一个HBaseConfiguration
实例来创建多个HTable 实例，这样可以共享ZooKeeper 和socket 实例。
例如， 最好这样做：

```java
HBaseConfiguration conf=HBaseConfiguration.create();
HTable table1=new HTable(conf,"myTable");
HTable table2=new HTable(conf,"myTable");
```

而不是这样：

```java
HBaseConfiguration conf1=HBaseConfiguration.create();
HTable table1=new HTable(conf1,"myTable");
HBaseConfiguration conf2=HBaseConfiguration.create();
HTable table2=new HTable(conf2,"myTable");
```

当然**最方便的方法就是使用HTablepool 了，维持一个线程安全的map 里面存放的是tablename和其引用的映射**，可以认为是一个简单的计数器，当需要new 一个HTable 实例时直接从该pool 中取，用完放回。

## 5.8 Hbase 中的memstore 是用来做什么的？
hbase 为了保证随机读取的性能，所以Hfile 里面的rowkey 是有序的。当客户端的请求在到达regionserver 之后，**为了保证写入rowkey 的有序性，所以不能将数据立刻写入到Hfile 中，而是将每个变更操作保存在内存中，也就是memstore 中**。memstore 能够很方便的支持操作的随机插入，并保证所有的操作在内存中是有序的。当memstore 达到一定的量之后， 会将memstore 里面的数据flush 到Hfile 中，这样能充分利用hadoop 写入大文件的性能优势， 提高写入性能。

由于memstore 是存放在内存中，如果regionserver 因为某种原因死了，会导致内存中数据丢失。所有为了保证数据不丢失，hbase 将更新操作在写入memstore 之前会写入到一个write aheadlog(WAL)中。WAL 文件是追加、顺序写入的，WAL 每个regionserver 只有一个， 同一个regionserver上所有region 写入同一个的WAL 文件。这样当某个regionserver 失败时， 可以通过WAL 文件，将所有的操作顺序重新加载到memstore 中。

## 5.9 HBase 中region 太小和region 太大会造成什么问题？

- Region 过大会发生多次compaction，将数据读一遍并重写一遍到hdfs 上，占用磁盘io
- region 过小会造成多次split，region 会下线，影响访问服务，最佳的解决方法是调整`hbase.hregion.max.filesize`为256m。