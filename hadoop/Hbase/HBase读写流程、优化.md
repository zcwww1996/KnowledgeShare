[TOC]
# HBase存储架构和读写过程
[![Hbase架构图](http://img.hb.aicdn.com/384c814afb612dc6592514be3d23323da97a1b7f2c9f6-qVXoyu "Hbase架构图")](https://z3.ax1x.com/2021/04/15/c2Jhw9.png "Hbase架构图")

HBase的架构图上可以看出，HBase中的存储包括HMaster、HRegionServer、HRegion、Store、MemStore、StoreFile、HFileHLog等

HBase中的每张表都通过行键按照一定的范围被分割成多个子表（HRegion），默认一个HRegion超过256M就要被分割成两个，这个过程由HRegionServer管理，而HRegion的分配由HMaster管理。


## 读数据流程：

Client先访问zookeeper，从meta表读取region的位置，然后读取meta表中的数据。meta中又存储了用 户表的region信息。

根据namespace、表名和rowkey在meta表中找到对应的region信息

找到这个region对应的regionserver
 
查找对应的region
 
先从MemStore找数据，如果没有，再到StoreFile上读(为了读取的效率)

### 新旧版本不同
**0.94- 版本** Client访问用户数据之前需要首先访问zookeeper，然后访问-ROOT-表，接着访问.META.表，最后才能找到用户数据的位置去访问，中间需要多次网络操作，如下图0：

[![0.png](https://p.pstatp.com/origin/1385b0000fdda45216dde)](https://pic.downk.cc/item/5fed7c7c3ffa7d37b329e228.png "图0")

**0.96+ 版本** 删除了root 表，改为zookeeper里面的文件，如下图 1， 以读为例，寻址示意图如图2

[![1.png](https://p.pstatp.com/origin/ffdd0003288314786a43)](https://pic.downk.cc/item/5fed7c7c3ffa7d37b329e22a.png "图1")

[![2.png](https://p.pstatp.com/origin/138e90000067b7f65e8f6)](https://pic.downk.cc/item/5fed7c7c3ffa7d37b329e231.png "图2")

## 写数据流程：
Client先访问zookeeper，从meta表获取相应region信息，然后找到meta表的数据

根据namespace、表名和rowkey根据meta表的数据找到写入数据对应的region信息

找到对应的regionserver

把数据分别写到HLog和MemStore上一份
 
MemStore达到一个阈值后则把数据刷成一个StoreFile文件。(若MemStore中的数据有丢失，则可以在HLog上恢复)
 
当多个StoreFile文件达到一定的大小后，会触发Compact合并操作，合并为一个StoreFile，(这里同时进行版本的合并和数据删除。)

当Storefile大小超过一定阈值后，会把当前的Region分割为两个(Split)，并由Hmaster分配到相应的HRegionServer，实现负载均衡

来源： http://faq.edu360.cn/article/14



## 补充
**为什么Client只需要知道Zookeeper地址就可以了呢？**

HMaster启动的时候，会把Meta信息表加载到zookeeper。
 
Meta信息表存储了HBase所有的表，所有的Region的详细的信息，比如Region开始的key，结束的key，所在Regionserver的地址。Meta信息表就相当于一个目录，通过它，可以快速定位到数据的实际位置，所以读写数据，只需要跟Zookeeper对应的Regionserver进行交互就可以了。HMaster只需要存储Region和表的元数据信息，协调各个Regionserver，所以他的负载就小了很多。

HBase三大模块如何一起协作的。(HMaster,RegionServer,Zookeeper)

通过三个问题解释

* 当HBase启动的时候发生了什么？
* 如果Regionserver失效了，会发生什么？
* 当HMaster失效后会发生什么？

### 1.当HBase启动的时候发生了什么？
HMaster启动的时候会连接zookeeper，将自己注册到Zookeeper，首先将自己注册到Backup Master上，因为可能会有很多的节点抢占Master，最终的Active Master要看他们抢占锁的速度。
 
将会把自己从Backup Master删除，成为Active Master之后，才会去实例化一些类，比如Master Filesytem，table state manager
 
当一个节点成为Active Master之后，他就会等待Regionserver汇报。

首先Regionserver注册Zookeeper，之后向HMaster汇报。HMaster现在手里就有一份关于Regionserver状态的清单，对各个Regionserver（包括失效的）的数据进行整理，

最后HMaster整理出了一张Meta表，这张表中记录了，所有表相关的Region，还有各个Regionserver到底负责哪些数据等等。然后将这张表，交给Zookeeper。

之后的读写请求，都只需要经过Zookeeper就行了。

Backup Master 会定期同步 Active Master信息，保证信息是最新的。

### 2.如果Regionserver失效了，会发生什么？
如果某一个Regionserver挂了，HMaster会把该Regionserver删除，之后将Regionserver存储的数据，分配给其他的Regionserver，将更新之后meta表，交给Zookeeper
 所以当某一个Regionserver失效了，并不会对系统稳定性产生任何影响。

### 3.当HMaster失效后会发生什么？
如果Active 的HMaster出现故障

处于Backup状态的其他HMaster节点会推选出一个转为Active状态。

当之前出现故障的HMaster从故障中恢复，他也只能成为Backup HMaster，等待当前Active HMaster失效了，他才有机会重新成为Active HMaster

对于HA高可用集群，当Active状态的HMaster失效，会有处于Backup的HMaster可以顶上去，集群可以继续正常运行。

如果没有配置HA，那么对于客户端的新建表，修改表结构等需求，因为新建表，修改表结构，需要HMaster来执行，会涉及meta表更新。那么 会抛出一个HMaster 不可用的异常，但是不会影响客户端正常的读写数据请求。

# -ROOT-表和.META.表结构详解
由于HBase中的表可能非常大，故HBase会将表按行分成多个region，然后分配到多台RegionServer上。数据访问的整个流程如下图所示：
[![Hbase寻址](http://img.hb.aicdn.com/47d130857f4f6e307beb913805854862c44bcb9329a7a-jvmBT5_fw658 "Hbase寻址")](http://img.hb.aicdn.com/47d130857f4f6e307beb913805854862c44bcb9329a7a-jvmBT5_fw658 "Hbase寻址")

**注意两点**：
-   Client端在访问数据的过程中并没有涉及到Master节点，也就是说HBase日常的数据操作并不需要Master，不会造成Master的负担。
-   并不是每次数据访问都要执行上面的整个流程，因为很多数据都会被Cache起来。

从存储结构和操作方法的角度来说，-ROOT-、.META.与其他表没有任何区别。它们与众不同的地方是HBase用它们来存贮一个重要的系统信息：
-   -ROOT-：记录.META.表的Region信息。
-   .META.：记录用户表的Region信息。

其中-ROOT-表本身只会有一个region，这样保证了只需要三次跳转，就能定位到任意region

## 一、META表结构

在 HBase Shell 里对.META.表进行 scan 和 describe ：
![在 HBase Shell 里对.META.表进行 scan 和 describe](http://img.hb.aicdn.com/f375872d67232418b9057a369fa40a57ad55aa1be27a-IbXuTM_fw658 "在 HBase Shell 里对.META.表进行 scan 和 describe")
可以看出，.META.表的结构如下：
![.META.表的结构](http://img.hb.aicdn.com/8e2da2e733d948a5c705f14e79e09e96cbb863181245e-LbhEC1_fw658 ".META.表的结构")
.META.表中每一行记录了一个Region的信息。

1) RowKey

RowKey就是Region Name，它的命名形式是TableName,StartKey,TimeStamp.Encoded.。

其中 Encoded 是TableName,StartKey,TimeStamp的md5值。

例如：

```java
mytable,,1438832261249.ea2b47e1eba6dd9a7121315cdf0e4f67.
```

表名是mytable，StartKey为空，时间戳是1438832261249，前面三部分的md5是：

```java
$ echo -n "mytable,,1438832261249" | md5sum   # -n选项表示不输出换行符
ea2b47e1eba6dd9a7121315cdf0e4f67  -
```
2) Column Family

.META.表有两个Column Family：info 和 historian。

其中info包含了三个Column：

-  regioninfo：region的详细信息，包括StartKey、EndKey以及Table信息等等。
-  server：管理该region的 RegionServer 的地址。
-  serverstartcode：RegionServer 开始托管该region的时间。

至于historian：

    That was a family used to keep track of region operations like open, 
    close, compact, etc. It proved to be more troublesome than handy so we 
    disabled this feature until coming up with a better solution. The 
    family stayed for backward compatibility.

大致的意思是：这个Column Family是用来追踪一些region操作的，例如open、close、compact等。事实证明这非常的麻烦，所以在想出一个更好的解决方案之前我们禁用了此功能。这个列族会保持向后兼容。

综上所述，.META.表中保存了所有用户表的region信息，在进行数据访问时，它是必不可少的一个环节。当Region被拆分、合并或者重新分配的时候，都需要来修改这张表的内容 来保证访问数据时能够正确地定位region。
## 二、ROOT表结构

当用户表特别大时，用户表的region也会非常多。.META.表存储了这些region信息，也变得非常大，这时.META.自己也需要划分成多个Region，托管到多个RegionServer上。

这时就出现了一个问题：**当.META.被托管在多个RegionServer上，如何去定位.META.呢**？

HBase的做法是用另外一个表来记录.META.的Region信息，就和.META.记录用户表的Region信息一样，这个表就是-ROOT-表。

在 HBase Shell 里对-ROOT-表进行 scan 和 describe ：
![在 HBase Shell 里对-ROOT-表进行 scan 和 describe ](http://img.hb.aicdn.com/8cbfb4c39c2562ef948fcc76b7d49566bc33057dcdd6-4xLMQb_fw658 "在 HBase Shell 里对-ROOT-表进行 scan 和 describe ")
-ROOT-表的结构如下：
![-ROOT-表的结构](http://img.hb.aicdn.com/5f2e0243328f575cff3c59d8005e148100067971fa26-FUHRPR_fw658 "-ROOT-表的结构")
可以看出，除了没有historian列族之外，-ROOT-表的结构与.META.表的结构是一样的。另外，-ROOT-表的 RowKey 没有采用时间戳，也没有Encoded值，而是直接指定一个数字。

-ROOT-表永远只有一个Region，也就只会存放在一台RegionServer上。—— 在进行数据访问时，需要知道管理-ROOT-表的RegionServer的地址。**这个地址被存在 ZooKeeper 中**。

# Hbase优化操作与建议

## 一、服务端调优
1. hbase.regionserver.handler.count：该设置决定了处理RPC的线程数量，默认值是10，通常可以调大，比如：150，当请求内容很大（上MB，比如大的put、使用缓存的scans）的时候，如果该值设置过大则会占用过多的内存，导致频繁的GC，或者出现OutOfMemory，因此该值不是越大越好。
2. hbase.hregion.max.filesize ：配置region大小，0.94.12版本默认是10G，region的大小与集群支持的总数据量有关系，如果总数据量小，则单个region太大，不利于并行的数据处理，如果集群需支持的总数据量比较大，region太小，则会导致region的个数过多，导致region的管理等成本过高，如果一个RS配置的磁盘总量为3T*12=36T数据量，数据复制3份，则一台RS服务器可以存储10T的数据，如果每个region最大为10G，则最多1000个region，如此看，94.12的这个默认配置还是比较合适的，不过如果要自己管理split，则应该调大该值，并且在建表时规划好region数量和rowkey设计，进行region预建，做到一定时间内，每个region的数据大小在一定的数据量之下，当发现有大的region，或者需要对整个表进行region扩充时再进行split操作，一般提供在线服务的hbase集群均会弃用hbase的自动split，转而自己管理split。
3. hbase.hregion.majorcompaction：配置major合并的间隔时间，默认为1天，可设置为0，禁止自动的major合并，可手动或者通过脚本定期进行major合并，有两种compact：minor和major，minor通常会把数个小的相邻的storeFile合并成一个大的storeFile，minor不会删除标示为删除的数据和过期的数据，major会删除需删除的数据，major合并之后，一个store只有一个storeFile文件，会对store的所有数据进行重写，有较大的性能消耗。
4. hbase.hstore.compactionThreshold：HStore的storeFile数量>= compactionThreshold配置的值，则可能会进行compact，默认值为3，可以调大，比如设置为6，在定期的major compact中进行剩下文件的合并。
5. hbase.hstore.blockingStoreFiles：HStore的storeFile的文件数大于配置值，则在flush memstore前先进行split或者compact，除非超过hbase.hstore.blockingWaitTime配置的时间，默认为7，可调大，比如：100，避免memstore不及时flush，当写入量大时，触发memstore的block，从而阻塞写操作。
6. hbase.regionserver.global.memstore.upperLimit：默认值0.4，RS所有memstore占用内存在总内存中的upper比例，当达到该值，则会从整个RS中找出最需要flush的region进行flush，直到总内存比例降至该数限制以下，并且在降至限制比例以下前将阻塞所有的写memstore的操作，在以写为主的集群中，可以调大该配置项，不建议太大，因为block cache和memstore cache的总大小不会超过0.8，而且不建议这两个cache的大小总和达到或者接近0.8，避免OOM，在偏向写的业务时，可配置为0.45，memstore.lowerLimit保持0.35不变，在偏向读的业务中，可调低为0.35，同时memstore.lowerLimit调低为0.3，或者再向下0.05个点，不能太低，除非只有很小的写入操作，如果是兼顾读写，则采用默认值即可。
7. hbase.regionserver.global.memstore.lowerLimit：默认值0.35，RS的所有memstore占用内存在总内存中的lower比例，当达到该值，则会从整个RS中找出最需要flush的region进行flush，配置时需结合memstore.upperLimit和block cache的配置。
8. file.block.cache.size：RS的block cache的内存大小限制，默认值0.25，在偏向读的业务中，可以适当调大该值，具体配置时需试hbase集群服务的业务特征，结合memstore的内存占比进行综合考虑。
9. hbase.hregion.memstore.flush.size：默认值128M，单位字节，超过将被flush到hdfs，该值比较适中，一般不需要调整。
10. hbase.hregion.memstore.block.multiplier：默认值2，如果memstore的内存大小已经超过了hbase.hregion.memstore.flush.size的2倍，则会阻塞memstore的写操作，直到降至该值以下，为避免发生阻塞，最好调大该值，比如：4，不可太大，如果太大，则会增大导致整个RS的memstore内存超过memstore.upperLimit限制的可能性，进而增大阻塞整个RS的写的几率。如果region发生了阻塞会导致大量的线程被阻塞在到该region上，从而其它region的线程数会下降，影响整体的RS服务能力，例如：

开始阻塞：
    ![Hbase-开始阻塞](https://images2017.cnblogs.com/blog/932932/201710/932932-20171029144925445-975971538.png "Hbase-开始阻塞")

解开阻塞：
![Hbase-解开阻塞](http://dl2.iteye.com/upload/attachment/0097/2291/1dca9b9f-0f2c-3a60-8382-db9bb932f376.png "Hbase-解开阻塞")

从10分11秒开始阻塞到10分20秒解开，总耗时9秒，在这9秒中无法写入，并且这期间可能会占用大量的RS handler线程，用于其它region或者操作的线程数会逐渐减少，从而影响到整体的性能，也可以通过异步写，并限制写的速度，避免出现阻塞。
11. hfile.block.index.cacheonwrite：在index写入的时候允许put无根（non-root）的多级索引块到block cache里，默认是false，设置为true，或许读性能更好，但是是否有副作用还需调查。
12. io.storefile.bloom.cacheonwrite：默认为false，需调查其作用。
13. hbase.regionserver.regionSplitLimit：控制最大的region数量，超过则不可以进行split操作，默认是Integer.MAX，可设置为1，禁止自动的split，通过人工，或者写脚本在集群空闲时执行。如果不禁止自动的split，则当region大小超过hbase.hregion.max.filesize时会触发split操作（具体的split有一定的策略，不仅仅通过该参数控制，前期的split会考虑region数据量和memstore大小），每次flush或者compact之后，regionserver都会检查是否需要Split，split会先下线老region再上线split后的region，该过程会很快，但是会存在两个问题：1、老region下线后，新region上线前client访问会失败，在重试过程中会成功但是如果是提供实时服务的系统则响应时长会增加；2、split后的compact是一个比较耗资源的动作。
14. Jvm调整
    1) 内存大小:master默认为1G，可增加到2G，regionserver默认1G，可调大到10G，或者更大，zk并不耗资源，可以不用调整；
    2) 垃圾回收:待研究。
      
## 二、开发调优

1. 列族、rowkey要尽量短，每个cell值均会存储一次列族名称和rowkey，甚至列名称也要尽量短
2. RS的region数量：一般每个RegionServer不要过1000，过多的region会导致产生较多的小文件，从而导致更多的compact，当有大量的超过5G的region并且RS总region数达到1000时，应该考虑扩容。
3. 建表时：
   1) 如果不需要多版本，则应设置version=1；
   2) 开启lzo或者snappy压缩，压缩会消耗一定的CPU，但是，磁盘IO和网络IO将获得极大的改善，大致可以压缩4~5倍；
   3) 合理的设计rowkey，在设计rowkey时需充分的理解现有业务并合理预见未来业务，不合理的rowkey设计将导致极差的hbase操作性能；
   4) 合理的规划数据量，进行预分区，避免在表使用过程中的不断split，并把数据的读写分散到不同的RS，充分的发挥集群的作用；
   5) 列族名称尽量短，比如：“f”，并且尽量只有一个列族；
   6) 视场景开启bloomfilter，优化读性能。
      
## 三、Client端调优

1. hbase.client.write.buffer：写缓存大小，默认为2M，推荐设置为6M，单位是字节，当然不是越大越好，如果太大，则占用的内存太多；
2. hbase.client.scanner.caching：scan缓存，默认为1，太小，可根据具体的业务特征进行配置，原则上不可太大，避免占用过多的client和rs的内存，一般最大几百，如果一条数据太大，则应该设置一个较小的值，通常是设置业务需求的一次查询的数据条数，比如：业务特点决定了一次最多100条，则可以设置为100
3. 设置合理的超时时间和重试次数
4. client应用**读写分离**
   1) 读和写分离，位于不同的tomcat实例，数据先写入redis队列，再异步写入hbase，如果写失败再回存redis队列，先读redis缓存的数据（如果有缓存，需要注意这里的redis缓存不是redis队列），如果没有读到再读hbase。
   
   2) 当hbase集群不可用，或者某个RS不可用时，因为HBase的重试次数和超时时间均比较大（为保证正常的业务访问，不可能调整到比较小的值，如果一个RS挂了，一次读或者写，经过若干重试和超时可能会持续几十秒，或者几分钟），所以一次操作可能会持续很长时间，导致tomcat线程被一个请求长时间占用，tomcat的线程数有限，会被快速占完，导致没有空余线程做其它操作，读写分离后，写由于采用先写redis队列，再异步写hbase，因此不会出现tomcat线程被占满的问题， 应用还可以提供写服务，如果是充值等业务，则不会损失收入，并且读服务出现tomcat线程被占满的时间也会变长一些，如果运维介入及时，则读服务影响也比较有限。
5. 如果把org.apache.hadoop.hbase.client.HBaseAdmin配置为spring的bean，则需配置为懒加载，避免在启动时链接hbase的Master失败导致启动失败，从而无法进行一些降级操作。
6. Scan查询编程优化：
   1) 调整caching；
   2) 如果是类似全表扫描这种查询，或者定期的任务，则可以设置scan的setCacheBlocks为false，避免无用缓存；
   3) 关闭scanner，避免浪费客户端和服务器的内存；
   4) 限定扫描范围：指定列簇或者指定要查询的列；
   5) 如果只查询rowkey时，则使用KeyOnlyFilter可大量减少网络消耗；

*作为hbase依赖的状态协调者ZK和数据的存储则HDFS，也需要调优：*

## 四、ZK调优
1. zookeeper.session.timeout：默认值3分钟，不可配置太短，避免session超时，hbase停止服务，线上生产环境由于配置为1分钟，出现过2次该原因导致的hbase停止服务，也不可配置太长，如果太长，当rs挂掉，zk不能快速知道，从而导致master不能及时对region进行迁移。

2. zookeeper数量：至少5个节点。给每个zookeeper 1G左右的内存，最好有独立的磁盘。 (独立磁盘可以确保zookeeper不受影响).如果集群负载很重，不要把Zookeeper和RegionServer运行在同一台机器上面。就像DataNodes 和 TaskTrackers一样，只有超过半数的zk存在才会提供服务，比如：共5台，则最多只运行挂2台，配置4台与3台一样，最多只运行挂1台。

3. hbase.zookeeper.property.maxClientCnxns：zk的最大连接数，默认为300，可配置上千

## 五、hdfs调优
1. dfs.name.dir： namenode的数据存放地址，可以配置多个，位于不同的磁盘并配置一个NFS远程文件系统，这样nn的数据可以有多个备份
2. dfs.data.dir：dn数据存放地址，每个磁盘配置一个路径，这样可以大大提高并行读写的能力
3. dfs.namenode.handler.count：nn节点RPC的处理线程数，默认为10，需提高，比如：60
4. dfs.datanode.handler.count：dn节点RPC的处理线程数，默认为3，需提高，比如：20
5. dfs.datanode.max.xcievers：dn同时处理文件的上限，默认为256，需提高，比如：8192
6. dfs.block.size：dn数据块的大小，默认为64M，如果存储的文件均是比较大的文件则可以考虑调大，比如，在使用hbase时，可以设置为128M，注意单位是字节
7. dfs.balance.bandwidthPerSec：在通过start-balancer.sh做负载均衡时控制传输文件的速度，默认为1M/s，可配置为几十M/s，比如：20M/s
8. dfs.datanode.du.reserved：每块磁盘保留的空余空间，应预留一些给非hdfs文件使用，默认值为0
9. dfs.datanode.failed.volumes.tolerated：在启动时会导致dn挂掉的坏磁盘数量，默认为0，即有一个磁盘坏了，就挂掉dn，可以不调整。

本文档转载自 [HBase性能调优](http://itindex.net/detail/49632-hbase-%E6%80%A7%E8%83%BD%E8%B0%83%E4%BC%98 "HBase性能调优")