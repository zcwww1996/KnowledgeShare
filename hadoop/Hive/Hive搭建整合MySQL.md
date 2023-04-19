[TOC]
# 1. 环境
- Centos6
- Hive2.3.4
- MySQL5.1

# 2. 安装步骤
## 2.1 配置环境变量：

```bash
vim /etc/profile
HIVE_HOME=/usr/local/hive-2.3.4
```
执行：source /etc/profile （使环境变量生效）
## 2.2 修改配置文件

```bash
mv hive-env.sh.template hive-env.sh
mv hive-default.xml.template hive-site.xml
```
### 2.2.1 修改hive-site.xml

```xml
<configuration>
  <!-- WARNING!!! This file is auto generated for documentation purposes ONLY! -->
  <!-- WARNING!!! Any changes you make to this file will be ignored by Hive.   -->
  <!-- WARNING!!! You must make your changes in hive-site.xml instead.         -->
  <!-- Hive Execution Parameters -->
 <property>
  <name>hive.default.fileformat</name>
  <value>TextFile</value>
  <description>Default file format for CREATE TABLE statement. Options are TextFile and SequenceFile. Users can explicitly say CREATE TABLE ... STORED AS &lt;TEXTFILE|SEQUENCEFILE&gt; to override</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionURL</name>
  <value>jdbc:mysql://192.168.111.103:3306/onhive</value>
  <description>JDBC connect string for a JDBC metastore</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionDriverName</name>
  <value>com.mysql.jdbc.Driver</value>
  <description>Driver class name for a JDBC metastore</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionUserName</name>
  <value>hive</value>
  <description>username to use against metastore database</description>
</property>
<property>
  <name>javax.jdo.option.ConnectionPassword</name>
  <value>123456</value>
  <description>password to use against metastore database</description>
</property> 
</configuration>
```
### 2.2.2 修改hive-env .sh

```bash
HADOOP_HOME=/usr/local/hadoop
export HIVE_CONF_DIR=/usr/local/hive/conf
export HIVE_AUX_JARS_PATH=/usr/local/hive/lib
```

## 2.3 mysql连接驱动
将mysql连接jar放到/hive/lib下

mysql-connector-java-5.1.35.jar

## 2.4 创建用户名密码
给mysql创建用户hive/密码123456

```sql
CREATE USER 'hive'@'192.168.111.101' IDENTIFIED BY "123456";
grant all privileges on *.* to hive@192.168.111.101 identified by '123456';
```

## 2.5 启动hdfs
```bash
start-dfs.sh
```

## 2.6 初始化操作
从 Hive 2.1 版本开始, 我们需要先运行 schematool 命令来执行初始化操作

```bash
schematool -dbType mysql -initSchema
```

**错误：当我们输入./schematool -initSchema -dbType mysql的时候，会出现以下错误**

```
Metastore connection URL: jdbc:mysql://192.168.*./hive?createDatabaseIfNotExist=true 
Metastore Connection Driver : com.mysql.jdbc.Driver 
Metastore connection User: hiveuser 
Starting metastore schema initialization to 2.1.0 
Initialization script hive-schema-2.1.0.mysql.sql 
Error: Duplicate key name ‘PCS_STATS_IDX’ (state=42000,code=1061) 
org.apache.hadoop.hive.metastore.HiveMetaException: Schema initialization FAILED! Metastore state would be inconsistent !! 
Underlying cause: java.io.IOException : Schema script failed, errorcode 2 
Use –verbose for detailed stacktrace. 
* schemaTool failed *
```

以上错误查看mysql是否已经创建了hive这个表, 如果创建，你想从新安装的话，把那个你创建的表删了即可

正确：

```
[root@Linux01 conf]# schematool -dbType mysql -initSchema
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hive-2.3.4/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop-2.7.6/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]
Metastore connection URL:        jdbc:mysql://192.168.111.103:3306/hive
Metastore Connection Driver :    com.mysql.jdbc.Driver
Metastore connection User:       hive
Starting metastore schema initialization to 2.3.0
Initialization script hive-schema-2.3.0.mysql.sql
Initialization script completed
schemaTool completed
```

## 2.7 启动hive

```bash
[root@Linux01 conf]# hive
which: no hbase in (/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin:/usr/local/jdk1.8.0/bin:/usr/local/hadoop-2.7.6/bin:/usr/local/hadoop-2.7.6/sbin:/usr/local/apache-hive-2.3.4/bin:/usr/local/spark-2.1.0/bin:/root/bin:/usr/local/jdk1.8.0/bin:/usr/local/hadoop-2.7.6/bin:/usr/local/hadoop-2.7.6/sbin:/usr/local/hive-2.3.4/bin:/usr/local/spark-2.1.0/bin)
SLF4J: Class path contains multiple SLF4J bindings.
SLF4J: Found binding in [jar:file:/usr/local/hive-2.3.4/lib/log4j-slf4j-impl-2.6.2.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: Found binding in [jar:file:/usr/local/hadoop-2.7.6/share/hadoop/common/lib/slf4j-log4j12-1.7.10.jar!/org/slf4j/impl/StaticLoggerBinder.class]
SLF4J: See http://www.slf4j.org/codes.html#multiple_bindings for an explanation.
SLF4J: Actual binding is of type [org.apache.logging.slf4j.Log4jLoggerFactory]

Logging initialized using configuration in jar:file:/usr/local/hive-2.3.4/lib/hive-common-2.3.4.jar!/hive-log4j2.properties Async: true
Hive-on-MR is deprecated in Hive 2 and may not be available in the future versions. Consider using a different execution engine (i.e. spark, tez) or using Hive 1.X releases.
hive> show tables;
OK
Time taken: 4.72 seconds
```


**启动hive时可能会出现下面的错误：**


```
hive启动报错 java.net.URISyntaxException: Relative path in absolute URI: ${system:java.io.tmpdir%7D/$%7B
```

配置文件(hive-site.xml)修改如下属性：（主要是设置目录）

```xml
<property>
    <name>hive.exec.scratchdir</name>
    <value>/tmp/hive</value>
    <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
  </property>

  <property>
    <name>hive.exec.local.scratchdir</name>
    <value>/tmp/hive/local</value>
    <description>Local scratch space for Hive jobs</description>
  </property>

  <property>
    <name>hive.downloaded.resources.dir</name>
    <value>/tmp/hive/resources</value>
    <description>Temporary local directory for added resources in the remote file system.</description>
  </property>
```
## 2.8 navicat远程连接查看

[![navicat_hive](https://upload.cc/i1/2019/12/04/WS2Zvz.png
 "navicat_hive")](https://img02.sogoucdn.com/app/a/100520146/01c0c0454ca6d05fbbdcf5f920c9dec6)

