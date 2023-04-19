[TOC]

# 一、环境
CentOS 6.6
MySQL 8.0.14
离线RPM方式安装
参考资料：https://blog.csdn.net/qq_42729058/article/details/83857000
# 二、初始设置
## 2.1、查询服务器安装的MySQL

```bash
[root@linux04 ~]# rpm -qa | grep -i mysql
  MySQL-client-5.6.38-1.el6.x86_64
  MySQL-server-5.6.38-1.el6.x86_64
```

## 2.2、停止MySQL服务

```bash
[root@linux04 ~]# service mysql stop
```

## 2.3、卸载已经安装的MySQL

```bash
[root@linux04 ~]# yum -y remove MySQL-*
```

## 2.4、查找遗留的MySQL文件

```bash
[root@linux04 ~]# find / -name mysql
/var/lib/mysql
/usr/lib64/mysql
```

## 2.5、删除卸载前一个版本MySQL的遗留文件

```bash
[root@linux04 ~]# rm -rf /usr/lib64/mysql
```

# 三、安装mysql
## 3.1、下载mysql8.0
链接一：https://cdn.mysql.com/archives/mysql-8.0/mysql-8.0.14-1.el6.x86_64.rpm-bundle.tar</br>
链接二：https://cloud.189.cn/t/JzIruqJZBvYf
## 3.2、创建解压目录并进行解压

```bash
[root@linux04 ~]# mkdir -p /usr/local/mysql
[root@linux04 ~]# tar -xf mysql-8.0.14-1.el6.x86_64.rpm-bundle.tar -C /usr/local/mysql
```

## 3.3、安装MySQL 8.0

```bash
[root@linux04 ~]# cd /usr/local/mysql
[root@linux04 mysql]# rpm -ivh mysql-community-{server,client,common,libs}-8.0.14-1.el6.x86_64.rpm
```

## 3.4、配置MySQL配置文件
参考资料：https://www.cnblogs.com/wajika/p/6323026.html

```bash
[root@linux04 mysql] # vim /etc/my.cnf

# For advice on how to change settings please see
# http://dev.mysql.com/doc/refman/8.0/en/server-configuration-defaults.html

[mysqld]
#
# Remove leading # and set to the amount of RAM for the most important data
# cache in MySQL. Start at 70% of total RAM for dedicated server, else 10%.
#
# ……
#
# Remove leading # to revert to previous value for default_authentication_plugin,
# this will increase compatibility with older clients. For background, see:
# https://dev.mysql.com/doc/refman/8.0/en/server-system-variables.html#sysvar_default_authentication_plugin
#
# 默认使用“mysql_native_password”插件认证
default-authentication-plugin=mysql_native_password
# 错误日志
log-error=/var/log/mysqld.log
# 为MySQL客户程序与服务器之间的本地通信指定一个套接字文件(仅适用于UNIX/Linux系统;默认设置一般是/var/lib/mysql/mysql.sock文件)
#在Windows环境下，如果MySQL客户与服务器是通过命名管道进行通信的，–sock选项给出的将是该命名管道的名字(默认设置是MySQL)
socket=/var/lib/mysql/mysql.sock
# 指定一个存放进程ID的文件(仅适用于UNIX/Linux系统)
# Init-V脚本需要使用这个文件里的进程ID结束mysqld进程
pid-file=/var/run/mysqld/mysqld.pid
# 设置mysql数据库的数据的存放目录
datadir=/var/lib/mysql
# 指定一个TCP/IP通信端口(通常是3306端口)
port=3306
# 新目录和数据表的名字是否只允许使用小写字母;这个选项在Windows环境下的默认设置是1(只允许使用小写字母)
lower_case_table_name=0
# 允许最大连接数
max_connections=200
# 允许连接失败的次数。这是为了防止有人从该主机试图攻击数据库系统
max_connect_errors=10
# 服务端使用的字符集默认为UTF8
character-set-server=utf8
# 创建新表时将使用的默认存储引擎
default-storage-engine=INNODB
# 设置mysql客户端默认字符集
default-character-set=utf8
```

这一步非常重要，在MySQL 8.0中有部分配置参数只能在初始化数据库前进行配置和更改，不支持初始化之后再更改，如忽略大小写配置就是如此，`lower_case_table_names`
## 3.5、初始化MySQL

```bash
[root@linux04 mysql] # mysqld --initialize
```

## 3.6、修改MySQL的datadir权限

```bash
[root@linux04 mysql] # chown -R mysql:mysql /var/lib/mysql
```

## 3.7、启动mysql服务

```bash
[root@linux04 mysql] # service mysqld start
```

## 3.8、查看初始化之后的root用户密码

```bash
[root@linux04 mysql] # grep 'temporary password' /var/log/mysqld.log

2019-01-28T03:17:32.746212Z 5 [Note] [MY-010454] [Server] A temporary password is generated for root@localhost: ih)Ncpwgk4ne
```

初始密码：ih)Ncpwgk4ne

## 3.9、mariadb安全配置向导
参考资料：http://blog.itpub.net/30936525/viewspace-2016528/

```bash
[root@linux04 mysql]# mysql_secure_installation -uroot -p 
Enter password: 

Securing the MySQL server deployment.
```

登陆安全配置向导，输入初始密码

```bash
The existing password for the user account root has expired. Please set a new password.

New password: 

Re-enter new password:
```

输入新密码

```bash
VALIDATE PASSWORD COMPONENT can be used to test passwords and improve security.
//密码验证插件，为了提高安全性，需要验证密码
It checks the strength of password // 它会检查密码的强度
and allows the users to set only those passwords which are secure enough.  //只允许用户设置足够安全的密码
Would you like to setup VALIDATE PASSWORD component?

Press y|Y for Yes, any other key for No: n
```

是否安装密码验证插件 **N**

```bash
Using existing password for root.
Change the password for root ? ((Press y|Y for Yes, any other key for No) : n

 ... skipping.
```

是否修改root密码 **N**

```bash
By default, a MySQL installation has an anonymous user,
allowing anyone to log into MySQL without having to have
a user account created for them. This is intended only for
testing, and to make the installation go a bit smoother.
You should remove them before moving into a production
environment.

Remove anonymous users? (Press y|Y for Yes, any other key for No) : y
Success.
```

默认情况下，MySQL有一个匿名用户，这个匿名用户，不必有一个用户为他们创建，匿名用户允许任何人登录到MySQL，在正式环境使用的时候，建议你移除它

删除系统创建的匿名用户（生产环境建议删除） **Y**

```bash
Normally, root should only be allowed to connect from
'localhost'. This ensures that someone cannot guess at
the root password from the network.

Disallow root login remotely? (Press y|Y for Yes, any other key for No) : n

 ... skipping.
```

一般情况下，root用户只允许使用"localhost"方式登录，以此确保，不能被某些人通过网络的方式访问，测试环境可以不禁止

<font style="color: red;">禁止root用户远程登录 **N**</font>

```bash
By default, MySQL comes with a database named 'test' that
anyone can access. This is also intended only for testing,
and should be removed before moving into a production
environment.


Remove test database and access to it? (Press y|Y for Yes, any other key for No) : n

... skipping.
```

默认情况下，MySQL数据库中有一个任何用户都可以访问的test库，在正式环境下，应该移除掉，这也仅仅是为了测试

删除test数据库 **Y**

```bash
Reloading the privilege tables will ensure that all changes
made so far will take effect immediately.

Reload privilege tables now? (Press y|Y for Yes, any other key for No) : y
Success.

All done! 
```

刷新权限表，以确保所有的修改可以立刻生效 **Y**

<font style="background-color: Yellow
;">查看MySQL安装是否成功<font>

```bash
mysqladmin -V
```

## 3.10、登陆mysql

```bash
[root@linux04 mysql] #  mysql -h localhost -u root -p
```

# 四、个性配置
## 4.1、配置远程登录
### 4.1.1、改表法

可能是你的帐号不允许从远程登陆，只能在localhost。这个时候只要在localhost的那台电脑，登入mysql后，更改 "mysql" 数据库里的 "user" 表里的 "host" 项，从"localhost"改称"%"
- 1、登录MySQL

```bash
mysql -u root -p
```

输入密码
- 2、选择 mysql 数据库

```sql
use mysql;
```

因为 mysql 数据库中存储了用户信息的 user 表。
- 3、查看user表中当前 root 用户的相关信息

```sql
select host, user, authentication_string, plugin from user;
```

执行完上面的命令后会显示一个表格

[![mysql访问权限](https://pic.downk.cc/item/5e24135a2fb38b8c3c757648.png "[mysql访问权限")](https://i0.wp.com/i.loli.net/2020/01/19/MVQmjroLiBkhCdE.png)

查看表格中 root 用户的 host，默认应该显示的 localhost，只支持本地访问，不允许远程访问。
- 4、修改root 用户可以在任意主机远程访问

```sql
update user set host = '%' where user = 'root';
```

% 表示通配所有host，可以访问远程。

### 4.1.2、授权法

- 1、在安装mysql的机器上登陆mysql:

```bash
mysql -u root - p
```

- 2、授权 root 用户的所有权限并设置远程访问

```sql
GRANT ALL PRIVILEGES ON *.* TO 'root'@'%'IDENTIFIED BY '123456' WITH GRANT OPTION;
```

GRANT ALL ON 表示所有权限，% 表示通配所有 host，可以访问远程

更改加密方式

```sql
ALTER USER 'root'@'%' IDENTIFIED BY '123456' PASSWORD EXPIRE NEVER;
ALTER USER 'root'@'%' IDENTIFIED WITH mysql_native_password BY '123456';
FLUSH PRIVILEGES;
```

- 3、刷新权限

```sql
flush privileges;
```

# 五、连接MySQL
## 5.1 MySQL 8.0 API
驱动包版本 **mysql-connector-java-8.0.12.jar**。

数据库 URL 需要声明是否使用 SSL 安全验证及指定服务器上的时区：

```java
static final String DB_URL = jdbc:mysql://localhost:3306/runoob?useSSL=false&serverTimezone=UTC;
conn = DriverManager.getConnection(DB_URL,USER,PASS);
```

原本的驱动器是:

```java
Class.forName("com.mysql.jdbc.Driver");
```

在 IDEA 里面提示是: **Loading class `com.mysql.jdbc.Driver`. This is deprecated. The new driver class is `com.mysql.cj.jdbc.Driver`. The driver is automatically registered via the SPI and manual loading of the driver class is generally unnecessary**

意思是说原本的驱动器不赞成 或者 是废弃了，自动换成了新的驱动器 `com.mysql.cj.jdbc.Driver`


```java
Class.forName("com.mysql.cj.jdbc.Driver");
```

## 5.2 MySQL连接驱动及支持的MySQL和Java版本

目前有两个 MySQL Conztor/j 版本可用:

- Conztor/j8.0 ( <font style="color: orange;">以前为 Conztor/j 6.0</font> ; 请参阅MySQL Conztor/j中的更改8.0.7 以了解版本号更改的说明) 是 java 8 平台的纯4类型 JAVA JDBC 4.2 驱动程序。它提供了与 MySQL 5.6、5.7 和8.0 的所有功能的兼容性。Conztor\ j 8.0 提供了易于开发的功能, 包括在驱动程序管理器中自动注册、标准化有效性检查、分类 Sqlexts、支持大型更新计数、支持本地和偏移日期时间变体，这些功能都在java.time包中。支持JDBC-4.x XML处理, 支持每个连接客户端信息, 以及支持NCHAR、NVARCHAR和NCLOB数据类型。

- Conztor\ J5.1 也是符合 JDBC 3.0、4.0、4.1 和4.2 规范的4型纯 Java JDBC 驱动程序。它提供了与 MySQL 5.6、5.7 和8.0 的所有功能的兼容性。Concitor/j5.1 由自己的手册覆盖.

下表总结了相关的 Connector/J 版本, 以及 JDBC 驱动程序类型的详细信息、支持的 JDBC Api 版本、支持 MySQL Server 的版本、支持 JRE、构建所需的 JDK 以及每个连接器/j 版本:

**表 2.1 Connector/J 版本摘要**

|Connector/J version|JDBC version|MySQL Server version|JRE Supported|JDK Required for Compilation|Status|
|---|---|---|---|---|---|
|8.0(6.0)|4.2|5.6, 5.7, 8.0|1.8.x|1.8.x|General availability.<br/><font style="color:green;">**Recommended version**.</font>|
|5.1|3.0, 4.0, 4.1, 4.2|5.6`*`, 5.7`*`, 8.0`*`|1.5.x, 1.6.x, 1.7.x, 1.8.x`*`|1.5.x and 1.8.x|General availability|

来源:https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-versions.html