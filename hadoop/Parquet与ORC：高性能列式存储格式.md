[TOC]
参考：https://blog.csdn.net/yu616568/article/details/51868447<br>
https://blog.csdn.net/asd136912/article/details/106382032/
# 1. Parquet存储格式
Apache Parquet 最初的设计动机是存储嵌套式数据,Parquet能够透明地将Protobuf和thrift类型的数据进行列式存储,这也是Parquet相比于ORC的优势

Parquet相对于ORC，不支持update操作（数据写成后不可修改），不支持ACID等

- 基于列(在列中存储数据):用于数据存储是包含大量读取操作的优化分析工作负载
- 与Snappy的压缩压缩率高(75%)
- 只需要列将获取/读(减少磁盘I / O)
- 可以使用Avro API和Avro读写模式
- 支持谓词下推(减少磁盘I / O的成本)

## 1.1 文件结构
Parquet文件是以二进制方式存储的，是不可以直接读取和修改的，Parquet文件是自解析的，文件中包括该文件的数据和元数据。在HDFS文件系统和Parquet文件中存在如下几个概念：

- HDFS块(Block)：它是HDFS上的最小的副本单位，HDFS会把一个Block存储在本地的一个文件并且维护分散在不同的机器上的多个副本，通常情况下一个Block的大小为256M、512M等。
- HDFS文件(File)：一个HDFS的文件，包括数据和元数据，数据分散存储在多个Block中。
- 行组(Row Group)：按照行将数据物理上划分为多个单元，每一个行组包含一定的行数，在一个HDFS文件中至少存储一个行组，Parquet读写的时候会将整个行组缓存在内存中，所以如果每一个行组的大小是由内存大的小决定的。
- 列块(Column Chunk)：在一个行组中每一列保存在一个列块中，行组中的所有列连续的存储在这个行组文件中。不同的列块可能使用不同的算法进行压缩。
- 页(Page)：每一个列块划分为多个页，一个页是最小的编码的单位，在同一个列块的不同页可能使用不同的编码方式。

通常情况下，在存储Parquet数据的时候会按照HDFS的Block大小设置行组的大小，由于一般情况下每一个Mapper任务处理数据的最小单位是一个Block，这样可以把每一个行组由一个Mapper任务处理，增大任务执行并行度

![Parquet文件结构.png](https://z3.ax1x.com/2021/06/09/2cSIsg.png)

上图展示了一个Parquet文件的结构，一个文件中可以存储多个行组，文件的首位都是该文件的Magic Code，用于校验它是否是一个Parquet文件，Footer length存储了文件元数据的大小，通过该值和文件长度可以计算出元数据的偏移量，文件的元数据中包括每一个行组的元数据信息和当前文件的Schema信息。除了文件中每一个行组的元数据，每一页的开始都会存储该页的元数据，在Parquet中，有三种类型的页：数据页、字典页和索引页。数据页用于存储当前行组中该列的值，字典页存储该列值的编码字典，每一个列块中最多包含一个字典页，索引页用来存储当前行组下该列的索引，目前Parquet中还不支持索引页，但是在后面的版本中增加


# 2. Apache ORC

ORC支持update操作，支持ACID，支持struct，array复杂类型。你可以使用复杂类型构建一个类似于parquet的嵌套式数据架构，但当层数非常多时，写起来非常麻烦和复杂，而parquet提供的schema表达方式更容易表示出多级嵌套的数据类型

- 用于(在列中存储数据):用于数据存储是包含大量读取操作的优化分析工作负载
- 高压缩率(ZLIB)
- 支持Hive(datetime、小数和结构等复杂类型,列表,地图,和联盟)
- 元数据使用协议缓冲区存储,允许添加和删除字段
- HiveQL兼容
- 支持序列化

## 2.1 文件结构
和Parquet类似，ORC文件也是以二进制方式存储的，所以是不可以直接读取，ORC文件也是自解析的，它包含许多的元数据，这些元数据都是同构ProtoBuffer进行序列化的。ORC的文件结构入图6，其中涉及到如下的概念：

- ORC文件：保存在文件系统上的普通二进制文件，一个ORC文件中可以包含多个stripe，每一个stripe包含多条记录，这些记录按照列进行独立存储，对应到Parquet中的row group的概念。
- 文件级元数据：包括文件的描述信息PostScript、文件meta信息（包括整个文件的统计信息）、所有stripe的信息和文件schema信息。
- stripe：一组行形成一个stripe，每次读取文件是以行组为单位的，一般为HDFS的块大小，保存了每一列的索引和数据。
- stripe元数据：保存stripe的位置、每一个列的在该stripe的统计信息以及所有的stream类型和位置。
- row group：索引的最小单位，一个stripe中包含多个row group，默认为10000个值组成。
- stream：一个stream表示文件中一段有效的数据，包括索引和数据两类。索引stream保存每一个row group的位置和统计信息，数据stream包括多种类型的数据，具体需要哪几种是由该列类型和编码方式决定

![ORC文件结构.png](https://z3.ax1x.com/2021/06/09/2cSoLQ.png)

在ORC文件中保存了三个层级的统计信息，分别为文件级别、stripe级别和row group级别的，他们都可以用来根据Search ARGuments（谓词下推条件）判断是否可以跳过某些数据，在统计信息中都包含成员数和是否有null值，并且对于不同类型的数据设置一些特定的统计信息




**相同点**

- 基于Hadoop文件系统优化出的存储结构
- 提供高效的压缩
- 二进制存储格式
- 文件可分割，具有很强的伸缩性和并行处理能力
- 使用schema进行自我描述
- 属于线上格式，可以在Hadoop节点之间传递数据

**不同点**

- 行式存储or列式存储：Parquet和ORC都以列的形式存储数据，而Avro以基于行的格式存储数据。
- 就其本质而言，面向列的数据存储针对读取繁重的分析工作负载进行了优化，而基于行的数据库最适合于大量写入的事务性工作负载。
- 压缩率：基于列的存储区Parquet和ORC提供的压缩率高于基于行的Avro格式。

**可兼容的平台**

- ORC常用于Hive、Presto；
- Parquet常用于Impala、Drill、Spark、Arrow；
- Avro常用于Kafka、Druid。


![Parquet、Avro、ORC对比](https://pic.imgdb.cn/item/60c0891b844ef46bb29dfdca.png)