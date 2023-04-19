[TOC]
# 一.hive常用函数

### 1.NVL(expr1,expr2)
如果该函数的第一个参数为空那么显示第二个参数的值，如果第一个参数的值不为空，则显示第一个参数本来的值。适用于数字型、字符型和日期型，但是 expr1和expr2的数据类型必须为同类型
eg：代码：
```sql
select NVL(null,0)  --0
```
### 2.concat 和 concat_ws
concat(str1,SEP,str2,SEP,str3,……) 和concat_ws(SEP,str1,str2,str3, ……)
字符串连接函数,需要是 string型字段。

eg:代码:
```sql
select concat('江苏省','-','南京市','-','玄武区','-','徐庄软件园'); --江苏省-南京市-玄武区-徐庄软件园
select concat_ws('-','江苏省','南京市','玄武区','徐庄软件园');  --江苏省-南京市-玄武区-徐庄软件园
```
结论：当连接的内容(字段)多于2个的时候，concat_ws的优势就显现了，写法简单、方便

###  3.unix_timestamp() 当前系统时间
  unix_timestamp() 是将当前系统时间转换成数字型秒数，from_unixtime 将数字型按照 格式进行时间转换。
eg：代码：
```sql
select unix_timestamp()  --1576060579
select from_unixtime(unix_timestamp(),'yyyy-MM-dd HH:mm:ss')  --2019-12-11 17:26:36
```
### 4.regexp_replace
regexp_replace(string A, string B, string C)
字符串替换函数，将字符串A 中的B 用 C 替换。
eg：代码：
 ```sql
select regexp_replace('www.tuniu.com','tuniu','jd');  --www.jd.com
```

### 5.repeat
repeat(string str, int n) 重复N次字符串
eg：代码：
```sql
select repeat('ab',3);  --ababab
```

### 6.lpad 和 rpad
#### 6.1. lpad(string str, int len, string pad)
将字符串str 用pad进行左补足 到len位(如果位数不足的话)
#### 6.2. rpad(string str, int len, string pad)
将字符串str 用pad进行右补足 到len位(如果位数不足的话)
eg：代码:
 ```sql
select lpad(3204,5,0) --03204
select rpad(3204,5,0) --32040
```
### 7.trim(string A)
trim(string A) 删除字符串两边的空格，中间的会保留,相应的 ltrim(string A) ,rtrim(string A)
### 8.cast(string as date) 
将string类型转换成 date   ,CAST进行显式的类型转换，例如CAST('1' as INT),如果转换失败，CAST返回NULL
### 9.if
if(boolean testCondition, T valueTrue, T valueFalseOrNull) ，根据条件返回不同的值
eg：代码：
```sql
select if(length('hello')=5,'真','假')  --真
select if(length('hello')=5,'真','假')  --假
```
### 10.greatest(T v1, T v2, ...) 
返回最大值，会过滤null
eg:代码：
```sql
select greatest('2016-01-01',NULL,'2017-01-01');  --2017-01-01
```
### 11.least(T v1, T v2, ...)
返回最小值，会过滤null
eg:代码:
```sql
select least('2016-01-01',NULL,'2017-01-01','2015-01-01'); --2015-01-01
```
### 12.rand()
返回0-1的随机值。rand(INT seed) 返回固定的随机值。
eg:代码:
```sql
select rand()  --0.9385521919794135
```

# 二、hive日期函数
### 1.to_date(string timestamp)
将时间戳转换成日期型字符串
eg：代码：
```sql
select to_date('2017-01-16 09:55:54');  --2017-01-16
```
### 2.date_format
(date/timestamp/string ts, string fmt) 按照格式返回字符串
eg：代码：
```sql
select date_format('2017-01-16 09:55:54', 'yyyy-MM-dd')  --2017-01-16
```
### 3.datediff(string enddate, string startdate)
返回int 的两个日期差
eg：代码：
```sql
select datediff('2017-01-16', '2017-01-10');  --6
```
### 4.date_add(string startdate, int days)
日期加减
eg：代码：
```sql
select date_add('2017-01-10', 7);
```
### 5.current_timestamp 和 current_date
返回当前时间戳，当前日期
eg：代码：
 ```sql
select current_timestamp;  --2019-12-11 17:40:52.829
select current_date; --2019-12-11 00:00:00.000
```
### 6.date_format(date/timestamp/string ts, string fmt) 
按照格式返回字符串
eg：代码：
```sql
select date_format('2017-01-16 09:55:54', 'yyyy-MM-dd');  --2017-01-16
```
### 7.last_day(string date) 
返回当前时间的月末日期
eg：代码：
```sql
select last_day('2017-01-16');  --2017-01-31
```
### 8.regexp_replace 日期格式转化
```sql
select regexp_replace('${v_date}','-','' ) --20191201
```
### 9.date_sub(string startdate, int days) 日期加减
```sql
select date_sub('2018-05-04','30')  --2018-04-04 00:00:00.000
select date_format(date_sub('2018-05-04','30'),'yyyy-MM-dd')   --2018-04-04
```


# 三.特殊的聚合函数
## 1.COUNT_BIG
返回指定组中的项目数量，与COUNT函数不同的是COUNT_BIG返回bigint值，而COUNT返回的是int值。 例：select count_big(prd_no) from sales
## 2.GROUPING 
产生一个附加的列，当用CUBE或ROLLUP运算符添加行时，输出值为1.当所添加的行不是由CUBE或ROLLUP产生时，输出值为0。例：select prd_no,sum(qty),grouping(prd_no) from sales group by prd_no with rollup
# 四.hive中的高阶函数
##  1.row_number、rank、dense_rank
  row_number()从1开始，按照顺序，生成分组内记录的序列,row_number()的值不会存在重复,当排序的值相同时,按照表中记录的顺序进行排列
RANK() 生成数据项在分组中的排名，排名相等会在名次中留下空位
DENSE_RANK() 生成数据项在分组中的排名，排名相等会在名次中不会留下空位

### NTILE
用于将分组数据按照顺序切分成n片，返回当前记录所在的切片值
### sum_over
1、hive中查询一组中的前几名，就用到dense_rank(),rank(),row_number()这几个函数，他们的区别在于
 rank（）就是排序 相同的排序是一样的，但是下一个小的会跳着排序，比如 
等级 排序
23 1
23 1
22 3
dense_rank()相同的排序相同，下一个小的会紧挨着排序，比如
等级 排序
23 1
23 1
22 2
这样总个数是相对减少的，适合求某些指标前几个等级的个数。
row_number()就很简单，顺序排序。比如
等级 排序
23 1
23 2
22 3
这种排序 总个数是不变的，适合求某些值的前几名。