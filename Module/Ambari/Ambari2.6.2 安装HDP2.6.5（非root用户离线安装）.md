[TOC]

参考：https://www.cnblogs.com/zhang-ke/p/8944240.html
# 1. 下载安装包
因为使用在线安装特别慢，所有的安装包加起来有9个G左右，所以本教程是通过下载包，然后上传到服务器，通过配置本地源的方式来实现的离线安装。也可以事先直接在服务器上下载好相应的包，如下：

- http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.0/ambari.repo
- http://public-repo-1.hortonworks.com/HDP-GPL/centos7/2.x/updates/2.6.5.0/HDP-GPL-2.6.5.0-centos7-gpl.tar.gz
- http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/HDP-2.6.5.0-centos7-rpm.tar.gz
- http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.6.2.0/ambari-2.6.2.0-centos7.tar.gz
- http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz
- http://public-repo-1.hortonworks.com/HDP/centos7/2.x/updates/2.6.5.0/hdp.repo
- http://public-repo-1.hortonworks.com/HDP-GPL/centos7/2.x/updates/2.6.5.0/hdp.gpl.repo &

特别提示：

<font style="color: red;">安装之前系统同时需要安装如下几个软件，如果能连接外网可以使用yum命令安装(也可以用安装包离线安装)：</font>


```bash
yum install -y nc libtirpc-devel python-devel  rpcbind postgresql96-server postgresql96-contrib
```

# 2. 系统环境配置



## 2.1 安装jdk（所有机器）

解压安装包，并放到/usr/local 下量<br>
```bash
[hdp@ecs-hdp-1 hdp]$ sudo tar -zxvf jdk-8ul21-linux-x64.tar.gz -C /usr/1ocal/
```

复制安装包到其他所有主机量<br>
```bash
for ((i=2;i<10;i++)) do scp /home/hdp/jdk-8u121-linux-x64.tar.gz hdp@ ecs-hdp-${i}:/home/hdp; done
```

其他主机安装jdk量<br>
```bash
sudo tar -zxvf /home/hdp/jdk-8u121-linux-x64.tar.gz  -C /usr/local/
```

配置环境变量<br>
```bash
echo "export JAVA_HOME=/usr/local/jdk1.8.0_121" >> /etc/profile
echo "export PATH=\$JAVA_HOME/bin:\$PATH" >> /etc/profile
echo "export CLASSPATH=\$CLASSPATH:\$JAVA_HOME/lib:." >> /etc/profile

source /etc/profile
```

## 2.2 修改网络及主机名
### 2.2.1 修改本机名（所有机器）

1) 临时修改：`echo hostName > /proc/sys/kernel/hostname`
2) 永久修改hostname<br>
`echo "HOSTNAME=$(hostname)" >> /etc/sysconfig/network`<br>
然后重启电脑

### 2.2.2 修改ip

### 2.2.3 修改hosts文件（所有机器）

所有涉及集群/etc/hosts文件中添加每台主机的域名信息，格式为：ip hostName


```bash
10.99.99.12 ecs-hdp-1
10.99.99.223 ecs-hdp-2
10.99.99.76 ecs-hdp-3
10.99.99.231 ecs-hdp-4
10.99.99.208 ecs-hdp-5
10.99.99.244 ecs-hdp-6
10.99.99.100 ecs-hdp-7
10.99.99.197 ecs-hdp-8
10.99.99.126 ecs-hdp-9
10.99.99.90 ecs-hdp-10
10.99.99.194 ecs-hdp-11
10.99.99.65 ecs-hdp-12
```
## 2.3 用户及免密
### 2.3.1 配置用户（所有机器）

**1) 新建hpd用户**

```bash
adduser hdp
passwd hdp #设置hdp用户密码 hadoop123！
```

**2) hdp用户添加sudo免密权限**

```bash
echo 'hdp ALL=(ALL) NOPASSWD:ALL' >> /etc/sudoers
```
 
### 2.3.2 服务器免密配置
每台主机都要做：

```bash
ssh-keygen -t dsa -P '' -f /home/hdp/.ssh/id_rsa
```


```bash
cat /home/hdp/.ssh/id_rsa.pub > /home/hdp/.ssh/authorized_keys

chmod 600 /home/hdp/.ssh/authorized_keyschmod 600 /root/.ssh/authorized_keys
```

 

仅在ambari管理节点上(ecs-hdp-1), 分发authorized_keys到集群其他主机上ecs-hdp-2到12

```bash
scp /home/hdp/.ssh/authorized_keys hdp@ecs-hdp-2:/home/hdp/.ssh/authorized_keys
```

## 2.4 linux安全设置
### 2.4.1 检查selinux
`vim /etc/selinux/config` 设置 `SELINUX=disabled`

### 2.4.2 修改文件打开限制（所有机器）

```bash
vi /etc/security/limits.conf
# End of file
* soft nofile 65536
* hard nofile 65536
* soft nproc 131072
* hard nproc 131072
```

### 2.4.3 关闭防火墙（所有机器）

```bash
systemctl status firewalld
systemctl disable firewalld
systemctl stop firewalld
```

## 2.5 时间同步
hdp-1作为时间同步服务器，其他同步hdp-1的时间<br>
Linux有自带的时间同步，需要修改配置文件。

### 2.5.1 服务端
1) 配置/etc/chrony.conf
 
注掉这几行

```bash
#server 0.centos.pool.ntp.org
#server 1.centos.pool.ntp.org
#server 2.centos.pool.ntp.org
#server 3.centos.pool.ntp.org
```

新增如下，采用本地主机时间

```bash
server 10.99.99.12 iburst 
allow 10.99.99.0/24

……

local stratum 10
```

[![1](https://www.helloimg.com/images/2020/09/11/d6e32efb0011c15b45b8c01cd52fac4af7b415eca653d8ee.png)](https://p.pstatp.com/origin/ffdd00024bd94dd798a9)

[![2](https://www.helloimg.com/images/2020/09/11/26f1caa8c3d828cf9.png)](https://p.pstatp.com/origin/137b50001599958fb86e7)

2) 重启服务：

```bash
systemctl restart chronyd.service
```
### 2.5.2 客户端
编辑配置文件/etc/chrony.conf

```
注销默认配置
#server 0.centos.pool.ntp.org
#server 1.centos.pool.ntp.org
#server 2.centos.pool.ntp.org
#server 3.centos.pool.ntp.org

#增加一下配置
server 10.99.99.12 iburst 
```

重启：

```bash
systemctl restart chronyd.service
```

xshell批量执行date命令，查看时间是否同步

## 2.6 离线yun源配置

## 2.7 mysql安装

1)	添加mysql组和mysql用户，用于设置mysql安装目录文件所有者和所属组。

```bash
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

 
2)	将二进制文件解压到指定的安装目录，我们这里指定为/usr/local，并创建软连接

```bash
cd /usr/local
tar -zxvf mysql-5.7.30-linux-glibc2.12-x86_64.tar
ln -s mysql-5.7.30-linux-glibc2.12-x86_64 mysql
```


3)	进入mysql文件夹，也就是mysql所在的目录，并更改所属的组和用户。

```bash
cd /usr/local/mysql
chown -R mysql:mysql *
```

4)	配置文件
vim /etc/my.cnf
清空重写
 

```bash
[mysqld]
    datadir=/data/mysql/mysqldata
    socket=/data/mysql/mysql.sock
    log-error=/data/mysql/log/mysqld.log
    log_bin=mysql-bin
    expire_logs_days=30
    server_id=1
    user = mysql
    port = 13306
    character_set_server = utf8
    lower_case_table_names = 1
    skip-name-resolve
    sql_mode = "STRICT_TRANS_TABLES,NO_ENGINE_SUBSTITUTION,NO_ZERO_DATE,NO_ZERO_IN_DATE,ERROR_FOR_DIVISION_BY_ZERO,NO_AUTO_CREATE_USE"
    log-bin-trust-function-creators=1

[mysqld_safe]
    pid-file=/data/mysql/mysqld.pid
[client]
    default-character-set = utf8
    socket=/data/mysql/mysql.sock
    port=3306
[mysql]
    no-auto-rehash
    prompt = "\\u@\\h : \\d \\r:\\m:\\s> "
    default-character-set = utf8
```

> **注意点**：检查配置文件中配置的目录**必须存在且目录的属组用户应是mysql**，如果不存在则手工创建，否则在启动mysql时会报错
> 
> 先查看/data/mysql 有没有文件，有的话 全删了
> 
> ```bash
> mkdir -p /data/mysql/mysqldata
> chown mysql:mysql  /data/mysql/mysqldata
> mkdir -p /data/mysql/log
> chown mysql:mysql  /data/mysql/log
> ```

 
5)	初始化MySQL

```bash
touch /data/mysql/log/mysqld.log
chmod 666 /data/mysql/log/mysqld.log   

bin/mysqld --initialize --user=mysql
bin/mysql_ssl_rsa_setup
```


 
> 以上操作会生成root用户的临时密码,需记住!!
> （由于我们先配置了配置文件所以临时密码生成在/data/mysql/log/mysqld.log）
>
> 查看用户临时mysql root密码，并记录下来,用于第一次登入<br>
**`grep 'temporary password' /data/mysql/log/mysqld.log`**
 

6)	将mysql/目录下除了data/目录的所有文件，改回root用户所有，mysql用户只需作为/data/mysql/mysqldata目录下所有文件的所有者。

```bash
cd /usr/local/mysql
chown -R root:root .  (注意后面的 .  这个点)
```

7)	添加mysql服务

```bash
cd /usr/local/mysql
cp support-files/mysql.server /etc/init.d/mysql.server
chown mysql:mysql /etc/init.d/mysql.server
```

8)	配置mysql环境变量

```bash
vim /etc/profile
最后一行添加
MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
 
source /etc/profile
```

9)	启动服务

```bash
sudo – u mysql service mysql.server start
```

10)	登录测试

```bash
mysql -uroot -p --port=13306
```

如果提示密码过期直接强制修改密码

```bash
bin/mysqladmin -uroot -p password root@12#$
```

输入之前的临时密码
首次登录会提示重设密码:

```sql
set password=password('mysql13306');
```

11)	授权root用户可远程访问

root用户登入mysql后<br>
执行 `use mysql`; 切换mysql库<br>
执行`update user set Host='%' where Host='localhost' and user='root'; `
 

12)	重启mysql

执行命令：

```bash
service mysql.server restart
```

# 3. 安装ambari-server

## 3.1 设置元数据库
进入mysql，创建ambari数据库

```sql
create database ambari default character set utf8;

create user 'ambari'@'%' identified by 'bigdata';

grant all privileges on ambari.* to ambari@'%';
flush privileges;
```

创建hive数据库
```sql
create database hive default character set utf8;
create user 'hive'@'%' identified by 'hive';
grant all privileges on hive.* to hive@'%';
flush privileges;
```

## 3.2 安装ambari-server并初始化

下载mysql连接驱动（mysql-connector-java-5.1.40.jar）,重命名为mysql-connector-java.jar，放入`/usr/share/java/`目录下
**`yum -y install ambari-server`**

**`ambari-server setup--jdbc-db=mysql--jdbc-driver=/usr/share/java/mysql-connector-java.jar`**


```bash
[hdp@hdp-1 ~]# yum -y install ambari-server
[hpd@hdp-1 ~]# ambari-server setup--jdbc-db=mysql--jdbc-driver=/usr/share/java/mysql-connector-java.jar

下面是配置执行流程，按照提示操作
（1） 提示是否自定义设置。输入：y
Customize user account for ambari-server daemon [y/n] (n)? y
（2）ambari-server 账号。
Enter user account for ambari-server daemon (root):
如果直接回车就是默认选择root用户
如果输入已经创建的用户就会显示：
Enter user account for ambari-server daemon (root):hdp
Adjusting ambari-server permissions and ownership...
（3）检查防火墙是否关闭
Adjusting ambari-server permissions and ownership...
Checking firewall...
WARNING: iptables is running. Confirm the necessary Ambari ports are accessible. Refer to the Ambari documentation for more details on ports.
OK to continue [y/n] (y)?
直接回车
（4）设置JDK。输入：3
Checking JDK...
Do you want to change Oracle JDK [y/n] (n)? y
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Oracle JDK 1.7 + Java Cryptography Extension (JCE) Policy Files 7
[3] Custom JDK
==============================================================================
Enter choice (1): 3
如果上面选择3自定义JDK,则需要设置JAVA_HOME。输入：/usr/local/jdk1.8.0_161
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/java/jdk1.8.0_131
Validating JDK on Ambari Server...done.
Completing setup...
（5）数据库配置。选择：y
Configuring database...
Enter advanced database configuration [y/n] (n)? y
（6）选择数据库类型。输入：3
Configuring database...
==============================================================================
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
==============================================================================
Enter choice (3): 3
（7）设置数据库的具体配置信息，根据实际情况输入，如果和括号内相同，则可以直接回车。如果想重命名，就输入。
Hostname (localhost):10.99.99.12
Port (3306):13306
Database name (ambari):bigdata
Username (ambari):
Enter Database Password (bigdata):mysql123
Re-Enter password: mysql123
（8）将Ambari数据库脚本导入到数据库
WARNING: Before starting Ambari Server, you must run the following DDL against the database to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql 
Proceed with configuring remote database connection properties [y/n] (y)? 
Extracting system views...
ambari-admin-2.6.0.0.267.jar
...........
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
```

## 3.3 初始化ambari元数据库

```sql
mysq1 -uroot -p --port=13306

use ambari;
source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql

quit
```

## 3.4 启动ambari

1) 修改ambari默认端口

```bash
vim /etc/ambari-server/conf/ambari.properties

增加一行
client.api.port=18080
```

2) 用hdp用户启动ambari


```bash
service ambari-server start
```


3) 查看启动是否成功

```
netstat -nltp|grep 18080

tcp6  0  0  :::18080  :::   *   LISTEN    29263/java
```

> **ssl免验证配置**
>
> 出现错误
> 
>`ERROR 2018-05-30 00:12:25,280 NetUtil.py:96 - EOF occurred in violation of protocol (_ssl.c:579)`
> 
>`ERROR 2018-05-30 00:12:25,280 NetUtil.py:97 - SSLError: Failed to connect. Please check openssl library versions.` 
> 
> 在所有的节点上
> 
> 1. `vi /etc/ambari-agent/conf/ambari-agent.ini`<br>
> 添加在[security]下 `force_https_protocol=PROTOCOL_TLSv1_2`
> 2. `vi /etc/python/cert-verification.cfg`
> ```bash
> [https]
> verify=disable
> ```

# 4. 安装配置部署HDP集群
## 4.1 登录
登录界面，默认管理员账户登录， 账户：admin 密码：admin

## 4.2 安装向导

[![1.png](https://www.helloimg.com/images/2020/09/11/984534-20180423153633613-1236299795a9a18044d3947654.png)](https://obs-hwe2-p1.obs.cn-east-2.myhwclouds.com/media/72000079/15998103299515.png)

1. 配置集群的名字为hadoop<br>
[![2.png](https://www.helloimg.com/images/2020/09/11/984534-20180423153908789-6412500271122d809f34eac0e.png)](https://obs-hwe2-p1.obs.cn-east-2.myhwclouds.com/media/72000079/15998103298955.png)

2. 选择版本并修改为本地源地址<br>
[![3.png](https://www.helloimg.com/images/2020/09/11/10a673934823e5d1e.png)](https://obs-hwe2-p1.obs.cn-east-2.myhwclouds.com/media/72000079/15998123331151.png)

3. 安装配置<br>
输入安装的服务器主机名，以回车分割。输入秘钥文件id_rsa内容

[![4.png](https://www.helloimg.com/images/2020/09/11/2a1b6b15dc903de7a.png)](https://obs-hwe2-p1.obs.cn-east-2.myhwclouds.com/media/72000079/15998103302113.png)

[![id_rsa](https://www.helloimg.com/images/2020/09/11/1c3a0ea1b280ba0b8.png)](https://s.pc.qq.com/tousu/img/20200911/6294430_1599811086.jpg)

4. 安装ambari的agent，同时检查系统问题<br>
[![1.png](https://www.helloimg.com/images/2020/09/11/1072e0847e3e925b0.png)](https://s.pc.qq.com/tousu/img/20200911/5981359_1599811298.jpg)

如果这里出了问题，请检查上面所有的步骤有没有遗漏和未设置的参数。同时在重新修改了配置以后，最好是重置ambari-server来重新进行安装


```bash
ambari-server stop    
ambari-server reset   #重置命令
ambari-server setup   #重新设置
ambari-server start
```

5. 选择要安装的服务<br>
[![2.png](https://www.helloimg.com/images/2020/09/11/206b3663f8813183d.png)](https://s.pc.qq.com/tousu/img/20200911/1277575_1599811294.jpg)

6. 选择分配服务<br>
[![3.png](https://www.helloimg.com/images/2020/09/11/3294cd42a08a21000.png)](https://s.pc.qq.com/tousu/img/20200911/5233023_1599811294.jpg)

7. 选择slaves及clients<br>
[![4.png](https://www.helloimg.com/images/2020/09/11/4bce4a085bdc2750a.png)](https://s.pc.qq.com/tousu/img/20200911/6034238_1599811303.jpg)

# 5. 集群高可用配置
## 5.1 hdfs高可用
1) 在Ambari Web页面, select Services > HDFS > Summary. 
2) 选择 Service Actions, 点击Enable NameNode HA.

[![1.png](https://ae04.alicdn.com/kf/U46bbb59085054d7fbd04968ad0ba851d8.jpg)](https://www.helloimg.com/images/2020/09/11/1e63c6b564768a77b.png)

Stop HBASE<br>
[![4.png](https://s.pc.qq.com/tousu/img/20200911/8317285_1599813531.jpg)](https://ae01.alicdn.com/kf/Udae6fed6b5644bfdaf72266dd8aaca31S.jpg)

配置第二台namenode,及同步节点<br>
[![2.png](https://ae02.alicdn.com/kf/U4dd9d291b162449a935965d765ac65dfg.jpg)](https://www.helloimg.com/images/2020/09/11/25db231e39b558723.png)

按指示执行以下命令<br>
[![3.png](https://ae01.alicdn.com/kf/U533c4ed3b9a54cc0a7d1a3dc421c55b4o.jpg)](https://www.helloimg.com/images/2020/09/11/3d107dac59b074c8f.png)

[![5.png](https://s.pc.qq.com/tousu/img/20200911/3230096_1599813747.jpg)](https://ae01.alicdn.com/kf/U597037b87e754157926daab25fad79b2E.jpg)

[![6.png](https://s.pc.qq.com/tousu/img/20200911/1379395_1599813745.jpg)](https://ae01.alicdn.com/kf/U7508cb130130432ea11de021469b8285e.jpg)

重启hdfs
[![9.png](https://p.pstatp.com/origin/13789000245cb212273df)](https://s.pc.qq.com/tousu/img/20200911/9352009_1599813965.jpg)
## 5.2 yarn高可用
1) 在Ambari Web页面, select Services > YARN > Summary. 
2) 选择 Service Actions, 点击Enable ResourceManager HA.

配置Resourcemanager主机<br>
[![7.png](https://p.pstatp.com/origin/137940001800324b71d37)](https://s.pc.qq.com/tousu/img/20200911/4145751_1599813966.jpg)

添加zookeeper信息<br>
[![8.png](https://p.pstatp.com/origin/137420002f8be4ed6cfe1)](https://s.pc.qq.com/tousu/img/20200911/9480875_1599813965.jpg)
