[TOC]

# 安装
## 下载sqoop-1.4.7
http://mirror.bit.edu.cn/apache/sqoop/1.4.7/


```bash
[hadoop@node1 ~]$ tar -zxvf sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
```

## 设置环境变量


```bash
[hadoop@node1 bin]$ sudo vi /etc/profile
export JAVA_HOME=/home/hadoop/jdk1.7.0_67
export HADOOP_HOME=/home/hadoop/hadoop-2.7.1
export ZK_HOME=/home/hadoop/zookeeper-3.4.6
export HIVE_HOME=/home/hadoop/apache-hive-1.2.1-bin
export HBASE_HOME=/home/hadoop/hbase-1.1.2
export SQOOP_HOME=/home/hadoop/sqoop-1.4.7.bin__hadoop-2.6.0
export PATH=$PATH:${JAVA_HOME}/bin:${HADOOP_HOME}/bin:${HADOOP_HOME}/sbin:${ZK_HOME}/bin:${HIVE_HOME}/bin:${HBASE_HOME}/bin:${SQOOP_HOME}/bin
```

**使用sqoop help报错**

```bash
[hadoop@node1 bin]$ sqoop help
Warning: /home/hadoop/sqoop-1.4.7.bin__hadoop-2.6.0/../hcatalog does not exist! HCatalog jobs will fail.
Please set $HCAT_HOME to the root of your HCatalog installation.
Warning: /home/hadoop/sqoop-1.4.7.bin__hadoop-2.6.0/../accumulo does not exist! Accumulo imports will fail.
Please set $ACCUMULO_HOME to the root of your Accumulo installation.
Warning: /home/hadoop/sqoop-1.4.7.bin__hadoop-2.6.0/../zookeeper does not exist! Accumulo imports will fail.
Please set $ZOOKEEPER_HOME to the root of your Zookeeper installation.
15/11/24 13:44:31 INFO sqoop.Sqoop: Running Sqoop version: 1.4.7
usage: sqoop COMMAND [ARGS]
```

把sqoop/bin/configure-sqoop里面的两段内容注释掉就可以了。根据fail搜索


```bash
Available commands:
  codegen            Generate code to interact with database records
  create-hive-table  Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
  import             Import a table from a database to HDFS
  import-all-tables  Import tables from a database to HDFS
  import-mainframe   Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases     List available databases on a server
  list-tables        List available tables in a database
  merge              Merge results of incremental imports
  metastore          Run a standalone Sqoop metastore
  version            Display version information

See 'sqoop help COMMAND' for information on a specific command.
```

## 设置配置文件

```bash
[hadoop@node1 conf]$ cp sqoop-env-template.sh sqoop-env.sh
[hadoop@node1 conf]$ vi sqoop-env.sh
[hadoop@node1 conf]$ vi sqoop-site.xml
```


## 复制需要的类

```bash
[hadoop@node1 ~]$ cp $HADOOP_HOME/share/hadoop/common/hadoop-common-2.7.1.jar $SQOOP_HOME/lib
[hadoop@node1 mysql-connector-java-5.1.37]$ cp mysql-connector-java-5.1.37-bin.jar $SQOOP_HOME/lib
```


mysql-connector-java-5.1.37-bin.jar这个包才有用

## 附配置：

```bash
[hadoop@node1 conf]$ vi sqoop-env.sh
export HADOOP_COMMON_HOME=/home/hadoop/hadoop-2.7.1/
export HADOOP_MAPRED_HOME=/home/hadoop/hadoop-2.7.1/
export HBASE_HOME=/home/hadoop/hbase-1.1.2
export HIVE_HOME=/home/hadoop/apache-hive-1.2.1-bin
export ZOOCFGDIR=/home/hadoop/zookeeper-3.4.6/conf
```


```bash
[hadoop@node1 conf]$ vi sqoop-site.xml 
sqoop list-databases --connect jdbc:mysql://node1 --username root --password 123456

  <property>
    <name>sqoop.metastore.client.autoconnect.url</name>
    <value>jdbc:hsqldb:file:/tmp/sqoop-meta/meta.db;shutdown=true</value>
    <value>jdbc:mysql://node1/hive?useUnicode=true&characterEncoding=utf-8</value>
    <description>The connect string to use when connecting to a
      job-management metastore. If unspecified, uses ~/.sqoop/.
      You can specify a different path here.
    </description>
  </property>
  <property>
    <name>sqoop.metastore.client.autoconnect.username</name>
    <value>root</value>
    <description>The username to bind to the metastore.
    </description>
  </property>
  <property>
    <name>sqoop.metastore.client.autoconnect.password</name>
    <value>123456</value>
    <description>The password to bind to the metastore.
    </description>
  </property>
```

# 测试


```bash
[hadoop@node1 hadoop]$ sudo service mysqld start
正在启动 mysqld： [确定]
```

## mysql导入到hive中
```sql
[hadoop@node1 hadoop]$ mysql -uroot -p123456
mysql>use
mysql> create table a(id int,name varchar(50));
mysql> insert into a values(1,'a1');
mysql> insert into a values(2,'a2');
mysql> commit;
mysql> select * from a;
+------+------+
| id   | name |
+------+------+
|    1 | a1   |
|    2 | a2   |
+------+------+
```


```bash
[hadoop@node1 lib]$ sqoop create-hive-table --connect jdbc:mysql://node1/hive --username root --password 123456 --table a --hive-table a --fields-terminated-by ',' --hive-overwrite
[hadoop@node1 conf]$ sqoop list-tables --connect jdbc:mysql://node1/hive --username root --password 123456
a
```

## 导出

```bash
[hadoop@node1 lib]$ sqoop import --connect jdbc:mysql://node1/hive --username root --password 123456 --table a --hive-table a --hive-import --fields-terminated-by ',' --hive-overwrite -m 1
```

```sql
mysql> create table b(id int,name varchar(50));    --先建立表
Query OK, 0 rows affected (0.13 sec)

mysql> select * from b;
+------+------+
| id   | name |
+------+------+
|    1 | a1   |
|    2 | a2   |
+------+------+
2 rows in set (0.15 sec)
```

## 将a文件夹导出到mysql中的b表

```bash
[hadoop@node1 lib]$ sqoop export --connect jdbc:mysql://node1/hive --username root --password 123456 --table b --export-dir /user/hive/warehouse/a --input-fields-terminated-by ','
```

## sqoop eval连接mysql直接select和dml

```bash
[hadoop@node1 lib]$ sqoop eval --connect jdbc:mysql://node1/hive --username root --password 123456 --query 'select * from a'
[hadoop@node1 lib]$ sqoop eval --connect jdbc:mysql://node1/hive --username root --password 123456 -e 'select * from a'
[hadoop@node1 lib]$ sqoop eval --connect jdbc:mysql://node1/hive --username root --password 123456 -e "insert into a values (4,'a4')"
[hadoop@node1 lib]$ sqoop eval --connect jdbc:mysql://node1/hive --username root --password 123456 --query "insert into a values (5,'a5')"
[hadoop@node1 lib]$ sqoop eval --connect jdbc:mysql://node1/hive --username root --password 123456 -e "select * from a"

sqoop job --create myjob -- import --connect jdbc:mysql://node1/hive --username root --password 123456 --table a  -m 1 --target-dir /test/a_old
sqoop job --list
sqoop job --show myjob
sqoop job --exec myjob
sqoop job --exec myjob -- --username root -P
sqoop job --delete myjob
```

## sqoop codegen生成java代码

```java
[hadoop@node1 ~]$  sqoop codegen --connect jdbc:mysql://node1/hive --username root --password 123456 --table a
...
15/11/25 00:25:21 INFO orm.CompilationManager: Writing jar file: /tmp/sqoop-hadoop/compile/0fc68731200a4f397cac20ef4a4c718f/a.jar

[hadoop@node1 ~]$ ll /tmp/sqoop-hadoop/compile/0fc68731200a4f397cac20ef4a4c718f/
总用量 28
-rw-rw-r--. 1 hadoop hadoop  8715 11月 25 00:25 a.class
-rw-rw-r--. 1 hadoop hadoop  3618 11月 25 00:25 a.jar
-rw-rw-r--. 1 hadoop hadoop 10346 11月 25 00:25 a.java
```

## mysql数据增量导入hive


```
Incremental import arguments:  --增量导入
   --check-column <column>        Source column to check for incremental
                                  change
   --incremental <import-type>    Define an incremental import of type
                                  'append' or 'lastmodified'
   --last-value <value>           Last imported value in the incremental
                                  check column
```

## append不支持

```bash
Append mode for hive imports is not  yet supported. Please remove the parameter --append-mode
```

### 1.mysql中建表

```sql
drop table a;
create table a(id int,name varchar(50),crt_date timestamp);
insert into a values(1,'a1',sysdate());
insert into a values(2,'a2',sysdate());
insert into a values(3,'a3',sysdate());
select * from a;
mysql> select * from a;
+------+------+---------------------+
| id   | name | crt_date            |
+------+------+---------------------+
|    1 | a1   | 2015-11-25 12:41:39 |
|    2 | a2   | 2015-11-25 12:41:39 |
|    3 | a3   | 2015-11-25 12:41:39 |
+------+------+---------------------+
```


### 2.第一次mysql导出到a_1,a_1不要创建

```bash
sqoop import --connect jdbc:mysql://node1/hive --username root --password 123456 --table a  -m 1 --target-dir /test/a_1
```


### 3.插入数据

```sql
mysql> insert into a values(4,'a4',sysdate());
mysql> insert into a values(5,'a5',sysdate());
mysql> select * from a;
+------+------+---------------------+
| id   | name | crt_date            |
+------+------+---------------------+
|    1 | a1   | 2015-11-25 12:41:39 |
|    2 | a2   | 2015-11-25 12:41:39 |
|    3 | a3   | 2015-11-25 12:41:39 |
|    4 | a4   | 2015-11-25 13:46:42 |
|    5 | a5   | 2015-11-25 13:46:42 |
+------+------+---------------------+
```


### 4.第二次导出

```bash
sqoop import --connect jdbc:mysql://node1/hive --username root --password 123456 --table a  -m 1 --target-dir /test/a_2 --incremental lastmodified --check-column crt_date --last-value "2015-11-25 12:41:40"

--where crt_date>="2015-11-25 12:41:40",时间要比id=3大一点,不然会把前面3条导进去
```



```bash
[hadoop@node1 ~]$ hadoop fs -cat /test/a_old/*
1,a1,2015-11-25 12:41:39.0
2,a2,2015-11-25 12:41:39.0
3,a3,2015-11-25 12:41:39.0

[hadoop@node1 ~]$ hadoop fs -cat /test/a_new/*
4,a4,2015-11-25 13:46:42.0
5,a5,2015-11-25 13:46:42.0
```


### 5.生成a.jar

```bash
sqoop codegen --connect jdbc:mysql://node1/hive --username root --password 123456 --table a
/tmp/sqoop-hadoop/compile/6e3034f9fa9b0b46716ff31aee94c2e4/a.jar
```



```bash
[hadoop@node1 ~]$ ll /tmp/sqoop-hadoop/compile/6e3034f9fa9b0b46716ff31aee94c2e4/
-rw-rw-r--. 1 hadoop hadoop 10321 11月 25 14:31 a.class
-rw-rw-r--. 1 hadoop hadoop  4201 11月 25 14:31 a.jar
-rw-rw-r--. 1 hadoop hadoop 12969 11月 25 14:31 a.java
```


### 6.合并，a_merge不要创建，--class-name a(这里是表名)

```bash
sqoop merge --new-data /test/a_2 --onto /test/a_1 --target-dir /test/a_merge --jar-file /tmp/sqoop-hadoop/compile/6e3034f9fa9b0b46716ff31aee94c2e4/a.jar --class-name a --merge-key id
```

```bash
[hadoop@node1 ~]$ hadoop fs -ls /test/a_merge
-rw-r--r--   3 hadoop supergroup          0 2015-11-25 15:57 /test/a_merge/_SUCCESS
-rw-r--r--   3 hadoop supergroup        135 2015-11-25 15:57 /test/a_merge/part-r-00000    --hive后面load进去后会在这里删除
```

```bash
[hadoop@node1 6e3034f9fa9b0b46716ff31aee94c2e4]$ hadoop fs -cat /test/a_merge/part*
1,a1,2015-11-25 12:41:39.0
2,a2,2015-11-25 12:41:39.0
3,a3,2015-11-25 12:41:39.0
4,a4,2015-11-25 13:46:42.0
5,a5,2015-11-25 13:46:42.0
```

### 7.导入hive

```sql
hive> create table a(id int,name string,crt_date string) row format delimited fields terminated by ',';

hive> load data inpath '/test/a_merge/part*' into table a;

hive> show create table a;
OK
CREATE TABLE `a`(
  `id` int, 
  `name` string, 
  `crt_date` string)
ROW FORMAT DELIMITED 
  FIELDS TERMINATED BY ',' 
STORED AS INPUTFORMAT 
  'org.apache.hadoop.mapred.TextInputFormat' 
OUTPUTFORMAT 
  'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION
  'hdfs://odscluster/user/hive/warehouse/a'
TBLPROPERTIES (
  'COLUMN_STATS_ACCURATE'='true', 
  'numFiles'='1', 
  'totalSize'='135', 
  'transient_lastDdlTime'='1448437545')
Time taken: 0.485 seconds, Fetched: 17 row(s)
```


### 8.检查数据文件，会从hdfs中移动到hive

```bash
[hadoop@node1 ~]$ hadoop fs -ls /test/a_merge
-rw-r--r--   3 hadoop supergroup          0 2015-11-25 15:57 /test/a_merge/_SUCCESS

[hadoop@node1 ~]$ hadoop fs -ls /user/hive/warehouse/a
-rwxr-xr-x   3 hadoop supergroup        135 2015-11-25 15:57 /user/hive/warehouse/a/part-r-00000
```



```sql
hive> select * from a;
OK
1       a1      2015-11-25 12:41:39.0
2       a2      2015-11-25 12:41:39.0
3       a3      2015-11-25 12:41:39.0
4       a4      2015-11-25 13:46:42.0
5       a5      2015-11-25 13:46:42.0
```
