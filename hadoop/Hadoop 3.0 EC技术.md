[TOC]

# 1. EC的设计目标


*   Hadoop默认的3副本方案需要额外的200%的存储空间、和网络IO开销
*   而一些较低I/O的warn和cold数据，副本数据的访问是比较少的（hot数据副本会被用于计算）
*   EC可以提供同级别的容错能力，存储空间要少得多（官方宣传不到50%），使用了EC，副本始终为1

# 2. EC背景


## 2.1 EC在RAID应用

*   EC在RAID也有应用，RAID通过EC将文件划分为更小的单位，例如：可以按照bit、byte或者block来划分。
*   然后将这些条纹单元存储在不同的磁盘中

> 条纹单元：官方称之为Stripe Unit，我把它隐喻为斑马身上的黑白条纹，就称每个文件经过EC处理后的就是一个个的条纹单元。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204740.png)](https://z3.ax1x.com/2021/11/10/Ia0kdO.png)

EC编码奇偶校验单元

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204746.png)](https://z3.ax1x.com/2021/11/10/Ia0PL6.png)

根据剩余条纹单元和奇偶校验单元恢复数据。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204742.png)](https://z3.ax1x.com/2021/11/10/Ia0Csx.png)

## 2.2 EC与HDFS

一个具有6个块，3副本会消耗6 x 3 = 18个块存储空间。而EC只需要 6个Block，再加上3个奇偶校验，仅需要6 + 3 = 9个块。节省了一半的存储空间。

# 3. EC在Hadoop架构的调整


使用EC有几个重要优势：

1.  Online-EC，在写入数据的时候就是以EC方式写入的，而不是先存完数据再开始进行EC编码处理（offline-EC）。
2.  Online-EC将一个小文件分发到多个DataNode，而不是将多个文件放到一个编码组中。这样，删除数据、Qutoa、数据迁移是更容易的。

## 3.1 NameNode元数据存储

基于EC的文件存储与Hadoop经典分块存储方式做了调整。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204752.png)](https://z3.ax1x.com/2021/11/10/Ia0FeK.png)

基于条纹的HDFS存储逻辑上是由Block Group（块组组成），每个Block Group包含了一定数量的Internal Block（后续我们称为EC Block）。如果一个文件有很多的EC Block，会占用NameNode较大的内存空间。HDFS引入了新的分层Block命名协议，通过Block的ID可以推断出Block Group的ID，NameNode是基于Block Group而不是EC Block级别管理。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204755.png)](https://z3.ax1x.com/2021/11/10/Ia0AoD.png)

## 3.2 Client

客户端读取、写入HDFS也做了调整，当以Online-EC写入一个文件时，是以并行方式来处理Block Group中的Internal Block。

*   当写入文件时，数据流通过DFSStripedOutputStream实现，会管理一组数据流，每个DataNode节点都对应一个数据流，并将Block Group的一个Internal Block存储。这些流操作都是异步进行的。还有一个Coordinator，负责一个文件的所有Block Group操作，包括：结束当前的Block Group、分配新的Block Group等。
*   当读取文件时，数据流通过DFSStripedInputStream实现，它将请求的文件转换为存储在DataNode的Internal Block，然后进行并行读取，出现故障时，发出奇偶校验数据请求来进行解码恢复Internal Block。

## 3.3 DataNode

DataNode上运行一个ErasureCodingWorker（ECWorker）任务，专门用于失败的EC Block进行后台数据恢复。一旦NameNode检测到失败的EC Block，NameNode会选择一个DataNode进行数据恢复。

*   先从错误数据所在的Block Group数据节点中读取正常的EC Block作为输入，基于EC策略，通过最小的EC Block进行数据恢复。
*   从输入的EC Block以及奇偶校验码块进行解码数据恢复，生成正常的EC Block。
*   EC解码完成后，将恢复的EC Block传输到对应的DataNode中。

# 4. EC存储方案


## 4.1 EC编码和解码

EC编解码器是对EC Block上的条纹单元进行处理。编码器将EC Block中的多个条纹单元作为输入，并输出许多奇偶校验单元。这个过程称为编码。条纹单元和奇偶校验单元称为EC编码组。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204801.png)](https://z3.ax1x.com/2021/11/10/Ia0VFe.png)

解码的过程就是恢复数据的过程，可以通过剩余的条纹单元和奇偶校验单元来恢复数据。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204808.png)](https://z3.ax1x.com/2021/11/10/Ia0ZJH.png)

## 4.2 容错性和存储效率

在比较不同的存储方案时，需要考虑两个重要因素：

*   数据的容错性（最多可允许同时出现多少故障来衡量，容错性越大越好）
*   存储效率（逻辑大小除以物理数据存储大小，存储效率也是越大越好）

HDFS中副本方案的容错性为：有N个副本，就可以容忍N-1同时发生故障。存储效率为：1/N。

下面这张表格，是针对不同的存储方案的数据容错性和存储效率。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204811.png)](https://z3.ax1x.com/2021/11/10/Ia0eWd.png)

可以看出来，XOR只能容忍一个数据块出现故障，而RS-6-3、RS-10-4允许3个、4个数据块同时出现故障。但XOR的存储效率是最高的、其次是RS-10-4、再下来是RS-6-3。而3副本的存储效率是33%。

## 4.3 连续存储还是条纹单元存储

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204814.png)](https://z3.ax1x.com/2021/11/10/Ia0nSA.png)

这张图对比了连续存储和条纹存储方案在HDFS的示意图。可以明显的看到，条纹存储方案，将一个Block继续分解为一个个的条纹单元。并在一组DataNode的block中，循环写入条纹单元。基于连续存储或者条纹存储，都是支持EC的。EC的方式存储效率比较高，但增加了复杂度、以及较高消耗的故障恢复。条纹存储方案比连续存储更好的I/O吞吐量。但与传统的MapReduce本地数据读取相悖，因为数据都是跨网络存储的。读取数据需要更多的网络I/O开销。

**连续存储**

连续存储容易实现，读写的方式与副本方式非常类似。但只有文件很大的场景适用。例如：使用RS-10-4，一个128M的文件仍然要写入4个128M的就校验块。存储开销为400%。这种方式，客户端需要有GB级别的缓存来计算奇偶校验单元。

**条纹存储**

条纹存储对小文件是友好的，可以节省很多空间。条纹单元大小通常是（64KB或者1MB）。客户端只需要有几MB的缓存就可以用于计算奇偶校验单元。但这种方案，需要跨网络I/O，性能会因此下降。要提升处理效率，需要将数据转换为连续存储，但这就需要重写整个文件了。

所以文件大小决定了使用哪种方式更合适。Cloudera做了一些调研，发现其实HDFS中小文件（少于一个EC Block Group）的使用率占整个集群的36%-97%。小文件的处理更重要。所以HDFS EC使用的是条纹存储的EC存储方案。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204818.png)](https://z3.ax1x.com/2021/11/10/Ia0uQI.png)

## 4.4 EC策略关键属性

为了适应不同的业务需求，在HDFS中可以针对文件、目录配置不同的副本和EC策略。EC策略实现了如何对文件进行编码/解码方式。每个策略包含以下属性：

*   EC schema：EC schema包含了EC Group中的EC Block数量以及奇偶校验Block的数量，例如：6+3，以及编解码器算法，例如：Reed-Solomon（索罗蒙算法）、XOR（异或算法）。
*   条纹单元大小（EC Block）：条纹单元的大小决定了条纹单元的读取、写入速度、Buffer的大小、以及编码的效率。

## 4.5 EC策略命名

EC策略命名策略：`EC编码器-EC Block数量-奇偶校验Block数量-条纹单元大小`。Hadoop中内置了5种策略：

*   RS-3-2-1024K
*   RS-6-3-1024K
*   RS-10-4-1024K
*   RS-LEGACY-6-3-1024K
*   XOR-2-1-1024K

同时，默认的副本策略也是支持的。副本策略设置在目录上，这样可以前置目录使用3副本方案，指定该目录不继承EC编码策略。这样，目录中是可以切换副本存储方式的。

## 4.6 online-EC

Replication存储方式是始终启用的，默认启用的EC策略是：RS-6-3-1024K。与Replication存储方式一样，如果父目录设置了EC策略，子文件/目录会继承父目录的EC策略。

目录级别的EC策略仅会影响在目录中创建的新文件，这也意味着就的文件不会重新进行EC编码，HDFS是使用online-EC，文件一旦创建，可以查询它的EC策略，但不能再更改

如果将已经进行EC编码的文件移动到其他EC策略的目录，文件的EC编码也不会改变。如果想要将文件转换为其他的EC策略，需要重写数据。可以通过distcp来移动数据，而不是mv。

## 4.7 自定义EC策略

HDFS允许用户基于XML自己来定义EC策略。例如：

```xml
<?xml version="1.0"?>
<configuration>

<layoutversion>1</layoutversion>
<schemas>
  
  <schema id="XORk2m1">
    
    
    <codec>xor</codec>
    <k>2</k>
    <m>1</m>
    <options> </options>
  </schema>
  <schema id="RSk12m4">
    <codec>RS</codec>
    <k>12</k>
    <m>4</m>
    <options> </options>
  </schema>
  <schema id="RS-legacyk12m4">
    <codec>RS-legacy</codec>
    <k>12</k>
    <m>4</m>
    <options> </options>
  </schema>
</schemas>
    
<policies>
  <policy>
    
    
    <schema>XORk2m1</schema>
    
    
    <cellsize>131072</cellsize>
  </policy>
  <policy>
    <schema>RS-legacyk12m4</schema>
    <cellsize>262144</cellsize>
  </policy>
</policies>
</configuration>
```

配置文件很容易理解，主要包含两个部分组成：

1.  EC Schema：编码器、k（EC Block数量）、m（奇偶校验Block数量）
2.  Policy：绑定schema、以及指定条纹单元大小

> RS-legacy：遗留的，基于纯Java语言实现的EC编解码器
> 
> 而HDFS默认的RS和XOR编解码器是基于Native实现的。

## 4.8 XOR算法与RS算法

**XOR算法**

XOR（异或）算法是最简单的EC实现，可以从任意数量的数据生成1个奇偶校验位。例如：1 ⊕ 0 ⊕ 1 ⊕ 1 = 1。但针对任意数量的条纹单元仅生成一个奇偶校验位。HDFS中如果出现多个故障，这种恢复方式是不够的。XOR的容错能力为1，存储效率为75%。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204824.png)](https://z3.ax1x.com/2021/11/10/Ia0Kyt.png)

如果某一个X、Y对丢失，可以通过奇偶检验位进行异或来恢复。

**Reed-Solomon算法**

RS算法克服了XOR算法的限制，基于线性代数运算来生成多个奇偶校验位，可以容忍多个失败。RS 算法使用生成矩阵（GT，Generator Matrix）与 m 个数据单元相乘，以获得具有 m 个数据单元（data cells）和 n 个奇偶校验单元（parity cells）的 extended codewords。RS算法的容错能力最高为n。存储效率为 m / m + n。例如：RS-6-3为67%的存储效率，而：RS-3-2为60%的存储效率。

[![](https://gitee.com/YGDBK2016/picbed_2021/raw/master/img-2021-01/20210130204826.png)](https://z3.ax1x.com/2021/11/10/Ia0MOP.png)

上图可以看到，RS是使用复杂的线性代码运算来生成多个奇偶校验单元，可以容忍每个组出现多个故障。一般生产环境都是使用RS算法。RS-k-m是将k个条纹单元与生成矩阵Gt相乘，生成具有k个条纹单元和m个奇偶校验单元。只要k + m个单元的k个可用，就可以通过剩余的条纹单元乘以Gt的倒数恢复存储失败。可以容忍m个数据单元的故障。

# 5. 部署HDFS EC


## 5.1 集群配置要求

*   EC对Hadoop集群的CPU、网络有额外的要求。EC编码、解码会消耗HDFS客户端、DataNode更多的CPU资源
*   EC要求集群中的DataNode最起码和EC条纹宽度（条纹宽度 = EC Block数量 + 奇偶校验Block数量）是一样的。也就是，如果我们用使用RS-6-3策略，至少需要9台DataNode。
*   EC Block文件也是分布在整个机架上，以实现机架级别的容错。在读写EC Block文件时，也需要保证机架的带宽。如果要实现机架级别的容错，需要拥有一定数量的机架容错。每个机架所存放的EC Block不能超过就校验块的数量。机架数量计算公式为：（EC Block数量 + 奇偶校验块数量）/ 奇偶校验块数量，然后四舍五入。例如：针对RS-6-3如果要实现机架级别的容错，至少需要（6 + 3）/ 3 = 3个机架。如果机架数小于这个数，将无法保证机架级别的容错。如果有进行机架级别停机维护需求，官方建议提供6 + 3以上个机架。

## 5.2 EC配置

默认，除了`dfs.namenode.ec.system.default.policy`指定的默认策略，其他的内置的EC策略都是禁用的。我们可以根据Hadoop集群的大小、以及所需的容错属性，通过hdfs ec -enablePolicy -policy 策略名称来启用EC策略。例如：如果有5个节点的集群，比较适合的就是RS-3-2-1024k，而RS-10-4-1024k策略就不合适了。

默认dfs.namenode.ec.system.default.policy为RS-6-3-1024k。

```properties

dfs.datanode.ec.reconstruction.stripedread.timeout.millis

dfs.datanode.ec.reconstruction.stripedread.buffer.size

dfs.datanode.ec.reconstruction.threads

dfs.datanode.ec.reconstruction.xmits.weight
```

## 5.3 EC命令

EC相关的操作，使用`hdfs ec`命令。

```null
hdfs ec [generic options]
[-setPolicy -path <path> [-policy <policyName>] [-replicate]]
[-getPolicy -path <path>]
[-unsetPolicy -path <path>]
[-listPolicies]
[-addPolicies -policyFile <file>]
[-listCodecs]
[-enablePolicy -policy <policyName>]
[-disablePolicy -policy <policyName>]
[-verifyClusterSetup -policy <policyName>...<policyName>]
[-help [cmd ...]]
```

1、查看当前HDFS支持的ec策略

```null
[root@node1 hadoop]# hdfs ec -listPolicies
Erasure Coding Policies:
ErasureCodingPolicy=[Name=RS-10-4-1024k, Schema=[ECSchema=[Codec=rs, numDataUnits=10, numParityUnits=4]], CellSize=1048576, Id=5], State=DISABLED
ErasureCodingPolicy=[Name=RS-3-2-1024k, Schema=[ECSchema=[Codec=rs, numDataUnits=3, numParityUnits=2]], CellSize=1048576, Id=2], State=DISABLED
ErasureCodingPolicy=[Name=RS-6-3-1024k, Schema=[ECSchema=[Codec=rs, numDataUnits=6, numParityUnits=3]], CellSize=1048576, Id=1], State=ENABLED
ErasureCodingPolicy=[Name=RS-LEGACY-6-3-1024k, Schema=[ECSchema=[Codec=rs-legacy, numDataUnits=6, numParityUnits=3]], CellSize=1048576, Id=3], State=DISABLED
ErasureCodingPolicy=[Name=XOR-2-1-1024k, Schema=[ECSchema=[Codec=xor, numDataUnits=2, numParityUnits=1]], CellSize=1048576, Id=4], State=ENABLED
```

我们看到目前我的HDFS集群上面启用了两个策略：一个是RS-6-3-1024k、一个是XOR-2-1-1024k。

2、查看当前HDFS支持的编解码器

```bash
[root@node1 hadoop]# hdfs ec -listCodecs
Erasure Coding Codecs: Codec [Coder List]
	RS [RS_NATIVE, RS_JAVA]
	RS-LEGACY [RS-LEGACY_JAVA]
	XOR [XOR_NATIVE, XOR_JAVA]
```

3、设置EC编码策略。因为我的测试集群只有3个节点，所以只能使用XOR-2-1-1024k。先要将XOR-2-1-1024k启用。

```bash
-- 创建用于存放冷数据的目录
[root@node1 hadoop]# hdfs dfs -mkdir -p /workspace/feng/cold_data

-- 启用XOR-2-1-1024 EC策略
[root@node1 hadoop]# hdfs ec -enablePolicy -policy XOR-2-1-1024k
Erasure coding policy XOR-2-1-1024k is enabled

-- 验证当前集群是否支持所有启用的或者指定的EC策略（这个命令应该是3.2.x添加的，我当前是3.1.4，还不支持这个命令）
-- hdfs ec -verifyClusterSetup -policy XOR-2-1-1024k

-- 设置冷数据EC存储策略
[root@node1 hadoop]# hdfs ec -setPolicy -path /workspace/feng/cold_data -policy XOR-2-1-1024k
Set XOR-2-1-1024k erasure coding policy on /workspace/feng/cold_data

-- 查看冷数据目录的存储策略
[root@node1 hadoop]# hdfs ec -getPolicy -path /workspace/feng/cold_data
XOR-2-1-1024k
```

# 6. 验证测试


## 6.1 新上传一个293M的文件到冷数据目录

```bash
[root@node1 software]# hdfs dfs -put hadoop-3.1.4.tar.gz /workspace/feng/cold_data
2021-01-16 14:23:28,681 WARN erasurecode.ErasureCodeNative: ISA-L support is not available in your platform... using builtin-java codec where applicable
```

此处，Hadoop警告提示，当前我的操作系统平台，不支持ISA-L，默认RS、XOR使用的是Native方式进行编解码，会基于Intel的ISA-L加速编解码。

我们来查看下HDFS文件的Block的信息：

```bash
[root@node3 subdir2]# hdfs fsck /workspace/feng/cold_data/hadoop-3.1.4.tar.gz -files -blocks 
```

我们看到文件是以XOR-2-1-1024k进行EC编码，并且有两个Block。总共有两个EC Block Group。

```text
0. BP-538037512-192.168.88.100-1600884040401:blk_-9223372036854775232_2020 len=268435456 Live_repl=3
1. BP-538037512-192.168.88.100-1600884040401:blk_-9223372036854775216_2021 len=38145321 Live_repl=3
```

总共的EC Block Group = 306580777字节，与原始的数据文件相等。

```bash
Erasure Coded Block Groups:
 Total size:	306580777 B
 Total files:	1
 Total block groups (validated):	2 (avg. block group size 153290388 B)
 Minimally erasure-coded block groups:	2 (100.0 %)
 Over-erasure-coded block groups:	0 (0.0 %)
 Under-erasure-coded block groups:	0 (0.0 %)
 Unsatisfactory placement block groups:	0 (0.0 %)
 Average block group size:	3.0
 Missing block groups:		0
 Corrupt block groups:		0
 Missing internal blocks:	0 (0.0 %)
FSCK ended at Sat Jan 16 16:59:46 CST 2021 in 1 milliseconds
```

原始文件大小：

```bash
[root@node1 software]# ll hadoop-3.1.4.tar.gz 
-rw-r--r-- 1 root root 306580777 Sep 25 09:29 hadoop-3.1.4.tar.gz
```

我们可以观察看到当前的Block Group大小为：256MB。而我的HDFS集群配置的dfs block size是：128MB。

```xml
<property>
    <name>dfs.blocksize</name>
    <value>134217728</value>
    <final>false</final>
    <source>hdfs-default.xml</source>
</property>
```

因为当前的Block size是128MB，而EC的策略是：XOR-2-1，也就是一个Block Group 2个Block，所以Block Group的大小就是256MB了。

## 6.2 使用distcp迁移数据

假设现在需要将一个3副本存储方式的文件，迁移到配置了EC策略的目录中。

```bash
-- 创建一个用于测试的数据目录
hdfs dfs -mkdir /workspace/feng/test_data
-- 上传一个测试文件
hdfs dfs -put hbase-logs.zip /workspace/feng/test_data

-- 启动YARN
start-yarn.sh

-- 使用distcp移动到EC策略的目录中（此处要跳过检验和，因为使用EC编码肯定校验失败）
hadoop distcp -update -skipcrccheck /workspace/feng/test_data/hbase-logs.zip /workspace/feng/cold_data
```

可以对比下该文件的block数据：

**3副本方式文件**

```null
[root@node1 hadoop]# hdfs fsck /workspace/feng/test_data/hbase-logs.zip  -files -blocks 
Connecting to namenode via http://node1:9870/fsck?ugi=root&files=1&blocks=1&path=%2Fworkspace%2Ffeng%2Ftest_data%2Fhbase-logs.zip
FSCK started by root (auth:SIMPLE) from /192.168.88.100 for path /workspace/feng/test_data/hbase-logs.zip at Sat Jan 16 19:43:39 CST 2021

/workspace/feng/test_data/hbase-logs.zip 6970734 bytes, replicated: replication=3, 1 block(s):  OK
0. BP-538037512-192.168.88.100-1600884040401:blk_1073742800_2023 len=6970734 Live_repl=3


Status: HEALTHY
 Number of data-nodes:	3
 Number of racks:		1
 Total dirs:			0
 Total symlinks:		0

Replicated Blocks:
 Total size:	6970734 B
 Total files:	1
 Total blocks (validated):	1 (avg. block size 6970734 B)
 Minimally replicated blocks:	1 (100.0 %)
 Over-replicated blocks:	0 (0.0 %)
 Under-replicated blocks:	0 (0.0 %)
 Mis-replicated blocks:		0 (0.0 %)
 Default replication factor:	3
 Average block replication:	3.0
 Missing blocks:		0
 Corrupt blocks:		0
 Missing replicas:		0 (0.0 %)

Erasure Coded Block Groups:
 Total size:	0 B
 Total files:	0
 Total block groups (validated):	0
 Minimally erasure-coded block groups:	0
 Over-erasure-coded block groups:	0
 Under-erasure-coded block groups:	0
 Unsatisfactory placement block groups:	0
 Average block group size:	0.0
 Missing block groups:		0
 Corrupt block groups:		0
 Missing internal blocks:	0
FSCK ended at Sat Jan 16 19:43:39 CST 2021 in 1 milliseconds


The filesystem under path '/workspace/feng/test_data/hbase-logs.zip' is HEALTHY

```

**EC编码后的文件**

```null
[root@node1 hadoop]# hdfs fsck /workspace/feng/cold_data/hbase-logs.zip  -files -blocks 
Connecting to namenode via http://node1:9870/fsck?ugi=root&files=1&blocks=1&path=%2Fworkspace%2Ffeng%2Fcold_data%2Fhbase-logs.zip
FSCK started by root (auth:SIMPLE) from /192.168.88.100 for path /workspace/feng/cold_data/hbase-logs.zip at Sat Jan 16 19:42:51 CST 2021

/workspace/feng/cold_data/hbase-logs.zip 6970734 bytes, erasure-coded: policy=XOR-2-1-1024k, 1 block(s):  OK
0. BP-538037512-192.168.88.100-1600884040401:blk_-9223372036854774560_2128 len=6970734 Live_repl=3

Status: HEALTHY
 Number of data-nodes:	3
 Number of racks:		1
 Total dirs:			0
 Total symlinks:		0

Replicated Blocks:
 Total size:	0 B
 Total files:	0
 Total blocks (validated):	0
 Minimally replicated blocks:	0
 Over-replicated blocks:	0
 Under-replicated blocks:	0
 Mis-replicated blocks:		0
 Default replication factor:	3
 Average block replication:	0.0
 Missing blocks:		0
 Corrupt blocks:		0
 Missing replicas:		0

Erasure Coded Block Groups:
 Total size:	6970734 B
 Total files:	1
 Total block groups (validated):	1 (avg. block group size 6970734 B)
 Minimally erasure-coded block groups:	1 (100.0 %)
 Over-erasure-coded block groups:	0 (0.0 %)
 Under-erasure-coded block groups:	0 (0.0 %)
 Unsatisfactory placement block groups:	0 (0.0 %)
 Average block group size:	3.0
 Missing block groups:		0
 Corrupt block groups:		0
 Missing internal blocks:	0 (0.0 %)
FSCK ended at Sat Jan 16 19:42:51 CST 2021 in 1 milliseconds
```

# 7. 基于Hive使用EC


基于副本冗余方式和EC方式共存。

## 7.1 按时间分区设置EC

*   对时间较早的分区（例如：半年前的数据），设置EC策略
*   默认分区还使用副本冗余方式，这样可以保证多个作业读取数据时，可以获取比较好的性能

## 7.2 按数据使用频率设置EC

*   一些只运行一两次就不再使用的数仓低层数据，可以使用EC存储。
*   一些非共享的、ETL系统的数据可以设置EC，获得更高的EC存储。