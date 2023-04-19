[TOC]

# 1. hive连续登录问题

## 1.1 需求
### 1.1.1 hql统计连续登陆的三天及以上的用户

这个问题可以扩展到很多相似的问题：连续几个月充值会员、连续天数有商品卖出、连续打车、连续逾期。

## 1.2 数据提供

```
用户ID、登入日期
user01,2018-02-28
user01,2018-03-01
user01,2018-03-02
user01,2018-03-04
user01,2018-03-05
user01,2018-03-06
user01,2018-03-07
user02,2018-03-01
user02,2018-03-02
user02,2018-03-03
user02,2018-03-06
```

## 1.3 输出字段  

```diff
+-----+-------+------------+---------+--+
| uid | times | start_date | end_date|
+-----+-------+------------+---------+--+
```

## 1.2 解决方案

这里就为大家提供一种比较常见的方案：  

### 1.2.1 建表

```sql
create table wedw_dw.t_login_info(
 user_id string COMMENT '用户ID'
,login_date date COMMENT '登录日期'
)
row format delimited
fields terminated by ',';
```

### 1.2.2 导数据

```bash
hdfs dfs -put /test/login.txt /data/hive/test/wedw/dw/t_login_info/
```

### 1.2.3 验证数据
    

```ruby
select * from wedw_dw.t_login_info;
+--------+------------+--+
| user_id| login_date |
+--------+------------+--+
| user01 | 2018-02-28 |
| user01 | 2018-03-01 |
| user01 | 2018-03-02 |
| user01 | 2018-03-04 |
| user01 | 2018-03-05 |
| user01 | 2018-03-06 |
| user01 | 2018-03-07 |
| user02 | 2018-03-01 |
| user02 | 2018-03-02 |
| user02 | 2018-03-03 |
| user02 | 2018-03-06 |
+--------+------------+--+
```

### 1.2.4 解决sql

```sql
select
 t2.user_id         as user_id
,count(1)           as times
,min(t2.login_date) as start_date
,max(t2.login_date) as end_date
from
(
    select
     t1.user_id
    ,t1.login_date
    ,date_sub(t1.login_date,rn) as date_diff
    from
    (
        select
         user_id
        ,login_date
        ,row_number() over(partition by user_id order by login_date asc) as rn 
        from
        wedw_dw.t_login_info
    ) t1
) t2
group by 
 t2.user_id
,t2.date_diff
having times >= 3
;
```

### 1.2.5 结果

```ruby
+---------+-----+------------+------------+--+
| user_id |times| start_date |  end_date  |
+---------+-----+------------+------------+--+
| user01  |  3  | 2018-02-28 | 2018-03-02 |
| user01  |  4  | 2018-03-04 | 2018-03-07 |
| user02  |  3  | 2018-03-01 | 2018-03-03 |
+---------+-----+------------+------------+--+
```
## 1.3 思路

### 1.3.1 先把数据按照用户id分组，根据登录日期排序

```sql
select
         user_id
        ,login_date
        ,row_number() over(partition by user_id order by login_date asc) as rn 
        from
        wedw_dw.t_login_info
```

```ruby
+---------+------------+---+--+
| user_id | login_date | rn|
+---------+------------+---+--+
| user01  | 2018-02-28 | 1 |
| user01  | 2018-03-01 | 2 |
| user01  | 2018-03-02 | 3 |
| user01  | 2018-03-04 | 4 |
| user01  | 2018-03-05 | 5 |
| user01  | 2018-03-06 | 6 |
| user01  | 2018-03-07 | 7 |
| user02  | 2018-03-01 | 1 |
| user02  | 2018-03-02 | 2 |
| user02  | 2018-03-03 | 3 |
| user02  | 2018-03-06 | 4 |
+---------+------------+---+--+
```

### 1.3.2 获取连续登陆日期
用登录日期减去排序数字rn，得到的差值日期如果是相等的，则说明这两天肯定是连续的

```sql
select
     t1.user_id
    ,t1.login_date
    ,date_sub(t1.login_date,rn) as date_diff
    from
    (
        select
         user_id
        ,login_date
        ,row_number() over(partition by user_id order by login_date asc) as rn 
        from
        wedw_dw.t_login_info
    ) t1
    ;
```

```ruby
+--------+------------+-----------+--+
| user_id| login_date | date_diff |
+--------+------------+-----------+--+
| user01 | 2018-02-28 | 2018-02-27|
| user01 | 2018-03-01 | 2018-02-27|
| user01 | 2018-03-02 | 2018-02-27|
| user01 | 2018-03-04 | 2018-02-28|
| user01 | 2018-03-05 | 2018-02-28|
| user01 | 2018-03-06 | 2018-02-28|
| user01 | 2018-03-07 | 2018-02-28|
| user02 | 2018-03-01 | 2018-02-28|
| user02 | 2018-03-02 | 2018-02-28|
| user02 | 2018-03-03 | 2018-02-28|
| user02 | 2018-03-06 | 2018-03-02|
+--------+------------+-----------+--+
```

### 1.3.3 统计最早最晚登陆时间
根据user\_id和日期差date\_diff 分组，最小登录日期即为此次连续登录的开始日期start\_date，最大登录日期即为结束日期end\_date，登录次数即为分组后的count(1)

```sql
select
 t2.user_id         as user_id
,count(1)           as times
,min(t2.login_date) as start_date
,max(t2.login_date) as end_date
from
(
    select
     t1.user_id
    ,t1.login_date
    ,date_sub(t1.login_date,rn) as date_diff
    from
    (
        select
         user_id
        ,login_date
        ,row_number() over(partition by user_id order by login_date asc) as rn 
        from
        wedw_dw.t_login_info
    ) t1
) t2
group by 
 t2.user_id
,t2.date_diff
having times >= 3
;
```

```ruby
+---------+-----+------------+-----------+--+
| user_id |times| start_date |  end_date |
+---------+-----+------------+-----------+--+
| user01  |  3  | 2018-02-28 | 2018-03-02|
| user01  |  4  | 2018-03-04 | 2018-03-07|
| user02  |  3  | 2018-03-01 | 2018-03-03|
+---------+-----+------------+-----------+--+
```


# 2. 前后数据对比

## 2.1 top数据求与下一名的差值

以下表记录了用户每天的充值流水记录。
```sql
table_name：user_topup

流水号     用户    日期  充值金额（元）
seq(key) user_id data_dt topup_amt

xxxxx01 u_001 2017/1/1 10
xxxxx02 u_001 2017/1/2 150
xxxxx03 u_001 2017/1/2 110
xxxxx04 u_001 2017/1/2 10
xxxxx05 u_001 2017/1/4 50
xxxxx06 u_001 2017/1/4 10
xxxxx07 u_001 2017/1/6 45
xxxxx08 u_001 2017/1/6 90
xxxxx09 u_002 2017/1/1 10
xxxxx10 u_002 2017/1/2 150
xxxxx11 u_002 2017/1/2 70
xxxxx12 u_002 2017/1/3 30
xxxxx13 u_002 2017/1/3 80
xxxxx14 u_002 2017/1/4 150
xxxxx14 u_002 2017/1/5 101
xxxxx15 u_002 2017/1/6 68
xxxxx16 u_002 2017/1/6 120
```

统计累计充值金额最多的top10用户，并计算每个用户比他后一名多充值多少钱。

答案
```sql
select 
    user_id, 
    amts,
    amts - lead(amts,1,0) over(sort by amts desc) 
from 
(select
    user_id,
    sum(topup_amt) as amts
from
    user_topup
group by user_id
order by amts desc
limit 10
) t1;
```

**补充**
## 2.2 Lag和Lead分析函数
Lag和Lead分析函数可以在同一次查询中取出同一字段的前N行的数据(Lag)和后N行的数据(Lead)作为独立的列。

**LAG**

```bash
LAG(col,n,DEFAULT) 用于统计窗口内往上第n行值
参数1为列名，参数2为往上第n行（可选，默认为1），参数3为默认值（当往上第n行为NULL时候，取默认值，如不指定，则为NULL）
```


**LEAD**

```bash
与LAG相反
LEAD(col,n,DEFAULT) 用于统计窗口内往下第n行值
参数1为列名，参数2为往下第n行（可选，默认为1），参数3为默认值（当往下第n行为NULL时候，取默认值，如不指定，则为NULL）
```

# 3. 美团面试题
参考：https://www.it610.com/article/1281627168186580992.htm

## 3.1 数据

Linux本地目录，存在文件test.txt，内容如下：<br>
`ymd` `user_id` `age` `view_time` `register_time`
```
20200401,0001,25,15,2020-04-01 13:01:23
20200401,0002,33,24,2020-04-01 15:06:36
20200402,0001,25,35,2020-04-01 13:01:23
20200403,0004,38,4,2020-04-03 21:00:01
20200404,0001,25,9,2020-04-01 13:01:23
20200402,0002,33,33,2020-04-01 15:06:36
20200403,0002,33,17,2020-04-01 15:06:36
20200402,0003,45,47,2020-04-02 11:00:01
20200404,0005,55,27,2020-04-04 21:00:01
20200405,0005,55,24,2020-04-04 21:00:01
```

**建表**

```sql
create table t1(ymd bigint,user_id string,age int,view_time int,register_time timestamp) row format delimited fields terminated by ',' stored as textfile ;
```


**本地导入数据**

```sql
load data local inpath '/home/hive/test.txt' into table t1;
```

## 3.2 问题

问题如下

- 统计：用户总量，用户平均年龄
- 统计：每天注册的用户数,和用户累计观看时长，用户平均观看时长
- 统计：每10岁一个分段，统计每个区间的用户总量，用户平均观看时长
- 统计：每天观看时长最长的用户userid
- 统计：用户次日留存情况
- 统计：每天观看时长都大于20min的用户总量，只要有一天观看时长小于20min就不算

### 3.2.1 统计：用户总量，用户平均年龄

这是一个汇总统计, 最后只需要输出一条信息,所以最外层不使用group by, 直接使用聚合函数即可

相应的, 内层需要得到用户, 年龄两个属性。 所以在内层使用分组聚合

```sql
select count(user_id), avg(age) from (
select user_id,age from t1 group by user_id,age
) t2;
```

### 3.2.2 统计：每天注册的用户数,和用户累计观看时长，用户平均观看时长

每天注册的用户数
```sql
select register_date,count(user_id) from (
select to_date(register_time) register_date ,user_id from t1 group by to_date(register_time),user_id
) t2
group by register_date;
```

> **补充：**<br>
> to\_date：日期时间转日期函数
> 
> ```
> select to_date('2015-04-02 13:34:12');
> 输出：2015-04-02
> ```

用户累计观看时长，用户平均观看时长
```sql
select user_id,round(avg(view_time),2),sum(view_time) from (
select user_id,view_time from t1 group by user_id, view_time
) t2 
group by user_id;
```

> **补充：**
> ```
> round(num,2)  四舍五入,保留两位小数
> round(3.1415,2) = 3.14
> 
> floor 向下取整
> floor(3.14) = 3
> 
> ceil 向上取整
> floor(3.14) = 4
> ```

### 3.2.3 统计：每10岁一个分段，统计每个区间的用户总量，用户平均观看时长

```sql
-- 手动标记很不方便, 学习大佬的思路,内层自动标记年龄段
select user_id
, floor(age / 10)*10 flag
, sum(view_time) sum
from t1
group by user_id, age
;

-- 汇总, 完成统计
select flag,count(user_id),round(avg(sum),2) from (
select floor(age/10)*10 flag,user_id,sum(view_time) sum from t1 group by user_id, age
) t2 
group by flag;
```

### 3.2.4 统计：每天观看时长最长的用户userid

```sql
select ymd,user_id,view_time,rank from (
select ymd,user_id,view_time,row_number() over(partition by ymd order by view_time desc) rank from t1
) t2
where rank=1;
```

### 3.2.5 统计：用户次日留存情况

### 3.2.6 统计：每天观看时长都大于20min的用户总量，只要有一天观看时长小于20min就不算

这是典型的not in逻辑
常见的解决思路是`left join` + `is null`

```sql

-- 寻找时长小于20的用户
-- 内层不要去重, 不然会产生大量job
select user_id
from t1
where view_time <= 20
;

-- 左半连接找null(job 1)

select t1.user_id 
from t1 
left join(
select user_id 
from t1 
where view_time <= 20
) t2 
on t1.user_id=t2.user_id 
where t2.user_id is null 
group by t1.user_id;
```


# 4. 14天到访地问题

## 4.1 数据

Linux本地目录，存在文件access.txt，内容如下：<br>
`user_id` `login_date` `city`
```sql
user01,2021-03-01,北京
user01,2021-03-02,北京
user01,2021-03-04,三亚
user01,2021-03-05,广州
user01,2021-03-06,广州
user01,2021-03-07,北京
user01,2021-03-10,天津
user01,2021-03-13,北京
user01,2021-03-16,北京
user01,2021-03-22,北京
user02,2021-03-01,北京
user02,2021-03-02,北京
user02,2021-03-03,济南
user02,2021-03-06,北京
user02,2021-03-07,北京
user02,2021-03-09,北京
user02,2021-03-11,北京
user02,2021-03-14,北京
user02,2021-03-16,上海
user02,2021-03-22,上海
user02,2021-03-23,北京
user02,2021-03-24,北京
user02,2021-03-25,上海
user02,2021-03-26,北京
user02,2021-03-27,北京
user03,2021-03-11,北京
user03,2021-03-12,三亚
user03,2021-03-14,北京
user03,2021-03-15,北京
user03,2021-03-16,西安
user03,2021-03-17,西安
user03,2021-03-20,北京
user03,2021-03-21,上海
user03,2021-03-22,上海
user03,2021-03-23,上海
```

**建表**

```sql
create table access_t(
 user_id string COMMENT '用户ID'
,login_date date COMMENT '登录日期'
,city string COMMENT '城市'
)
row format delimited
fields terminated by ',';
```


**本地导入数据**

```sql
load data local inpath '/home/hive/access.txt' into table access_t;
```

## 4.2 问题

问题如下：

求用户以最晚时间为界限，求14天内去过的城市

## 4.3 答案

方法1：
```sql
WITH t1 AS(
        SELECT user_id, login_date, city, MAX(login_date) OVER(
            PARTITION BY user_id) AS last_date FROM access_t
    ),
    t2 AS(
        SELECT user_id, login_date, city, datediff(last_date,
            login_date) AS diff FROM t1
    ),
    t3 AS(
        SELECT user_id, city FROM t2 WHERE diff < 14 GROUP BY user_id,
        city
    )
SELECT user_id, CONCAT_WS(',', collect_list(city)) AS citys
FROM t3
GROUP BY user_id;
```


方法2

```sql
WITH t1 AS(
        SELECT user_id, login_date, city, MAX(login_date) OVER(
            PARTITION BY user_id) AS last_date FROM access_t
    ),
    t2 AS(
        SELECT user_id, login_date, city, datediff(last_date,
            login_date) AS diff FROM t1
    ),
    t3 AS(
        SELECT user_id, city FROM t2 WHERE diff < 14
    )
SELECT user_id, CONCAT_WS(',', collect_set(city)) AS citys
FROM t3
GROUP BY user_id;
```

> 补充：**with as**<br>
> 当我们书写一些结构相对复杂的SQL语句时，可能某个子查询在多个层级多个地方存在重复使用的情况，这个时候我们可以使用 with as 语句将其独立出来，极大提高SQL可读性，简化SQL
> - with as 最后必须跟sql语句结束，不允许单独使用
> - 前面的with子句定义的查询在后面的with子句中可以使用。但是一个with子句内部不能嵌套with子句
> - with as 是子查询部分，并不是真正的临时表,查询结果保存在内存中。定义一个sql片段，该片段在整个sql都可以被利用，可以提高代码的可读性以及减少重复读，减少性能成本

### 4.3.1 求出每个用户的最晚访问时间
`MAX(login_date) OVER (PARTITION BY user_id )` 分组求每组最大值

按用户id分组，求出每个用户最晚访问时间
```sql
  SELECT
     user_id,
     login_date,
     city,
     MAX(login_date) OVER(partition by user_id) as last_date
  FROM
      access_t 
```

结果

```sql
user_id|login_date| city | last_date
-------|----------|------|------------
user01	2021-03-01	北京	2021-03-22
user01	2021-03-02	北京	2021-03-22
user01	2021-03-04	三亚	2021-03-22
user01	2021-03-05	广州	2021-03-22
user01	2021-03-06	广州	2021-03-22
user01	2021-03-07	北京	2021-03-22
user01	2021-03-10	天津	2021-03-22
user01	2021-03-13	北京	2021-03-22
user01	2021-03-16	北京	2021-03-22
user01	2021-03-22	北京	2021-03-22
user02	2021-03-23	北京	2021-03-27
user02	2021-03-24	北京	2021-03-27
user02	2021-03-25	上海	2021-03-27
user02	2021-03-26	北京	2021-03-27
user02	2021-03-27	北京	2021-03-27
user02	2021-03-14	北京	2021-03-27
user02	2021-03-02	北京	2021-03-27
user02	2021-03-03	济南	2021-03-27
user02	2021-03-06	北京	2021-03-27
user02	2021-03-07	北京	2021-03-27
user02	2021-03-09	北京	2021-03-27
user02	2021-03-11	北京	2021-03-27
user02	2021-03-01	北京	2021-03-27
user02	2021-03-16	上海	2021-03-27
user02	2021-03-22	上海	2021-03-27
user03	2021-03-23	上海	2021-03-23
user03	2021-03-11	北京	2021-03-23
user03	2021-03-12	三亚	2021-03-23
user03	2021-03-14	北京	2021-03-23
user03	2021-03-15	北京	2021-03-23
user03	2021-03-16	西安	2021-03-23
user03	2021-03-17	西安	2021-03-23
user03	2021-03-20	北京	2021-03-23
user03	2021-03-21	上海	2021-03-23
user03	2021-03-22	上海	2021-03-23
Time taken: 3.495 seconds, Fetched: 35 row(s)
```

### 4.3.2 求出每个用户的最晚的14天访问记录

求出每个时间与最晚时间的时间差

```sql
WITH t1 as (
  SELECT
     user_id,
     login_date,
     city,
     MAX(login_date) OVER(partition by user_id) as last_date
  FROM
      access_t 
)
  SELECT
     user_id,
     login_date,
     city,
     datediff(last_date,login_date) as diff
  FROM
     t1；
```

结果
```sql
user_id|login_date| city | diff
-------|----------|------|------------
user01	2021-03-01	北京	21
user01	2021-03-02	北京	20
user01	2021-03-04	三亚	18
user01	2021-03-05	广州	17
user01	2021-03-06	广州	16
user01	2021-03-07	北京	15
user01	2021-03-10	天津	12
user01	2021-03-13	北京	9
user01	2021-03-16	北京	6
user01	2021-03-22	北京	0
user02	2021-03-23	北京	4
user02	2021-03-24	北京	3
user02	2021-03-25	上海	2
user02	2021-03-26	北京	1
user02	2021-03-27	北京	0
user02	2021-03-14	北京	13
user02	2021-03-02	北京	25
user02	2021-03-03	济南	24
user02	2021-03-06	北京	21
user02	2021-03-07	北京	20
user02	2021-03-09	北京	18
user02	2021-03-11	北京	16
user02	2021-03-01	北京	26
user02	2021-03-16	上海	11
user02	2021-03-22	上海	5
user03	2021-03-23	上海	0
user03	2021-03-11	北京	12
user03	2021-03-12	三亚	11
user03	2021-03-14	北京	9
user03	2021-03-15	北京	8
user03	2021-03-16	西安	7
user03	2021-03-17	西安	6
user03	2021-03-20	北京	3
user03	2021-03-21	上海	2
user03	2021-03-22	上海	1
Time taken: 3.92 seconds, Fetched: 35 row(s)
```

求出最近14天的数据


```sql
WITH t1 as (
  SELECT
     user_id,
     login_date,
     city,
     MAX(login_date) OVER(partition by user_id) as last_date
  FROM
      access_t 
),
t2 AS(
  SELECT
     user_id,
     login_date,
     city,
     datediff(last_date,login_date) as diff
  FROM
     t1
)
  SELECT
     user_id,
     login_date,
     city 
  FROM
     t2 
WHERE
  diff < 14
```


```sql
user_id|login_date| city
-------|----------|------
user01	2021-03-10	天津
user01	2021-03-13	北京
user01	2021-03-16	北京
user01	2021-03-22	北京
user02	2021-03-23	北京
user02	2021-03-24	北京
user02	2021-03-25	上海
user02	2021-03-26	北京
user02	2021-03-27	北京
user02	2021-03-14	北京
user02	2021-03-16	上海
user02	2021-03-22	上海
user03	2021-03-23	上海
user03	2021-03-11	北京
user03	2021-03-12	三亚
user03	2021-03-14	北京
user03	2021-03-15	北京
user03	2021-03-16	西安
user03	2021-03-17	西安
user03	2021-03-20	北京
user03	2021-03-21	上海
user03	2021-03-22	上海
Time taken: 1.2 seconds, Fetched: 22 row(s)

```

利用gropu by去重
```sql
WITH t1 as (
  SELECT
     user_id,
     login_date,
     city,
     MAX(login_date) OVER(partition by user_id) as last_date
  FROM
      access_t 
),
t2 AS(
  SELECT
     user_id,
     login_date,
     city,
     datediff(last_date,login_date) as diff
  FROM
     t1
)
  SELECT
     user_id,
     city 
  FROM
     t2 
WHERE
  diff < 14
GROUP BY user_id,city
```


```sql
user_id| city
-------|-----
user01	北京
user01	天津
user02	上海
user02	北京
user03	三亚
user03	上海
user03	北京
user03	西安
Time taken: 4.175 seconds, Fetched: 8 row(s)
```

### 4.3.3 `CONCAT_WS`列拼接


```sql
WITH t1 as (
  SELECT
     user_id,
     login_date,
     city,
     MAX(login_date) OVER(partition by user_id) as last_date
  FROM
      access_t 
),
t2 AS(
  SELECT
     user_id,
     login_date,
     city,
     datediff(last_date,login_date) as diff
  FROM
     t1
),
t3 AS(
  SELECT
     user_id,
     city 
  FROM
     t2 
WHERE
  diff < 14
GROUP BY user_id,city
)
SELECT user_id,
   CONCAT_WS(',',collect_list(city)) as citys 
FROM t3
GROUP BY user_id
;
```


```sql
user_id| citys
-------|-----
user01	北京,天津
user02	上海,北京
user03	三亚,上海,北京,西安
Time taken: 4.602 seconds, Fetched: 3 row(s)
```

> 补充：
> 
> 参考：https://www.cnblogs.com/cc11001100/p/9043946.html
>
> `concat_ws(seperator, string s1, string s2...)`<br>
> - 功能：制定分隔符将多个字符串连接起来，实现“列转行”<br>
> - 例子：常常结合group by与collect_set使用
> - collect_set 只能返回不重复的集合,
若要返回带重复的要用collect_list
> 
> ```sql
> select user_id,
> concat_ws(',',collect_list(order_id)) as order_value 
> from col_lie
> group by user_id
> limit 10;
> ```
> 
>  **collect_set 和 concat_ws 一起用，实现字段元素去重**
> 
> 
> ```sql
> select a, b, concat_ws(',' , collect_set(c)) as c_set
> from table group by a,b;
> ```
