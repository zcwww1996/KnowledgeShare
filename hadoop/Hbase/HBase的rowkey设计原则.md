[TOC]

HBase是三维有序存储的，通过rowkey（行键），column key（column family和qualifier）和TimeStamp（时间戳）这个三个维度可以对HBase中的数据进行快速定位。

# 一、hbase查询方式
HBase中rowkey可以唯一标识一行记录，在HBase查询的时候，有以下几种方式：

1. 通过get方式，指定rowkey获取唯一一条记录
2. 通过scan方式，设置startRow和stopRow参数进行范围匹配
3. 全表扫描，即直接扫描整张表中所有行记录

# 二、rowkey设计原则
## (一) rowkey长度原则
rowkey是一个二进制码流，可以是任意字符串，最大长度 64kb ，实际应用中一般为10-100bytes，以 byte[] 形式保存，**<font color="DarkRed">一般设计成定长</font>**。

**<font color="DarkRed">建议越短越好，不要超过16个字节</font>**，原因如下：

1. 数据的持久化文件HFile中是按照KeyValue存储的，如果rowkey过长，比如超过100字节，1000w行数据，光rowkey就要占用100*1000w=10亿个字节，将近1G数据，这样会极大影响HFile的存储效率；
2. MemStore将缓存部分数据到内存，如果rowkey字段过长，内存的有效利用率就会降低，系统不能缓存更多的数据，这样会降低检索效率。
3. 目前操作系统都是64位系统，内存8字节对齐，控制在16个字节，8字节的整数倍利用了操作系统的最佳特性。

## (二) rowkey散列原则
如果rowkey按照时间戳的方式递增，<font color="DarkRed">**不要将时间放在二进制码的前面**</font>，建议将rowkey的**高位作为散列字段**，由程序随机生成，**低位放时间字段**，这样将提高数据均衡分布在每个RegionServer，以实现负载均衡的几率。如果没有散列字段，首字段直接是时间信息，所有的数据都会集中在一个RegionServer上，这样在数据检索的时候负载会集中在个别的RegionServer上，造成<font style="background-color:SkyBlue;color: red;">**热点问题**</font>，会降低查询效率。

## (三) rowkey唯一原则
必须在设计上保证其唯一性，rowkey是按照字典顺序排序存储的，因此，设计rowkey的时候，要充分利用这个排序的特点，将经常读取的数据存储到一块，将最近可能会被访问的数据放到一块。


# 三、补充：热点
## (一) 什么是热点
HBase中的行是按照rowkey的字典顺序排序的，这种设计优化了scan操作，可以将相关的行以及会被一起读取的行存取在临近位置，便于scan。然而糟糕的rowkey设计是热点的源头。

**热点发生在大量的client直接访问集群的一个或极少数个节点(访问可能是读，写或者其他操作)**。大量访问会使热点region所在的单个机器超出自身承受能力，引起性能下降甚至region不可用，这也会影响同一个RegionServer上的其他region，由于主机无法服务其他region的请求。 设计良好的数据访问模式以使集群被充分，均衡的利用。

在更多数据情况下，数据应该被写入集群的多个region，而不是一个。


## (二) 避免热点的方法
参考：https://blog.csdn.net/qq_32727095/article/details/114025881

### 预分区
region应该进行预分区，建表的时候就预分区(分区就是region)，分区语句例如：

```sql
create ‘test_region’,‘info’,‘info2’,SPLITS => [‘1’,‘2’,‘3’,‘4’,‘5’,‘6’,‘7’,‘8’,‘9’]
```
此时我们region个数是十个，Regin的取值范围分别是：

```bash
[无穷小, 1)
[1, 2)
[2, 3)
…
[9, 无穷大]
```
==**左闭右开区间**==

**hbase默认是字典排序，你的数据到底放到哪个region中，关键在于你的rowkey第一个元素**，如果元素开头是3，那么会放在region4中


一台服务器最好放2~3个region即可，也就是说region个数最好等于服务器总数*3 即可

> 我们既然预分区为10个region，hbase还会自动给我们region分区吗?
> 
> 答案是：会的<br>
> 当我们某一个region大于了默认阈值10G后，此region会被hbase自动分区;

下面是一些常见的避免热点的方法以及它们的优缺点：
### 加盐

这里所说的加盐不是密码学中的加盐，而是<font color="DarkRed">**在rowkey的前面增加随机数**</font>，具体就是给rowkey分配一个随机前缀以使得它和之前的rowkey的开头不同。分配的前缀种类数量应该和你想使用数据分散到不同的region的数量一致。加盐之后的rowkey就会根据随机生成的前缀分散到各个region上，以避免热点。

### 哈希（hase）

哈希会使同一行永远用一个前缀加盐。哈希也可以使负载分散到整个集群，但是读取却是可以预测的。使用确定的哈希可以让客户端重构完整的rowkey，可以使用get操作准确获取某一个行数据

#### 按分区取余
`rowkey(手机号) % 分区数 取余 做为rowkey的第一个元素`

```bash
0_18518885160
1_11111111111
等等
```

这样rowkey的首个元素就会将此rowkey放置到对应的region中，后期取数也是手机号 % 分区数 取余 + 原本rowkey 即可获取数据；

**此时相同的手机号还放在了同一个region中，解决了跨region读取数据问题**


#### 让rowkey根据最近时间排序
我们可以再在后面加上Long.maxValue - 时间戳

```bash
0_18518885160_Long.maxValue - 时间戳
1_111111111111_Long.maxValue - 时间戳
```

时间戳是递增的，而Long.maxValue是固定的，所以时间戳越大，此时的后缀值就越小，那么也就最靠前，即使后面region分裂也不影响，为什么呢，因为即使一个region分裂为两个region；
这两个region范围和，依然是之前的一个region的范围：

比如，region [2,3) 分裂了，假设分裂后的两个region是

```bash
Region1
[2,2555555555)
Region2:
[2555555555,3)
```


### 反转（reverse）

第三种防止热点的方法是反转固定长度或者数字格式的rowkey。这样可以使得rowkey中经常改变的部分（最没有意义的部分）放在前面。这样可以有效的随机rowkey，但是牺牲了rowkey的有序性。

反转rowkey的例子以手机号为rowkey，可以将手机号反转后的字符串作为rowkey，取数据的时候先将手机号倒序去查找表中的region即可找到。这样就避免了以手机号那样比较固定数字开头导致的热点问题

### 时间戳反转

一个常见的数据处理问题是快速获取数据的最近版本，使用反转的时间戳作为rowkey的一部分，对这个问题十分有用，可以用 Long.Max_Value - timestamp 追加到key的末尾，例如 [key][reverse_timestamp] , [key] 的最新值可以通过scan [key]获得[key]的第一条记录，因为HBase中rowkey是有序的，第一条记录是最后录入的数据。

比如需要保存一个用户的操作记录，按照操作时间倒序排序，在设计rowkey的时候，可以这样设计

[userId反转][Long.Max_Value - timestamp]，在查询用户的所有操作记录数据的时候，直接指定反转后的userId，startRow是[userId反转][000000000000],stopRow是[userId反转][Long.Max_Value - timestamp]

如果需要查询某段时间的操作记录，startRow是[user反转][Long.Max_Value - 起始时间]，stopRow是[userId反转][Long.Max_Value - 结束时间]



## 四、其他一些建议

- 尽量减少行和列的大小在HBase中，value永远和它的key一起传输的。当具体的值在系统间传输时，它的rowkey，列名，时间戳也会一起传输。如果你的rowkey和列名很大，甚至可以和具体的值相比较，那么你将会遇到一些有趣的问题。HBase storefiles中的索引（有助于随机访问）最终占据了HBase分配的大量内存，因为具体的值和它的key很大。可以增加block大小使得storefiles索引再更大的时间间隔增加，或者修改表的模式以减小rowkey和列名的大小。压缩也有助于更大的索引。
- <font style="background-color: yellow;color: blue;">**列族尽可能越短越好，最好是一个字符**</font>
- 冗长的属性名虽然可读性好，但是更短的属性名存储在HBase中会更好

