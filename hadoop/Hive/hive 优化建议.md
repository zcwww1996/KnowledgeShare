[TOC]
# 1. Hive SQL优化方案
## 1.1 所有的数据采用压缩+小文件合并的方式写表

**原因:大多数的hive任务不是CPU密集型任务,压缩和解压缩不是系统瓶颈压缩和解压缩会提高HDFS使用效率**

```sql
-- 小文件合并&文件压缩
set mapred.max.split.size=256000000;
set hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat ;
set hive.merge.mapfiles = true ;
set hive.merge.mapredfiles= true ;
set hive.merge.size.per.task = 256000000 ;
set hive.merge.smallfiles.avgsize=256000000 ;
set hive.exec.compress.output=true;
set mapreduce.output.fileoutputformat.compress=true ;
set mapreduce.output.fileoutputformat.compress.type=BLOCK ;
set mapreduce.output.fileoutputformat.compress.codec=org.apache.hadoop.io.compress.GzipCodec ;
set hive.exec.dynamic.partition.mode=nonstrict;
set hive.hadoop.supports.splittable.combineinputformat=true;
```

## 1.2 insert overwrite 全部改成 insert into

**原因:insert overwrite 是一个线程插数据,insert into 是多个线程插数据**

```sql
alter table xxx drop if exists partition(xx=xx);
insert into table xxx select * from xxx
或者
drop table xxx;
create table xxxx select * from xxx
```


## 1.3 distinct xx 语句全部改为 group by xx 
 
**原因:数据量大使用group by效果优于distinct**


## 1.4 将子查询改为with临时表的方式

**原因:with临时表多表join的时候只需要计算一次,而且对Spark SQL Server Driver端生成代码并且编译比较友好,不容易卡死**


## 1.5 小表写在前面,大表在后面,小表较小时(<25M)可以使用Map JOIN


## 1.6 添加数据倾斜相关配置
对于数据出现数据倾斜相关的 sql应该特殊处理,比如,进行采样判断出现量比较大的key,可以按照逻辑进行过来或者哈希join

```sql
-- 数据倾斜
set hive.map.aggr=true ;
set hive.groupby.skewindata=true;
set hive.groupby.mapaggr.checkinterval = 100000;
set hive.map.aggr.hash.min.reduction=0.5;
```

## 1.7 对于标签数据无其他业务逻辑
全部只取T+2当天的数据,全量数据进行计算应小于10分钟

## 1.8 两表JOIN如果是一大一小表,应该将小表压入内存中

```sql
set hive.auto.convert.join = true;
-- 小表的最大文件大小，默认为25000000，即25M
set hive.mapjoin.smalltable.filesize = 25000000;
-- 是否将多个mapjoin合并为一个
set hive.auto.convert.join.noconditionaltask = true;
-- 多个mapjoin转换为1个时，所有小表的文件大小总和的最大值。
set hive.auto.convert.join.noconditionaltask.size = 10000000;

INSERT
INTO TABLE xxx PARTITION (l_date = '${v_date}')
select /*+MAPJOIN(temp2)*/ xx
from temp2 b
         inner join temp a on a.xx = b.xx;
```

# 2. 逻辑优化方案

## 2.1 优先使用hive
如果SQL使用Hive可以跑通的,不要使用spark sql

## 2.2 加解密操作
在进行加解密操作是不要去使用`get_index`函数,应该使用`proj_gucp_dw.encryption_index`或者 `proj_gucp_dw_tst.encryption_index`

## 2.3 stts标签表的时间
stts标签表的时间必须优先取表中符合业务逻辑的字段