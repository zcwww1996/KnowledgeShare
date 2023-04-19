[TOC]
<font style="background-color: Cyan;">***使用场景：在技术对app进行埋点时，会讲多个字段存放在一个数组中，因此模型调用数据时，要对埋点数据进行解析，以作进一步的清洗***</font>

# 解析json字符串的两个函数
解析json字符串的两个函数：**get_json_object**和**json_tuple**

建表：
```sql
create table proj_gucp_dw_beta1.json_table(name string,json_fields string)
insert into proj_gucp_dw_beta1.json_table  values ('xiaoli','{"source":"7fresh","monthSales":4900,"userCount":1900,"score":"9.9"}')
```
结果如下：
[![QKJ6Nd.png](https://s2.ax1x.com/2019/12/03/QKJ6Nd.png)](https://s2.ax1x.com/2019/12/03/QKJ6Nd.png)

## 1. get_json_object
**函数的作用：用来解析json字符串的一个字段**：
```sql
select select get_json_object(json_fields,'$.source') as source,
       get_json_object(json_fields,'$.monthSales') as monthSales,
       get_json_object(json_fields,'$.userCount') as userCount  
from proj_gucp_dw_beta1.json_table 
```
运行结果如下：
[![QKJDBD.png](https://s2.ax1x.com/2019/12/03/QKJDBD.png)](https://s2.ax1x.com/2019/12/03/QKJDBD.png)
## 2. json_tuple
**函数的作用：用来解析json字符串中的多个字段**
```sql
select 
       b.source,
       b.monthSales,
       b.userCount
from proj_gucp_dw_beta1.json_table  a 
lateral view json_tuple('{"source":"7fresh","monthSales":4900,"userCount":1900,"score":"9.9"}','source','monthSales','userCount') b  as source,monthSales,userCount
```
运行结果如下：
[![QKJrHe.png](https://s2.ax1x.com/2019/12/03/QKJrHe.png)](https://s2.ax1x.com/2019/12/03/QKJrHe.png)
