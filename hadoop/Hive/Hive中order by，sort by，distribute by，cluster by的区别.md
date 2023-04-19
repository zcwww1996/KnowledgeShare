[TOC]

# 1. 排序函数
## 1.1 order by

**order by会对输入做全局排序**，因此只有一个Reducer(多个Reducer无法保证全局有序)，然而只有一个Reducer，会导致当输入规模较大时，消耗较长的计算时间。关于order by的详细介绍请参考这篇文章：[Hive Order by操作](https://blog.csdn.net/lzm1340458776/article/details/43230517)。


## 1.2 sort by

**sort by不是全局排序，其在数据进入reducer前完成排序(组内排序)**，因此，如果用sort by进行排序，并且设置mapred.reduce.tasks>1，则sort by只会保证每个reducer的输出有序，并不保证全局有序。sort by不同于order by，它不受hive.mapred.mode属性的影响，sort by的数据只能保证在同一个reduce中的数据可以按指定字段排序。使用sort by你可以指定执行的reduce个数(通过set mapred.reduce.tasks=n来指定)，对输出的数据再执行归并排序，即可得到全部结果。


## 1.3 distribute by

distribute by是控制在map端如何拆分数据给reduce端的。hive会根据distribute by后面列，对应reduce的个数进行分发，默认是采用hash算法，含有相同字段值的记录会进入同一个reduce中，但是每个reduce中的数据并不是有序的。

sort by为每个reduce产生一个排序文件。在有些情况下，你需要控制某个特定行应该到哪个reducer，这通常是为了进行后续的聚集操作。distribute by刚好可以做这件事。因此，distribute by经常和sort by配合使用。

注：Distribute by和sort by的使用场景

1. Map输出的文件大小不均。
2. Reduce输出文件大小不均。
3. 小文件过多。
4. 文件超大。


## 1.4 cluster by

**cluster by**除了具有distribute by的功能外还兼具sort by的功能。但是排序**只能是倒叙排序**，不能指定排序规则为ASC或者DESC。



# 2. 示例

## 2.1 sort by


```sql
hive (hive)> select * from user;
    OK
    id	name
    1	lavimer
    2	liaozhongmin
    3	liaozemin
```


使用sort by按id降序排列：


```sql
hive (hive)> select * from user sort by id desc;
    //MapReduce...
    Execution completed successfully
    Mapred Local Task Succeeded . Convert the Join into MapJoin
    OK
    id	name
    3	liaozemin
    2	liaozhongmin
    1	lavimer
    Time taken: 3.828 seconds
```


## 2.2 distribute by


```sql
hive (hive)> select * from user;
    OK
    id	name
    1	lavimer
    2	liaozhongmin
    3	liaozemin
    100	hello
    200	hadoop
```

设置reduce的个数


```sql
hive (hive)> set mapred.reduce.tasks=2;
    hive (hive)> set mapred.reduce.tasks;  
    mapred.reduce.tasks=2
```



使用带distribute by的数据从user表中导出数据


```sql
hive (hive)> insert overwrite local directory '/usr/local/src/user.txt' select * from user distribute by id;
    //MapReduce...
    Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 2
```


注：从上述语句执行过程可以看到启动了两个Reducer。

导出到本地的数据


```bash
[root@liaozhongmin5 src]# cd user.txt/
    [root@liaozhongmin5 user.txt]# ll
    总用量 8
    -rwxrwxrwx. 1 root root 36 1月  30 14:35 000000_0
    -rwxrwxrwx. 1 root root 22 1月  30 14:35 000001_0
    [root@liaozhongmin5 user.txt]# more 000000_0 
    2<span style="white-space:pre">	</span>liaozhongmin
    100<span style="white-space:pre">	</span>hello
    200<span style="white-space:pre">	</span>hadoop
    [root@liaozhongmin5 user.txt]# more 000001_0 
    1<span style="white-space:pre">	</span>lavimer
    3<span style="white-space:pre">	</span>liaozemin
    [root@liaozhongmin5 user.txt]#
```


注：从上述结果中，我们可以看到数据被分发到了两个Reducer中处理。


## 2.3 distribute by和sort by结合使用


```sql
hive (hive)> select * from temperature;
    OK
    year	tempra
    2008	30`C
    2008	35`C
    2008	32.5`C
    2008	31.5`C
    2008	31`C
    2015	41`C
    2015	39`C
    2015	36`C
    2015	33`C
    2015	35`C
    2015	37`C
```


根据年份和气温对气象数据进行排序，以确保所具有相同年份的行最终都在一个reduce分区中。


```sql
hive (hive)> select * from temperature distribute by year sort by year asc,tempra desc;
    //MapReduce...
    Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 2
    //MapReduce...
    OK
    year	tempra
    2008	35`C
    2008	32.5`C
    2008	31`C
    2008	31.5`C
    2008	30`C
    2015	41`C
    2015	39`C
    2015	37`C
    2015	36`C
    2015	35`C
    2015	33`C
    Time taken: 17.358 seconds
```
