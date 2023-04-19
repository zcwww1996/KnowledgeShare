[TOC]
#### 1. 获取logbiao里当天的数据
```sql
CREATE TABLE IF NOT EXISTS d_gucp.rps_did_type_id_software(
id     string COMMENT '用户id',
interval  string COMMENT '抓取前置窗口的间隔时间',
productname   string COMMENT '软件名称',
times   string  COMMENT '当前窗口前置时被抓取到的次数',
servertime   string COMMENT 'servertime') COMMENT  '软件使用详情表'
PARTITIONED BY(p_event_date string)
ROW FORMAT DELIMITED 
FIELDS TERMINATED BY '\u0005'
COLLECTION ITEMS TERMINATED BY '\u0002'
MAP KEYS TERMINATED BY '\u0003'
STORED AS  TEXTFILE;
```

```sql
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
insert into table d_gucp.rps_did_type_id_software partition(p_event_date='${v_date}')
select  id,`interval`,get_json_object(concat('{',topwindow_01,'}'),'$.productname') as productname,get_json_object(concat('{',topwindow_01,'}'),'$.times') as times,servertime from
(select id,servertime,
    get_json_object(param1_value,'$.interval') as `interval`,
    split(regexp_replace(regexp_replace(get_json_object(param1_value,'$.topwindow'),'\\[\\{',''),'}]',''),'},\\{')  as topwindow 
from (select
    regexp_replace(regexp_replace(split(preserve6,
    '#')[2],
    'MACWIFI:',
    ''),
    'SN:',
    '') as id,
    substr(server_timestamp,1,10) as servertime,base64RC4(param1_value) as param1_value,p_event_date
from d_pc_ace.rps__h_date_partition_log_pc_ace
where p_event_date ='${v_date}'
  and app_key = 'O9F58YHSM971'
  and event_action = 'TW0001'
  and param1_value not like '%\\%EF\\%BF\\%BD%'
  and param1_value not like '%\\%E0\\%88\\%BE\\%14D%'
  and param1_value not like '%\\%E0\\%88\\%BE\\%14g%'
  and param1_value not like '%\\%E0\\%90\\%B5r%'
  and param1_value not like '%\\%2N%'
  and param1_value not like '%\\%:F%'
  and param1_value not like '%\\%E8\\%A1\\%85%'
  and param1_value not like '%\\%7B\\%22%'
  and param1_value not like '%3zjHeMrzQXotrxnobn66%'
  and param1_value not like '%3zjHeMrzQXotrxnobnS3%'
  and param1_value not like  '%\\%E0\\%98\\%BAv%'
  and param1_value not like  '%x\\%E0\\%97\\%B%'
  and param1_value not like '%3zjKedP3WmJu\\%2BWCpfifzCrQY2xPr3x918T5kpKzYf7kA6399eQqCykj2QlqSL0zOKk\\%2FvxywP6aZzwCv1jqWmqXjKU4q99S%2FFyGkxxQ\\%2BXyxcoq8HthPZf5PmdF9x0seplnf3ecUUBhUknlus5rrxHsBaHYtsUcfX511vzQ3ecZbrd%2B4alnQcqI2syTZLEmqeXa%2BLQc4w9pbKwX6g\\%3@D%'
  and param1_value not like '%\\%10\\%E0\\%97\\%88%'
  and param1_value not like '%\\%E0\\%90\\%86\\%07\\%01\\%E0\\%98\\%BA\\%07\\%01%'
  and param1_value not like   '%\\%E0\\%A1\\%A9%'
  and param1_value not like   '%\\%14\\%01\\%16\\%01\\%E0\\%88\\%A3\\%1D\\%07\\%5C%'
  and param1_value not like  '%https\\%3A\\%2F\\%2F%'
  and param1_value not like '%\\%20\\%E0\\%84\\%80\\%1D\\%20\\%E0\\%84\\%80\\%1D\\%20\\%E0\\%84\\%80\\%1D%'
  and param1_value not like '%x\\%3C\\%3E\\%01\\%E0\\%90\\%A0F\\%06%'
  and param1_value not like '%\\%E0%'
  and param1_value not like  '%��ǹi��m�Gd�%') t )  t1
LATERAL VIEW  explode(topwindow) productname_table as topwindow_01
group by id,`interval`,get_json_object(concat('{',topwindow_01,'}'),'$.productname'),get_json_object(concat('{',topwindow_01,'}'),'$.times'),servertime
```

##### 1. /设备信息/PC管家信息/PC管家信息/PC应用偏好
 --来源1：
```sql
with temp as(select lps_did,soft_name from d_pc_ace.dwd_nb_software_software_open  where p_event_date='${v_date}' and lps_did is not null  and lps_did <> '' group by lps_did,soft_name)
--insert into table d_gucp.stts_pc_manager_software_category partition(l_date='${v_date}')
select distinct 
'global' cid,
    'sn',
 m.lps_did,
case when software_category='股票网银' then '10021000010000100019'
when software_category='音频编辑' then '10021000010000100005'
when software_category='3D制作软件' then '10021000010000100020'
when software_category='音乐播放' then '10021000010000100023'
when software_category='视频编辑' then '10021000010000100011'
when software_category='CAD设计软件' then '10021000010000100002'
when software_category='浏览器'  then '10021000010000100012'
when software_category='办公学习' then '10021000010000100010'
when software_category='聊天通讯' then '10021000010000100018'
when software_category='下载工具' then '10021000010000100016'
when software_category='程序开发' then '10021000010000100015'
when software_category='压缩刻录' then '10021000010000100003'
when software_category='系统工具' then '10021000010000100014'
when software_category='驱动程序' then '10021000010000100024'
when software_category='影视播放' then '10021000010000100008'
when software_category='手机管理' then '10021000010000100009'
when software_category='桌面壁纸' then '10021000010000100001'
when software_category='游戏'   then '10021000010000100013'
when software_category='输入法'  then '10021000010000100021'
when software_category='网络应用' then '10021000010000100022'
when software_category='图像编辑' then '10021000010000100004'
when software_category='杀毒防护' then '10021000010000100006'
else '10021000010000100007' end as software_category,
'${v_date}'
from temp m  join d_pc_ace.ace_device_software_category n 
on  m.soft_name= n.software_name
```

--来源2

```sql
insert into table d_gucp.stts_pc_manager_software_category partition(l_date='${v_date}')
select distinct 
'global' cid,
    'sn',
 m.id,
case when software_category='股票网银' then '10021000010000100019'
when software_category='音频编辑' then '10021000010000100005'
when software_category='3D制作软件' then '10021000010000100020'
when software_category='音乐播放' then '10021000010000100023'
when software_category='视频编辑' then '10021000010000100011'
when software_category='CAD设计软件' then '10021000010000100002'
when software_category='浏览器'  then '10021000010000100012'
when software_category='办公学习' then '10021000010000100010'
when software_category='聊天通讯' then '10021000010000100018'
when software_category='下载工具' then '10021000010000100016'
when software_category='程序开发' then '10021000010000100015'
when software_category='压缩刻录' then '10021000010000100003'
when software_category='系统工具' then '10021000010000100014'
when software_category='驱动程序' then '10021000010000100024'
when software_category='影视播放' then '10021000010000100008'
when software_category='手机管理' then '10021000010000100009'
when software_category='桌面壁纸' then '10021000010000100001'
when software_category='游戏'   then '10021000010000100013'
when software_category='输入法'  then '10021000010000100021'
when software_category='网络应用' then '10021000010000100022'
when software_category='图像编辑' then '10021000010000100004'
when software_category='杀毒防护' then '10021000010000100006'
else '10021000010000100007' end as software_category,
servertime as updatetime
from  (select id,servertime,productname as soft_name from d_gucp.rps_did_type_id_software where id is not null and id <> '') m
join d_pc_ace.ace_device_software_category n
on  m.soft_name= n.software_name

```