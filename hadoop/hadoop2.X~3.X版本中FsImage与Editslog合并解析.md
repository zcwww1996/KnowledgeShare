[TOC]

# 1. 前言
我们知道HDFS是一个分布式文件存储系统，文件分布式存储在多个DataNode节点上。一个文件存储在哪些DataNode节点的哪些位置的元数据信息（metadata）由NameNode节点来处理。随着存储文件的增多，NameNode上存储的信息也会越来越多。那么HDFS是如何及时更新这些metadata的呢？

在HDFS中主要是通过两个组件FSImage和EditsLog来实现metadata的更新。在某次启动HDFS时，会从FSImage文件中读取当前HDFS文件的metadata，之后对HDFS的操作步骤都会记录到edit log文件中。比如下面这个操作过程

[![](https://i.loli.net/2021/03/05/riK7WE6skmjPzxA.jpg)](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwNjE1MjE1NjQ0Mzk1?x-oss-process=image/format,png)

那么完整的metadata信息就应该由FSImage文件和edit log文件组成。fsimage中存储的信息就相当于整个hdfs在某一时刻的一个快照。

FSImage文件和EditsLog文件可以通过ID来互相关联。在参数`dfs.namenode.name.dir`设置的路径下，会保存FSImage文件和EditsLog文件，如果是QJM方式HA的话，EditsLog文件保存在参数`dfs.journalnode.edits.dir`设置的路径下。

## 1.1 secondary namenode节点数据情况
先看下namenode节点的备份节点  secondary namenode节点

[![](https://i.loli.net/2021/03/05/RChV2Mwxtvf6r5B.png)](https://img-blog.csdnimg.cn/2019090216205890.png)

[![](https://i.loli.net/2021/03/05/mYyzDHheQuAKaVx.png)](https://img-blog.csdnimg.cn/20190902162139102.png)

[![](https://i.loli.net/2021/03/05/auOefB2dAbyh4k3.png)](https://img-blog.csdnimg.cn/20190902162355588.png)

# 2. FsImage和Editslog分别是什么 ？
- Editslog ：保存了所有对hdfs中文件的操作信息
- FsImage：是内存元数据在本地磁盘的映射，用于维护管理文件系统树，即元数据(metadata)

在hdfs中主要是通过两个数据结构FsImage和EditsLog来实现metadata的更新。在某次启动hdfs时，会从FSImage文件中读取当前HDFS文件的metadata，之后对HDFS的操作步骤都会记录到edit log文件中。

metadata信息就应该由FSImage文件和edit log文件组成。fsimage中存储的信息就相当于整个hdfs在某一时刻的一个快照。

FsImage文件和EditsLog文件可以通过ID来互相关联。如果是非HA集群的话，这两个数据文件保存在dfs.namenode.name.dir设置的路径下，会保存FsImage文件和EditsLog文件，如果是HA集群的话，EditsLog文件保存在参数dfs.journalnode.edits.dir设置的路径下，即edits文件由qjournal集群管理。

## 2.1 查看namenode节点 fsimage和editlog文件
查看一下hdfs.site.xml文件,找到 fsimage和editlog文件存储路径
![image](https://img-blog.csdnimg.cn/20191105210833431.png)

`cd /export/servershadoop-2.6.0-cdh5.14.0/hadoopDatas/namenodeDatas/current`

[![](https://z3.ax1x.com/2021/04/06/c11m1x.png)](https://img.imgdb.cn/item/606c28418322e6675ca69fc0.png)

### 2.1.1 查看Fsimage
文件是不能直接查看的,我们要先转化为XML文件,下载到本地,然后才可以查看
1) 转化文件格式

```bash
hdfs oiv -i fsimage_0000000000000001024 -p XML -o fs001.xml
```

[![fsimage.png](https://z3.ax1x.com/2021/04/06/c13P2t.png)](https://img.imgdb.cn/item/606c28ed8322e6675ca75084.png)

fsimage包含hadoop文件系统中的所有的目录和文件IDnode的序列化信息。

**对于文件来说，包含的信息有修改时间、访问时间、块大小和组成一个文件块信息等；对于目录来说，包含的信息主要有修改时间、访问控制权等信息。**

**fsimage并不包含DataNode的信息，而是==包含DataNode上块的映射信息==**，并存在内存中。当一个新的DataNode加入集群时，DataNode都会向NameNode提供块的信息，并且NameNode也会定期的获取块的信息，以便NameNode拥有最新的块映射信息。

又因为fsimage包含hadoop文件系统中的所有目录和文件IDnode的序列化信息，所有一旦fsimage出现丢失或者损坏的情况，那么即使DataNode上有块的数据，但是我们没有文件到块的映射关系，所以我们也是没有办法使用DataNode上的数据


### 2.1.2 查看editlog
```bash
hdfs oev -i  edits_0000000000000000011-0000000000000000309 -p XML -o edit002.xml
```

[![editlog.png](https://z3.ax1x.com/2021/04/06/c1G9jP.png)](https://img.imgdb.cn/item/606c2a1e8322e6675ca87bbb.png)

**edits文件存放的是hadoop文件系统的所有更新操作的记录，文件系统客户端执行的所有写操作首先会被记录到edits文件中。**

在NameNode启动之后，hdfs中的更新操作会重新写到edits文件中，因为fsimage文件一般情况下都是非常大的，可以达到GB级别甚至更高，如果所有的更新操作都向fsimage中添加的话，势必会导致系统运行的越来越慢。但是如果向edits文件中写的话就不会导致这样的情况出现，每次执行写操作后，且在向客户端发送成功代码之前，edits文件都需要同步更新的。

如果一个文件比较大，会使得写操作需要向多台机器进行操作，只有当所有的写操作都执行完成之后，写操作才可以称之为成功。这样的优势就是在任何的操作下都不会因为机器的故障而导致元数据不同步的情况出现。

### 2.1.3 current目录结构

[![](https://z3.ax1x.com/2021/04/06/c11m1x.png)](https://img.imgdb.cn/item/606c28418322e6675ca69fc0.png)

在上图中edit log文件以edits\_开头，后面跟一个txid范围段，并且多个edit log之间首尾相连，正在使用的edit log名字edits\_inprogress\_txid。该路径下还会保存两个fsimage文件（{dfs.namenode.num.checkpoints.retained}在namenode上保存的fsimage的数目，超出的会被删除。默认保存2个），文件格式为fsimage\_txid。上图中可以看出fsimage文件已经加载到了最新的一个edit log文件，仅仅只有inprogress状态的edit log未被加载。

在启动HDFS时，只需要读入fsimage\_0000000000000000058以及edits\_inprogress\_0000000000000000062就可以还原出当前hdfs的最新状况。

（FsImageid总是比editslogid小）

但是这里又会出现一个问题，如果edit log文件越来越多、越来越大时，当重新启动hdfs时，由于需要加载fsimage后再把所有的edit log也加载进来，就会出现第一段中出现的问题了。怎么解决？HDFS会采用checkpoing机制定期将edit log合并到fsimage中生成新的fsimage。这个过程就是接下来要讲的了。

那么这两个文件是如何合并的呢？这就引入了checkpoint机制

# 3. checkpoint机制
fsimage和edit log合并的过程如下图所示：

[![](https://i.loli.net/2021/03/05/EaI6TuWnP9iryDG.jpg)](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwNjE1MjIwMjI3NjQw?x-oss-process=image/format,png)

因为文件合并过程需要消耗io和cpu所以需要将这个过程独立出来，**在Hadoop1.x中是由Secondnamenode来完成，且Secondnamenode必须启动在单独的一个节点最好不要和namenode在同一个节点**，这样会增加namenode节点的负担，而且维护时也比较方便。同样在**HA集群中这个合并的过程是由Standby namenode完成的**。

其实这个合并过程是一个很耗I/O与CPU的操作，并且在进行合并的过程中肯定也会有其他应用继续访问和修改hdfs文件。所以，这个过程一般不是在单一的NameNode节点上进行从。如果HDFS没有做HA的话，checkpoint由SecondNameNode进程(一般SecondNameNode单独起在另一台机器上)来进行。在HA模式下，checkpoint则由StandBy状态的NameNode来进行。

什么时候进行checkpoint由两个参数`dfs.namenode.checkpoint.preiod`(默认值是3600，即1小时)和`dfs.namenode.checkpoint.txns`(默认值是1000000)来决定。period参数表示，经过1小时就进行一次checkpoint，txns参数表示，hdfs经过100万次操作后就要进行checkpoint了。这两个参数任意一个得到满足，都会触发checkpoint过程。进行checkpoint的节点每隔`dfs.namenode.checkpoint.check.period`(默认值是60）秒就会去统计一次hdfs的操作次数

## 3.1 合并的过程：过程类似于TCP协议的关闭过程（四次挥手）

首先Standbynamenode进行判断是否达到checkpoint的条件（是否距离上次合并过了1小时或者事务条数是否达到100万条）
当达到checkpoint条件后，Standbynamenode会将qjournal集群中的edits和本地fsImage文件合并生成一个文件fsimage\_ckpt\_txid（此时的txid是与合并的editslog\_txid的txid值相同），同时Standbynamenode还会生成一个MD5文件，并将fsimage\_ckpt\_txid文件重命名为fsimage\_txid
向Activenamenode发送http请求（请求中包含了Standbynamenode的域名，端口以及新fsimage\_txid的txid），询问是否进行获取
Activenamenode获取到请求后，会返回一个http请求来向Standbynamenode获取新的fsimage\_txid，并保存为fsimage.ckpt\_txid，生成一个MD5，最后再改名为fsimage\_txid。合并成功。
## 3.2 合并的时机：
什么时候进行checkpoint呢？这由两个参数 **`dfs.namenode.checkpoint.preiod`** (默认值是3600，即1小时)和 **`dfs.namenode.checkpoint.txns`** (默认值是1000000)来决定

1) 距离上次checkpoint的时间间隔 {dfs.namenode.checkpoint.period}

2) Edits中的事务条数达到{dfs.namenode.checkpoint.txns}限制，事物条数又由{dfs.namenode.checkpoint.check.period(默认值是60）}决定，checkpoint节点隔60秒就会去统计一次hdfs的操作次数。

在HA模式下checkpoint过程由StandBy NameNode来进行，以下简称为SBNN，Active NameNode简称为ANN。

HA模式下的edit log文件会同时写入多个JournalNodes节点的`dfs.journalnode.edits.dir`路径下，JournalNodes的个数为大于1的奇数，类似于Zookeeper的节点数，当有不超过一半的JournalNodes出现故障时，仍然能保证集群的稳定运行。

SBNN会读取FSImage文件中的内容，并且每隔一段时间就会把ANN写入edit log中的记录读取出来，这样SBNN的NameNode进程中一直保持着hdfs文件系统的最新状况namespace。当达到checkpoint条件的某一个时，就会直接将该信息写入一个新的FSImage

[![](https://i.loli.net/2021/03/05/QFLzGNJ1lWwbD8H.jpg)](https://imgconvert.csdnimg.cn/aHR0cDovL2ltZy5ibG9nLmNzZG4ubmV0LzIwMTYwNjE1MjIzNTUzODg4?x-oss-process=image/format,png)

如上图所示，主要由4个步骤：<br>
1) SBNN检查是否达到checkpoint条件：离上一次checkpoint操作是否已经有一个小时，或者HDFS已经进行了100万次操作。
2) SBNN检查达到checkpoint条件后，将该namespace以fsimage.ckpt\_txid格式保存到SBNN的磁盘上，并且随之生成一个MD5文件。然后将该fsimage.ckpt\_txid文件重命名为fsimage\_txid。
3) 然后SBNN通过HTTP联系ANN。
4)  ANN通过HTTP从SBNN获取最新的fsimage\_txid文件并保存为fsimage.ckpt\_txid，然后也生成一个MD5，将这个MD5与SBNN的MD5文件进行比较，确认ANN已经正确获取到了SBNN最新的fsimage文件。然后将fsimage.ckpt\_txid文件重命名为fsimage\_txit。

通过上面一系列的操作，SBNN上最新的FSImage文件就成功同步到了ANN上。

文章部分内容参考：<br>
[https://blog.csdn.net/q35445762/article/details/91472746](https://blog.csdn.net/q35445762/article/details/91472746)

[https://www.cnblogs.com/nucdy/p/5892144.html](https://www.cnblogs.com/nucdy/p/5892144.html)