[TOC]
# 1. hive建库建表与数据导入
## 1.1 建库
hive中有一个默认的库：

库名： default

库目录：`hdfs://hdp20-01:9000/user/hive/warehouse`

新建库：

```sql
create database db_order;
```

库建好后，在hdfs中会生成一个库目录：

```bash
hdfs://hdp20-01:9000/user/hive/warehouse/db_order.db
```

## 1.2 建表
### 1.2.1 基本建表语句


```sql
use db_order;

create table t_order(id string,create_time string,amount float,uid string);
```


表建好后，会在所属的库目录中生成一个表目录

`/user/hive/warehouse/db_order.db/t_order`

只是，这样建表的话，hive会认为表数据文件中的字段分隔符为 ^A

 

正确的建表语句为：

```sql
create table t_order(id string,create_time string,amount float,uid string)

row format delimited

fields terminated by ',';
```

这样就指定了，我们的表数据文件中的字段分隔符为 ","

### 1.2.2 删除表

```sql
drop table t_order;
```

删除表的效果是：

1. hive会从元数据库中清除关于这个表的信息；
2. hive还会从hdfs中删除这个表的表目录；

### 1.2.3 内部表与外部表
- 内部表(MANAGED_TABLE)：表目录按照hive的规范来部署，位于hive的仓库目录/user/hive/warehouse中

- 外部表(EXTERNAL_TABLE)：表目录由建表用户自己指定


```sql
create external table t_access(ip string,url string,access_time string)

row format delimited

fields terminated by ','

location '/access/log';
```


**外部表和内部表的特性差别**：

1. 内部表的目录在hive的仓库目录中 VS 外部表的目录由用户指定

2. drop一个内部表时：hive会清除相关元数据，并删除表数据目录。drop一个外部表时：hive只会清除相关元数据；

3. 内部表数据存储的位置是`hive.metastore.warehouse.dir`（默认：`/user/hive/warehouse`），外部表数据的存储位置由自己制定（如果没有LOCATION，Hive将在HDFS 上的/user/hive/warehouse 文件夹下以外部表的表名创建一个文件夹，并将数据这个表的数据存放在这里）

4. 对内部表的修改会直接同步给元数据，而对外部表的表结构和分区进行修改，在需要修复（mask repair table tablename）

一个hive的数据仓库，最底层的表，一定是来自于外部系统，为了不影响外部系统的工作逻辑，在hive中可建external表来映射这些外部系统产生的数据目录；

然后，后续的etl操作，产生的各种表建议用managed_table

### 1.2.4 分区表

分区表的实质是：在表目录中为数据文件创建分区子目录，以便于在查询时，MR程序可以针对分区子目录中的数据进行处理，缩减读取数据的范围。

比如，网站每天产生的浏览记录，浏览记录应该建一个表来存放，但是，有时候，我们可能只需要对某一天的浏览记录进行分析

<font style="background-color: #FFF0AC
;">这时，就可以将这个表建为分区表，每天的数据导入其中的一个分区；</font>

当然，每日的分区目录，应该有一个目录名（分区字段）

#### 1.2.4.1 一个分区字段的实例：

示例如下：

**1、创建带分区的表**

```sql
create table t_access(ip string,url string,access_time string)

partitioned by(dt string)

row format delimited

fields terminated by ',';
```

注意：分区字段不能是表定义中的已存在字段


**2、向分区中导入数据**


```sql
load data local inpath '/root/access.log.2017-08-04.log' into table t_access partition(dt='20170804');

load data local inpath '/root/access.log.2017-08-05.log' into table t_access partition(dt='20170805');
```


**3、针对分区数据进行查询**

1) 统计8月4号的总PV：

```sql
select count(*) from t_access where dt='20170804';
```

实质：就是将分区字段当成表字段来用，就可以使用where子句指定分区了

2) 统计表中所有数据总的PV：

```sql
select count(*) from t_access;
```

实质：不指定分区条件即可


#### 1.2.4.2 多个分区字段示例

建表：

```sql
create table t_partition(id int,name string,age int) 
partitioned by(department string,sex string,howold int)
row format delimited fields terminated by ',';
```

导数据：


```sql
load data local inpath '/root/p1.dat' into table t_partition partition(department='xiangsheng',sex='male',howold=20);
```

### 1.2.5 CTAS建表语法

**1、可以通过已存在表来建表**：

```sql
create table t_user_2 like t_user;
```

新建的t_user_2表结构定义与源表t_user一致，但是没有数据

**2、在建表的同时插入数据**

```sql
create table t_access_user
as
select ip,url from t_access;
```

t_access_user会根据select查询的字段来建表，同时将查询的结果插入新表中

## 1.3 数据导入导出
### 1.3.1 将数据文件导入hive的表

**方式1：导入数据的一种方式：**

手动用hdfs命令，将文件放入表目录；

**方式2：在hive的交互式shell中用hive命令来导入本地数据到表目录**


```sql
hive>load data local inpath '/root/order.data.2' into table t_order;
```

**方式3：用hive命令导入hdfs中的数据文件到表目录**

```sql
hive>load data inpath '/access.log.2017-08-06.log' into table t_access partition(dt='20170806');
```

注意： <font style="background-color: #FFFF00
;">**导本地文件和导HDFS文件的区别**</font>：

本地文件导入表：复制

hdfs文件导入表：<font color="red">**移动**</font>

### 1.3.2 将hive表中的数据导出到指定路径的文件

**1、将hive表中的数据导入HDFS的文件**

```sql
insert overwrite directory '/root/access-data'
row format delimited fields terminated by ','
select * from t_access;
```

**2、将hive表中的数据导入本地磁盘文件**

```sql
insert overwrite local directory '/root/access-data'
row format delimited fields terminated by ','
select * from t_access limit 100000;
```

## 1.4 数据类型
### 1.4.1.数字类型

**TINYINT** (1字节整数)

**SMALLINT** (2字节整数)

**INT/INTEGER** (4字节整数)

**BIGINT** (8字节整数)

**FLOAT** (4字节浮点数)

**DOUBLE** (8字节双精度浮点数)

示例：

```sql
create table t_test(a string ,b int,c bigint,d float,e double,f tinyint,g smallint)
```

### 1.4.2 日期时间类型

**TIMESTAMP** (时间戳)

**DATE** (日期)

示例，假如有以下数据文件：

```bash
1,zhangsan,1985-06-30
2,lisi,1986-07-10
3,wangwu,1985-08-09
```

那么，就可以建一个表来对数据进行映射

```sql
create table t_customer(id int,name string,birthday date) row format delimited fields terminated by ',';
```

然后导入数据

```sql
load data local inpath '/root/customer.dat' into table t_customer;
```

然后，就可以正确查询

### 1.4.3 字符串类型

**STRING**

**VARCHAR** (字符串1-65355长度，超长截断)

**CHAR** (字符串，最大长度255)

 

### 1.4.4 其他类型

**BOOLEAN**（布尔类型）：true  false

**BINARY** (二进制)：

 

### 1.4.5 复合类型

#### 1.4.5.1 array数组类型

arrays: **ARRAY**<data_type> )

示例：array类型的应用

假如有如下数据需要用hive的表去映射：

```bash
战狼2,吴京:吴刚:龙母,2017-08-16
三生三世十里桃花,刘亦菲:痒痒,2017-08-20
```

设想：如果主演信息用一个数组来映射比较方便


建表：

```sql
create table t_movie(moive_name string,actors array<string>,first_show date)
row format delimited fields terminated by ','
collection items terminated by ':';
```

导入数据：

```sql
load data local inpath '/root/movie.dat' into table t_movie;
```

查询：

```sql
select * from t_movie;

select moive_name,actors[0] from t_movie;

select moive_name,actors from t_movie where array_contains(actors,'吴刚');

select moive_name,size(actors) from t_movie;
```


#### 1.1.5.2 map类型

maps: **MAP**<primitive_type, data_type> 

1) 假如有以下数据：

```bash
1,zhangsan,father:xiaoming#mother:xiaohuang#brother:xiaoxu,28
2,lisi,father:mayun#mother:huangyi#brother:guanyu,22
3,wangwu,father:wangjianlin#mother:ruhua#sister:jingtian,29
4,mayun,father:mayongzhen#mother:angelababy,26
```

可以用一个map类型来对上述数据中的家庭成员进行描述

2) 建表语句：

```sql
create table t_person(id int,name string,family_members map<string,string>,age int)
row format delimited fields terminated by ','
collection items terminated by '#'
map keys terminated by ':';
```

3) 查询

```sql
select * from t_person;
```

- 取map字段的指定key的值

```sql
select id,name,family_members['father'] as father from t_person;
```
- 取map字段的所有key

```sql
select id,name,map_keys(family_members) as relation from t_person;
```
- 取map字段的所有value

```sql
select id,name,map_values(family_members) from t_person;

select id,name,map_values(family_members)[0] from t_person;
```

- 综合：查询有brother的用户信息

```sql
select id,name,father
from
(select id,name,family_members['brother'] as father from t_person) tmp
where father is not null;
```

#### 1.1.5.3 struct类型

structs: **STRUCT**<col_name : data_type, ...>

1) 假如有如下数据：

```bash
1,zhangsan,18:male:beijing
2,lisi,28:female:shanghai
```
其中的用户信息包含：年龄：整数，性别：字符串，地址：字符串

设想用一个字段来描述整个用户信息，可以采用struct

2) 建表：


```sql
create table t_person_struct(id int,name string,info struct<age:int,sex:string,addr:string>)
row format delimited fields terminated by ','
collection items terminated by ':';
```

3) 查询

```sql
select * from t_person_struct;

select id,name,info.age from t_person_struct;
```

## 1.2 修改表定义
仅修改Hive元数据，不会触动表中的数据，用户需要确定实际的数据布局符合元数据的定义。

### 1.2.1 修改表名

```sql
ALTER TABLE table_name RENAME TO new_table_name
```

示例：alter table t_1 rename to t_x;
 

### 1.2.2 修改分区名


```sql
alter table t_partition partition(department='xiangsheng',sex='male',howold=20) rename to partition(department='1',sex='1',howold=20);
```

### 1.2.3 添加分区

```sql
alter table t_partition add partition (department='2',sex='0',howold=40);
```

### 1.2.4 删除分区


```sql
alter table t_partition drop partition (department='2',sex='2',howold=24);
```

### 1.2.5 修改表的文件格式定义

```sql
ALTER TABLE table_name [PARTITION partitionSpec] SET FILEFORMAT file_format
```
 
例子：
```sql
alter table t_partition partition(department='2',sex='0',howold=40 ) set fileformat sequencefile;
```

### 1.2.6 修改列名定义

```sql
ALTER TABLE table_name CHANGE [COLUMN] col_old_name col_new_name column_type [COMMENTcol_comment] [FIRST|(AFTER column_name)]
```

例子：
```sql
alter table t_user change price jiage float first;
```

### 1.2.7 增加/替换列

```sql
ALTER TABLE table_name ADD|REPLACE COLUMNS (col_name data_type[COMMENT col_comment], ...)
```

例子：
```sql
alter table t_user add columns (sex string,addr string);

alter table t_user replace columns (id string,age int,price float);
```


# 2. hive查询语法
提示：在做小数据量查询测试时，可以让hive将mrjob提交给本地运行器运行，可以在hive会话中设置如下参数：

```sql
hive> set hive.exec.mode.local.auto=true;
```

## 2.1 基本查询示例

```sql
select * from t_access;

select count(*) from t_access;

select max(ip) from t_access;
```


## 2.2 条件查询

```sql
select * from t_access where access_time<'2017-08-06 15:30:20'

select * from t_access where access_time<'2017-08-06 16:30:20' and ip>'192.168.33.3';
```

## 2.3 join关联查询示例
假如有a.txt文件

```bash
a,1
b,2
c,3
d,4
```
假如有b.txt文件

```bash
a,xx
b,yy
d,zz
e,pp
```
进行各种join查询：

### 2.3.1 inner join（join）


```
select
a.name as aname,
a.numb as anumb,
b.name as bname,
b.nick as bnick
from t_a a
join t_b b
on a.name=b.name
```

结果：

```bash
+--------+--------+--------+--------+--+
| aname  | anumb  | bname  | bnick  |
+--------+--------+--------+--------+--+
|   a    |   1    |   a    |   xx   |
|   b    |   2    |   b    |   yy   |
|   d    |   4    |   d    |   zz   |
+--------+--------+--------+--------+--+
```

### 2.3.2 left outer join（left join）

```sql
select
a.name as aname,
a.numb as anumb,
b.name as bname,
b.nick as bnick
from t_a a
left outer join t_b b
on a.name=b.name
```
结果：

```bash
+--------+--------+--------+--------+--+
| aname  | anumb  | bname  | bnick  |
+--------+--------+--------+--------+--+
|   a    |   1    |   a    |   xx   |
|   b    |   2    |   b    |   yy   |
|   c    |   3    |  NULL  |  NULL  |
|   d    |   4    |   d    |   zz   |
+--------+--------+--------+--------+--+
```

### 2.3.3 right outer join（right join）


```sql
select
a.name as aname,
a.numb as anumb,
b.name as bname,
b.nick as bnick
from t_a a
right outer join t_b b
on a.name=b.name
```

结果：

```bash
+--------+--------+--------+--------+--+
| aname  | anumb  | bname  | bnick  |
+--------+--------+--------+--------+--+
|   a    |   1    |   a    |   xx   |
|   b    |   2    |   b    |   yy   |
|   d    |   4    |   d    |   zz   |
|  NULL  |  NULL  |   e    |   pp   |
+--------+--------+--------+--------+--+
```

### 2.3.4 full outer join（full join）

```sql
select
a.name as aname,
a.numb as anumb,
b.name as bname,
b.nick as bnick
from t_a a
full join t_b b
on a.name=b.name;
```

结果：

```bash
+--------+--------+--------+--------+--+
| aname  | anumb  | bname  | bnick  |
+--------+--------+--------+--------+--+
|   a    |   1    |   a    |   xx   |
|   b    |   2    |   b    |   yy   |
|   c    |   3    |  NULL  |  NULL  |
|   d    |   4    |   d    |   zz   |
|  NULL  |  NULL  |   e    |   pp   |
+--------+--------+--------+--------+--+
```

## 2.4 left semi join
<font style="background-color: #FFFF00
;">Left semi join ：相当于join连接两个表后产生的数据中的左半部分</font>

hive中不支持exist/IN子查询，可以用left semi join来实现同样的效果：

```sql
select
a.name as aname,
a.numb as anumb
from t_a a
left semi join t_b b
on a.name=b.name;
```

结果：
```bash
+--------+--------+--+
| aname  | anumb  |
+--------+--------+--+
|   a    |   1    |
|   b    |   2    |
|   d    |   4    |
+--------+--------+--+
```

注意：<font style="background-color:#FF69B4
;">left semi join的 **select**子句中，不能有右表的字段</font>

## 2.5 group by分组聚合

```bash
20170804,192.168.33.66,http://www.edu360.cn/job
20180804,192.168.33.40,http://www.edu360.cn/study
20180805,192.168.20.18,http://www.edu36.cn/job
20180805,192.168.20.28,http://www.edu36.cn/login
20180806,192.168.20.38,http://www.edu36.cn/job
20180806,192.168.20.38,http://www.edu36.cn/study
20180807,192.168.33.40,http://www.edu36.cn/login
20180807,192.168.20.88,http://www.edu36.cn/job
```

```sql
select dt,count(*),max(ip) as cnt from t_access group by dt;
```

```sql
select dt,count(*),max(ip) as cnt from t_access group by dt having dt>'20170804';
```

```sql
select
dt,count(*),max(ip) as cnt
from t_access
where url='http://www.edu360.cn/job'
group by dt having dt>'20170804';
```

注意：<font style="background-color:#FF69B4
;">一旦有group by子句，那么，在**select子句**中就不能有 （分组字段，聚合函数）以外的字段</font>

 

> **为什么where必须写在group by的前面，为什么group by后面的条件只能用having**
> 
> 因为，where是用于在真正执行查询逻辑之前过滤数据用的
> 
> having是对group by聚合之后的结果进行再过滤；

 
上述语句的执行逻辑：

<font style="background-color:red
;">1、where过滤不满足条件的数据</font>

<font style="background-color:red
;">2、用聚合函数和group by进行数据运算聚合，得到聚合结果</font>

<font style="background-color:red
;">3、用having条件过滤掉聚合结果中不满足条件的数据</font>

## 2.6 子查询

```bash
1,zhangsan,father:xiaoming#mother:xiaohuang#brother:xiaoxu,28
2,lisi,father:mayun#mother:huangyi#brother:guanyu,22
3,wangwu,father:wangjianlin#mother:ruhua#sister:jingtian,29
4,mayun,father:mayongzhen#mother:angelababy,26
```

-- 查询有兄弟的人


```sql
select id,name,brother
from
(select id,name,family_members['brother'] as brother from t_person) tmp
where brother is not null;
```

另一种写法：

```sql
select id,name,family_members[‘brother’]
from t_person where array_contains(map_keys(family_members),”brother”);
```

# 3 hive函数使用
小技巧：测试函数的用法，可以专门准备一个专门的dual表

~~create table dual(x string);~~

~~insert into table dual values('');~~

其实：<font style="color:  	#A52A2A;">直接用常量来测试函数即可</font>


```sql
select substr("abcdefg",1,3);
```

hive的所有函数手册：

https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF#LanguageManualUDF-Built-inTable-GeneratingFunctions(UDTF)

## 3.1  常用内置函数
### 3.1.1 类型转换函数

```sql
select cast("5" as int) ;

select cast("2017-08-03" as date) ;

select cast(current_timestamp as date);
```

示例：

```bash
1,1995-05-05 13:30:59,1200.3
2,1994-04-05 13:30:59,2200
3,1996-06-01 12:20:30,80000.5
```

```bash
create table t_fun(id string,birthday string,salary string)
row format delimited fields terminated by ',';
```

```sql
select id,cast(birthday as date) as bir,cast(salary as float) from t_fun;
```

### 3.1.2 数学运算函数

```sql
select round(5.4);   ## 5  四舍五入

select round(5.1345,3) ;  ##5.135

select ceil(5.4) ; // select ceiling(5.4) from dual;   ## 6  向上取整

select floor(5.4);  ## 5  向下取整

select abs(-5.4) ;  ## 5.4绝对值

select greatest(3,5,6) ;  ## 6 

select least(3,5,6) from dual;  ##求多个输入参数中的最小值
```

示例：

有表如下：

```bash
0:jdbc:hive2://hadoop01:10000> select * from t_fun2;

+-----------+-------------+------------------+---------------+---------------+--+
| t_fun2.id | t_fun2.name |  t_fun2.salay1   | t_fun2.salay2 | t_fun2.salay3 |
+-----------+-------------+------------------+---------------+---------------+--+
|     1     |   zhangsa   |1000.7999877929688|    2000.0     |    1800.0     |
|     2     |   lisi      |9000.7998046875   |    6000.0     |    9800.0     |
+-----------+-------------+------------------+---------------+---------------+--+
```


```sql
select greatest(cast(s1 as double),cast(s2 as double),cast(s3 as double)) from t_fun2;
```
结果：

```bash
+--------+--+
|  _c0   |
+--------+--+
| 2000.0 |
| 9800.0 |
+--------+--+
```

```sql
select max(age) from t_person;    聚合函数
select min(age) from t_person;    聚合函数
```

### 3.1.3 字符串函数

#### 3.1.3.1 substr(string str, int start)  截取子串

substring(string str, int start)

示例：

```sql
select substr("abcdefg",2) from dual;
```


**substr(string, int start, int len)**

substring(string, int start, int len)

示例：

```sql
select substr("abcdefg",2,3) from dual;
```


 

#### 3.1.3.2 concat(string A, string B...)  拼接字符串

concat_ws(string SEP, string A, string B...)

示例：

```sql
select concat("ab","xy") from dual;  ## abxy

select concat_ws(".","192","168","33","44") from dual; ## 192.168.33.44
```

#### 3.1.3.3 length(string A)

示例：

```sql
select length("192.168.33.44") from dual;  ## 13
```

#### 3.1.3.3 split(string str, string pat)

示例：<font style="color: red;">~~select split("192.168.33.44",".") from dual;~~</font> 错误的，因为 **`.号`是正则语法中的特定字符**


```sql
select split("192.168.33.44","\\.") from dual;
```


 
#### 3.1.3.4 upper(转大写) and lower(转小写)

```sql
upper(string str) ##转大写

lower(string str)
```


 

### 3.1.4 时间函数

```sql
select current_timestamp; ## 获取当前的时间戳(详细时间信息)

select current_date;   ## 获取当前的日期
```

#### 3.1.4.1 取当前时间的秒数时间戳
(距离格林威治时间1970-1-1 0:0:0秒 的差距)

```sql
select unix_timestamp();
```

#### 3.1.4.2 unix时间戳转字符串

```sql
from_unixtime(bigint unixtime[, string format])
```


示例：

```sql
select from_unixtime(unix_timestamp());

select from_unixtime(unix_timestamp(),"yyyy/MM/dd HH:mm:ss");
```


 

#### 3.1.4.3 字符串转unix时间戳

```sql
unix_timestamp(string date, string pattern)
```

示例：
```sql
select unix_timestamp("2017-08-10 17:50:30");

select unix_timestamp("2017-08-10 17:50:30","yyyy-MM-dd HH:mm:ss");
```

#### 3.1.4.4 将字符串转成日期date


```sql
select to_date("2017-09-17 16:58:32");
```


 

 

### 3.1.5 条件控制函数

#### 3.1.5.1 case when

语法：

```
CASE[ expression ]
WHEN condition1 THEN result1
WHEN condition2 THEN result2
...
WHEN conditionn THEN resultn
ELSE result
END
```

示例：

```sql
select id,name,
case
when age<28 then 'youngth'
when age>27 and age<40 then 'zhongnian'
else 'old'
end
from t_user;
```

#### 3.1.5.2 IF

```sql
select id,if(age>25,'working','worked') from t_user;

select moive_name,if(array_contains(actors,'吴刚'),'好电影',’烂片儿’) from t_movie;
```

### 3.1.6 集合函数

#### 3.1.6.1 array_contains(Array<T>, value)
返回boolean值

示例：

```sql
select moive_name,array_contains(actors,'吴刚') from t_movie;
select array_contains(array('a','b','c'),'c') from dual;
```

#### 3.1.6.2 sort_array(Array<T>)
返回排序后的数组

示例：

```sql
select sort_array(array('c','b','a')) from dual;
select 'haha',sort_array(array('c','b','a')) as xx from (select 0) tmp;
```

#### 3.1.6.3 size(Array<T>)
返回一个集合的长度，int值

示例：

```sql
select moive_name,size(actors) as actor_number from t_movie;
```

#### 3.1.6.4 size(Map<K.V>)
返回一个int值

```sql
map_keys(Map<K.V>)  返回一个map字段的所有key，结果类型为：数组

map_values(Map<K.V>) 返回一个map字段的所有value，结果类型为：数组
```

### 3.1.7 常见聚合函数

- **sum**

- **avg**

- **max**

- **min**

- **count**

### 3.1.8 表生成函数

#### 3.1.8.1 行转列函数：explode()

假如有以下数据：


```bash
1,zhangsan,化学:物理:数学:语文
2,lisi,化学:数学:生物:生理:卫生
3,wangwu,化学:语文:英语:体育:生物
```

映射成一张表：

```sql
create table t_stu_subject(id int,name string,subjects array<string>)
row format delimited fields terminated by ','
collection items terminated by ':';
```

使用explode()对数组字段“炸裂”

[![explode](https://pic.downk.cc/item/5e21224a2fb38b8c3c2c6acb.jpg)](https://s2.ax1x.com/2020/01/17/lzpWM8.jpg)

然后，我们利用这个explode的结果，来求去重的课程：

```sql
select distinct tmp.sub
from
(select explode(subjects) as sub from t_stu_subject) tmp;
```

#### 3.1.8.2 表生成函数lateral view


```sql
select id,name,tmp.sub
from t_stu_subject lateral view explode(subjects) tmp as sub;
```

[![lateral view](https://pic.downk.cc/item/5e21224a2fb38b8c3c2c6acd.jpg)](https://s2.ax1x.com/2020/01/17/lzp2xf.jpg)

<font style="background-color:#FF69B4
;">理解：lateral view 相当于两个表在join<br/></font>
<font style="background-color:#FF69B4
;">左表：是原表<br/></font>
<font style="background-color:#FF69B4
;">右表：是explode(某个集合字段)之后产生的表<br/></font>
<font style="background-color:#FF69B4
;">而且：这个join只在同一行的数据间进行</font>


那样，可以方便做更多的查询：

比如，查询选修了生物课的同学


```sql
select a.id,a.name,a.sub from
(select id,name,tmp.sub as sub from t_stu_subject lateral view explode(subjects) tmp as sub) a
where sub='生物';
```
