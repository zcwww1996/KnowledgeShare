[TOC]

来源：https://blog.csdn.net/martin_liang/article/details/82531262

参考：https://www.cnblogs.com/lhfcws/p/7828811.html

MVCC (Multiversion Concurrency Control)，即多版本并发控制技术，它使得大部分支持行锁的事务引擎，不再单纯的使用行锁来进行数据库的并发控制，取而代之的是，把数据库的行锁与行的多个版本结合起来，只需要很小的开销，就可以实现非锁定读，从而大大提高数据库系统的并发性能。

**HBase正是通过行锁+MVCC保证了高效的并发读写**。

# 为什么需要并发控制

HBase系统本身只能保证单行的ACID特性。ACID的含义是：

- 原子性(Atomicity)
- 一致性(Consistency)
- 隔离性(Isolation)
- 持久性(Durability)

传统的关系型数据库一般都提供了跨越所有数据的ACID特性；为了性能考虑，HBase只提供了基于单行的ACID。

下面是一个hbase并发写的例子。

原始数据如下  
[![](http://pic.yupoo.com/flying5_v/DLZa6LA5/9xovz.png)](https://s3.jpg.cm/2021/04/29/isFL4.png)

从[Apache HBase Write Path](http://blog.cloudera.com/blog/2012/06/hbase-write-path/)一文可以知道hbase写数据是分为两步：  
1) 写Write-Ahead-Log(WAL)文件  
2) 写MemStore：将每个cell\[(row,column)对\]的数据写到内存中的memstore

# 写写同步

假定对写没有采取并发控制，并考虑以下的顺序：

[![](http://pic.yupoo.com/flying5_v/DLZa70Tr/kdDM4.png)](https://s3.jpg.cm/2021/04/29/ispDp.png)

最终得到的结果是：

[![](http://pic.yupoo.com/flying5_v/DLZ9WzdB/CeUhw.png)](https://s3.jpg.cm/2021/04/29/iss2G.png)

这样就得到了不一致的结果。显然我们需要对并发写操作进行同步。  
最简单的方式是提供一个基于行的独占锁来保证对同一行写的独立性。所以写的顺序是：

- (0) 获取行锁
- (1) 写WAL文件
- (2) 更新MemStore：将每个cell写入到memstore
- (3) 释放行锁

# 读写同步

尽管对并发写加了锁，但是对于读呢？见下面的例子：

[![](http://pic.yupoo.com/flying5_v/DLZa6mPT/COfD0.png)](https://s3.jpg.cm/2021/04/29/iss2G.png)

如果在上面的图中红线所示的地方进行读操作，最终得到的结果是： 

[![](http://pic.yupoo.com/flying5_v/DLZa6VD0/15jfbe.png)](https://s3.jpg.cm/2021/04/29/ismoD.png)

可见需要对读和写也进行并发控制，不然会得到不一致的数据。最简单的方案就是读和写公用一把锁。这样虽然保证了ACID特性，但是读写操作同时抢占锁会互相影响各自的性能。

# MVCC算法

HBase采用了MVCC算法来避免读操作去获取行锁。

对于写操作：

- (w1) 获取行锁后，每个写操作都立即分配一个写序号
- (w2) 写操作在保存每个数据cell时都要带上写序号
- (w3) 写操作需要申明以这个写序号来完成本次写操作

对于读操作:

- (r1) 每个读操作开始都分配一个读序号，也称为读取点
- (r2) 读取点的值是所有的写操作完成序号中的最大整数(所有的写操作完成序号<=读取点)
- (r3) 对某个(row,column)的读取操作r来说，结果是满足写序号为“写序号<=读取点这个范围内”的最大整数的所有cell值的组合

在采用MVCC后的数据执行图：

[![](http://pic.yupoo.com/flying5_v/DLZa6ANo/MaIj7.png)](https://s3.jpg.cm/2021/04/29/iswV6.png)

注意到采用MVCC算法后，每一次写操作都有一个写序号(即w1步)，每个cell数据写memstore操作都有一个写序号(w2，例如：“Cloudera \[wn=1\]”))，并且每次写操作完成也是基于这个写序号(w3)。

如果在“Restaurant \[wn=2\]” 这步之后，“Waiter \[wn=2\]”这步之前，开始一个读操作。根据规则r1和r2，读的序号为1。根据规则3，读操作以序号1读到的值是：

[![](http://pic.yupoo.com/flying5_v/DLZa7x0z/jVAMs.png)](https://s3.jpg.cm/2021/04/29/isrNu.png)

这样就实现了以无锁的方式读取到一致的数据了。

重新总结下MVCC算法下写操作的执行流程：

- (0) 获取行锁
  - **(0-a) 获取写序号**
- (1) 写WAL文件
- (2) 更新MemStore：将每个cell写入到memstore,如果写中间失败则回滚，否则则当做成功继续执行
  - **(2-a) 以写序号完成操作**
- (3) 释放行锁

> 补充：<br>
> 一般我们说先记录在预写日志(wal),然后再写入缓存(memstore),实际上我们从源码中可以发现有一些小小的偏差.
> 
> 实际操作顺序应该是:
> 1) hbase做写操作时,先记录在本地的wal(Write-Ahead logfile)中,但是不同步到hdfs
> 2) 然后再把数据写入到memstore
> 3) 开始将wal同步到hdfs
> 4) 最后如果wal同步成功则结束,如果同步失败则回滚memstore

本文是基于HBase 0.92. 在HBase 0.94中会有些优化策略，比如 [HBASE-5541](https://issues.apache.org/jira/browse/HBASE-5541) 提到的。

英文原文：[https://blogs.apache.org/hbase/entry/apache\_hbase\_internals\_locking\_and](https://blogs.apache.org/hbase/entry/apache_hbase_internals_locking_and)