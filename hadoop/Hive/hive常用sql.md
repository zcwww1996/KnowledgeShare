[TOC]
# 一、hive分区

## 1.查看分区
```sql
show partitions table_name;
```

## 2.删除分区
```sql
ALTER TABLE table_name  DROP IF EXISTS PARTITION (day_id='20180509');
```
## 3.重命名分区
```sql
ALTER TABLE my_partition_test_table partition(l_date='2019-10-07') rename to partition(l_date='2019-10-10');
```
## 4.增加分区
```sql
alter table table_name if not exists add partition(month_id='201805',day_id='20180509') location '/user/tuoming/part/201805/20180509';
```
## 5.恢复分区
```sql
msck repair table table_name
```
# 二、hive 表的增删改查
## （一）查
### 1.查看数据库/表
```sql
show databases/tables;
```
### 2.查看表结构
```sql
desc table_name;
```
### 3.查看表详细属性
```sql
desc formatted tablename;
```
### 4.模糊搜索表
```sql
 show tables like ‘_name_’
```
## （二）改
### 1.修改表
#### 1.重命名表
```sql
ALTER TABLE table_name RENAME TO new_table_name
```
### 2.修改字段
#### 1.修改字段注释
```sql
alter table tablename change column original_union_id 列名 字段类型 COMMENT '列注释’;
```
#### 2.重命名字段
```sql
ALTER TABLE tablename CHANGE 原列名 修改后列名 字段类型  COMMENT '列注释' AFTER col3
```
```sql
CREATE TABLE test_change (a int, b int, c int);
ALTER TABLE test_change CHANGE a a1 INT; //将a列的名称改为a1
ALTER TABLE test_change CHANGE a a1 STRING AFTER b; //将a列的名称改为a1，将a的数据类型改为string，并将其放在b列之后。新表的结构是:b int, a1 string, c int
ALTER TABLE test_change CHANGE b b1 INT FIRST; ////将b列的名称改为b1，并将其作为第一列。新表的结构是:b1 int, a string, c int
```
#### 3.修改表属性
```sql
alter table table_name set TBLPROPERTIES ('EXTERNAL'='TRUE');  //内部表转外部表
alter table table_name set TBLPROPERTIES ('EXTERNAL'='FALSE');  //外部表转内部表
```
## （三）增
### 1.增加表注释
```sql
ALTER TABLE 表名 SET TBLPROPERTIES ('comment' = '注释内容')
```
eg:
ALTER TABLE test SET TBLPROPERTIES ('comment' = '财务月结数据表')

### 2.新增字段
```sql
ALTER TABLE table_name ADD COLUMNS (col_name STRING);  //在所有存在的列后面，但是在分区列之前添加一列
```
### 3.hive 创建表
```sql
create table proj_gucp_dw_beta1.temp1_app_20191213(
column1 string,
column2 string,
column3 string)
PARTITIONED BY(l_date string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\u0005'
COLLECTION ITEMS TERMINATED BY '\u0002'
MAP KEYS TERMINATED BY '\u0003'
STORED AS  TEXTFILE;
```
## （四）删
### 1.删除字段
```sql
ALTER TABLE test REPLACE COLUMNS(id BIGINT, name STRING)

```
### 2. 删除表
#### (1).清空表
```sql
truncate table table_name;  //清空表不能删除外部表！因为外部表里的数据并不是存放在Hive Meta store中
```
TRUNCATE:truncate用于删除所有的行，这个行为在[Hive](http://lib.csdn.net/base/hive)元存储删除数据是不可逆的
#### (2).删除表
```sql
DROP table table_name；
```
Hive不支持行级插入操作、更新操作和删除操作。
eg1. 
```sql
delete from tmp.pid_free_2_paid_ana_time where pre_free_pack_id = 104 //报错
```


