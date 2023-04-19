[TOC]
### 一：建表语句：
```sql
drop table explode_lateral_view;
create table explode_lateral_view
(`area` string,
`goods_id` string,
`sale_info` string)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '|'
STORED AS textfile;
```
###  二：导入数据：
> a:shandong,b:beijing,c:hebei|1,2,3,4,5,6,7,8,9|[{"source":"7fresh","monthSales":4900,"userCount":1900,"score":"9.9"},{"source":"jd","monthSales":2090,"userCount":78981,"score":"9.8"},{"source":"jdmart","monthSales":6987,"userCount":1600,"score":"9.0"}]

表内数据如下

[![](https://c-ssl.duitang.com/uploads/item/201907/22/20190722233902_yQthz.png)](https://c-ssl.duitang.com/uploads/item/201907/22/20190722233902_yQthz.png)
### 三：explode函数的使用：
#### 1.拆解<font color="red">array字段</font>，语句为

```sql
select explode(split(goods_id,',')) as goods_id from explode_lateral_view;
```

结果如下:

[![](https://hbimg.huabanimg.com/499e8121f11efc69e3207bff6963473b817ecd3e22ab-yCBhNw_fw658)](https://hbimg.huabanimg.com/499e8121f11efc69e3207bff6963473b817ecd3e22ab-yCBhNw_fw658)

#### 2.拆解<font color="red">map字段</font>，语句为

```sql
select explode(split(area,',')) as area from explode_lateral_view;
```


我们会得到如下结果：

![](https://hbimg.huabanimg.com/5ba2d9c287ccaef827beab08d4fdf669bda8c5098be2-UafEHa_fw658)
#### 3.拆解<font color="red">json字段</font>, 配get_json_object
我们想获取所有的monthSales，第一步我们先把这个字段拆成list，并且拆成行展示：
```sql
select explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{')) as  sale_info from explode_lateral_view;
```
[![](https://hbimg.huabanimg.com/eb3d95bf16a0674fec9144aa9f41b08bf16438cf2007d-V1o0H2_fw658)](https://hbimg.huabanimg.com/eb3d95bf16a0674fec9144aa9f41b08bf16438cf2007d-V1o0H2_fw658)

然后我们想用get_json_object来获取key为monthSales的数据：
```sql
select get_json_object(explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{')),'$.monthSales') as  sale_info from explode_lateral_view;
```
<font color="red">然后挂了FAILED: SemanticException [Error 10081]: UDTF's are not supported outside the SELECT clause, nor nested in expressions</font>

<font style="background-color: yellow;">UDTF explode不能写在别的函数景</font>

如果你这么写，想查两个字段，select explode(split(area,',')) as area,good_id from explode_lateral_view;

<font color="red">会报错FAILED: SemanticException 1:40 Only a single expression in the SELECT clause is supported with UDTF's. Error encountered near token 'good_id'</font>

<font style="background-color: yellow;">使用UDTF的时候，只支持一个字段</font>
，这时候就需要LATERAL VIEW出场了

### 四：explode函数的使用：
LATERAL VIEW的使用：
<font style="background-color: blue;">侧视图的意义是配合explode（或者其他的UDTF），一个语句生成把单行数据拆解成多行后的数据结果集。</font>
```sql
select goods_id2,sale_info from explode_lateral_view LATERAL VIEW explode(split(goods_id,','))goods as goods_id2;
```
[![](https://hbimg.huabanimg.com/92607356ff0bef1de13f97e2b81e8b4dafea807771a6b-3IqoBq_fw658)](https://hbimg.huabanimg.com/92607356ff0bef1de13f97e2b81e8b4dafea807771a6b-3IqoBq_fw658)

其中LATERAL VIEW explode(split(goods_id,','))goods相当于一个<font color="red">虚拟表</font>，与原表explode_lateral_view笛卡尔积关联。

也可以多重使用
```sql
select goods_id2,sale_info,area2
from explode_lateral_view 
LATERAL VIEW explode(split(goods_id,','))goods as goods_id2 
LATERAL VIEW explode(split(area,','))area as area2;
```
也是三个表笛卡尔积的结果

[![](https://hbimg.huabanimg.com/5f3681a7fd14a40aaea57ee929a715643c7bf472140c7-1mDHMm_fw658)](https://hbimg.huabanimg.com/5f3681a7fd14a40aaea57ee929a715643c7bf472140c7-1mDHMm_fw658)
现在我们解决一下上面的问题，从sale_info字段中找出所有的monthSales并且行展示

```sql
select get_json_object(concat('{',sale_info_r,'}'),'$.monthSales') as monthSales from explode_lateral_view 
LATERAL VIEW explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{'))sale_info as sale_info_r;
```

[![](https://hbimg.huabanimg.com/3271972817c388273fcfeb4c7a112cbf11b5c31580e2-r3QoFw_fw658)](https://hbimg.huabanimg.com/3271972817c388273fcfeb4c7a112cbf11b5c31580e2-r3QoFw_fw658)

最终，我们可以通过下面的句子，把这个json格式的一行数据，完全转换成二维表的方式展现


```sql
select get_json_object(concat('{',sale_info_1,'}'),'$.source') as source,
     get_json_object(concat('{',sale_info_1,'}'),'$.monthSales') as monthSales,
     get_json_object(concat('{',sale_info_1,'}'),'$.userCount') as monthSales,
     get_json_object(concat('{',sale_info_1,'}'),'$.score') as monthSales
  from explode_lateral_view 

LATERAL VIEW explode(split(regexp_replace(regexp_replace(sale_info,'\\[\\{',''),'}]',''),'},\\{'))sale_info as sale_info_1;
```

[![](https://hbimg.huabanimg.com/14f8fb356388c37293ade9b0a813ff2cd148cf0c1c8ba-aZfhhs_fw658)](https://hbimg.huabanimg.com/14f8fb356388c37293ade9b0a813ff2cd148cf0c1c8ba-aZfhhs_fw658)