[TOC]

# 一. 表连接优化

## 1.  将大表放后头
Hive假定查询中最后的一个表是大表。它会将其它表缓存起来，然后扫描最后那个表。

因此通常需要将小表放前面，或者标记哪张表是大表：/*streamtable(table_name) */

## 2. 使用相同的连接键
当对3个或者更多个表进行join连接时，如果每个on子句都使用相同的连接键的话，那么只会产生一个MapReduce job。
## 3. 尽量尽早地过滤数据

减少每个阶段的数据量,对于分区表要加分区，同时只选择需要使用到的字段。

## 4. 尽量原子化操作

尽量避免一个SQL包含复杂逻辑，可以使用中间表来完成复杂的逻辑
# 二. 用insert into替换union all

如果union all的部分个数大于2，或者每个union部分数据量大，应该拆成多个insert into 语句，实际测试过程中，执行时间能提升50%
# 三.  order by & sort by 

order by : 对查询结果进行全局排序，消耗时间长。需要 set hive.mapred.mode=nostrict

sort by : 局部排序，并非全局有序，提高效率。
# 四. 调整mapper和reducer的个数

## 1 Map阶段优化

map个数的主要的决定因素有： input的文件总个数，input的文件大小，集群设置的文件块大小（默认128M，不可自定义）。
## 2 Reduce阶段优化

调整方式：
-- set mapred.reduce.tasks=?
-- set hive.exec.reducers.bytes.per.reducer = ?

一般根据输入文件的总大小,用它的estimation函数来自动计算reduce的个数：reduce个数 = InputFileSize / bytes per reducer
# 五. 数据倾斜
1. group by造成的
参数可以解决：
```java
hive.map.aggr=true
```
2. join造成的
可以使用skew join，其原理是把特殊值先不在Reduce端计算掉，而是先写入HDFS，然后启动一轮Map join专门做这个特殊值的计算。需要设置set Hive.optimize.skewjoin=true; 然后根据Hive.skewjoin.key的值来判断超过多少条算特殊值。这种情况可以在SQL语句里将特殊值隔离开来避免数据倾斜
# 六. Map与Reduce之间的优化（Spill、Copy、Sort phase）


1. 在Spill(溢出)阶段，由于内存不够，数据需要溢写到磁盘再排序，然后对所有的文件进行Merge。可以通过设置io.Sort.mb来增大环形缓冲区的大小，避免Spill

2. 在Copy阶段是把文件从map端Copy到Reduce端。默认情况下是5%的Map完成时Reduce就开始启动Copy，这个时候是很浪费资源的，因为Reduce一旦启动就被占用，直到所有的Map完成，Reduce才可以进行接下来的动作。这个比例可以通过Mapred.Reduce.slowstart.completed.Maps这个参数来设置。
# 七. JVM重用

正常情况下MapReduce启动JVM完成一个task后就退出了，如果任务花费时间短，但又要多次启动JVM的情况下，这种情况下可以通过设置Mapred.Job.reuse.jvm.num.tasks=5; 可以让JVM运行多次任务之后再退出