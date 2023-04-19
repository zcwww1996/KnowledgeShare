[TOC]
## 安装环境
CentOS 7.2
MySQL 5.7.24
离线tar方式安装
mysql-5.7.24-linux-glibc2.12-x86_64.tar

## 1、前期工作
### 删除旧版MySQL相关文件
```bash
find / -name mysql|xargs rm -rf
```
### 下载
下载tar包，这里使用wget从官网下载

wget https://dev.mysql.com/get/Downloads/MySQL-5.7/mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
## 2、安装
安装到/usr/local/mysql-5.7.24下

### 解压
```bash
tar -xvf mysql-5.7.24-linux-glibc2.12-x86_64.tar.gz
```
### 移动
```bash
mv mysql-5.7.24-linux-glibc2.12-x86_64 /usr/local/
```
### 重命名
```bash
mv /usr/local/mysql-5.7.24-linux-glibc2.12-x86_64 /usr/local/mysql-5.7.24
```
## 3、新建data目录
```bash
mkdir /usr/local/mysql-5.7.24/data
```
## 4、新建mysql用户、mysql用户组
### mysql用户组
```bash
groupadd mysql
```
### mysql用户
```bash
useradd -r -m -g mysql mysql
```
## 5、修改所属组
将/usr/local/mysql-5.7.24的所有者及所属组改为mysql
```bash
chown -R mysql:mysql /usr/local/mysql-5.7.24
```
## 6、配置
==**建议记住最后一行生成的临时密码**==
```bash
/usr/local/mysql-5.7.24/bin/mysql_install_db --user=mysql --basedir=/usr/local/mysql-5.7.24/ --datadir=/usr/local/mysql-5.7.24/data
```


**（1）如果出现以下错误：**

```bash
2018-07-14 06:40:32 [WARNING] mysql_install_db is deprecated. Please consider switching to mysqld --initialize
2018-07-14 06:40:32 [ERROR]   Child process: /usr/local/mysql/bin/mysqldterminated prematurely with errno= 32
2018-07-14 06:40:32 [ERROR]   Failed to execute /usr/local/mysql/bin/mysqld --bootstrap --datadir=/usr/local/mysql/data --lc-messages-dir=/usr/local/mysql/share --lc-messages=en_US --basedir=/usr/local/mysql
-- server log begin --

-- server log end --
```

**则使用以下命令：**

```bash
/usr/local/mysql-5.7.24/bin/mysqld --user=mysql --basedir=/usr/local/mysql-5.7.24/ --datadir=/usr/local/mysql-5.7.24/data --initialize
```

**（2）如果出现以下错误：**
```bash
/usr/local/mysql-5.7.24/bin/mysqld: error while loading shared libraries: libnuma.so.1: cannot open shared object file: No such file or directory
```
1、则执行以下命令：
```bash
yum -y install numactl

```
2、完成后继续安装：

```bash
/usr/local/mysql/bin/mysqld --user=mysql --basedir=/usr/local/mysql/ --datadir=/usr/local/mysql/data --initialize
```


## 7、编辑/etc/my.cnf
```bash
[mysqld]
datadir=/usr/local/mysql-5.7.24/data
basedir=/usr/local/mysql-5.7.24
socket=/tmp/mysql.sock
log_bin=mysql-bin
server_id=100

user=mysql
port=3306
character-set-server=utf8
# 取消密码验证
skip-grant-tables
# Disabling symbolic-links is recommended to prevent assorted security risks
symbolic-links=0
# skip-grant-tables
[mysqld_safe]
log-error=/var/log/mysqld.log
pid-file=/var/run/mysqld/mysqld.pid
```
## 8、开启服务
### 将mysql加入服务
```bash
cp /usr/local/mysql-5.7.24/support-files/mysql.server /etc/init.d/mysql
```
### 开机自启
```bash
chkconfig mysql on
```
### 开启
```bash
service mysql start
```

> **补充**<br>
> 查看开机自启服务列表：`chkconfig --list`<br>
> 添加mysql为开机服务: `chkconfig --add mysql`或`chkconfig mysql on`<br>
> 其中 ==`mysql`为/etc/init.d/中的文件名==<br>
>
>看到3、4、5状态为开或者为 on 则表示成功。如果是 关或者 off 则执行一下：`chkconfig --level 345 mysql on`

### 加入环境变量
编辑 /etc/profile，这样可以在任何地方用mysql命令了
```bash
PATH=/usr/local/mysql-5.7.24/bin:$PATH
```
**source /etc/profile**

注：

<font color="red">①修改了环境变量，要使用**source /etc/profile**命令，重新部署环境变量</font>

<font color="red">②可能有些资料PATH是放到前面的，这里放到后面为了解决新旧版本冲突问题</font>

## 9、设置密码
### 方案一
#### 登录
**(由于/etc/my.cnf中设置了取消密码验证，所以此处密码任意)**
```bash
mysql -u root -p
```
错误信息:

```bash
ERROR 2002 (HY000): Can't connect to local MySQL server through socket '/tmp/mysql.sock' (2)
```

解决办法：
打开`/etc/my.cnf`,看看配置的socket位置是什么目录，发现socket目录在`socket=/var/lib/mysql/mysql.sock`，路径和报错出来的路径不一样，直接创建一个软链接过去：<br>
`ln -s /var/lib/mysql/mysql.sock /tmp/mysql.sock` 
#### 操作mysql数据库
```bash
use mysql;
```

#### 修改密码
```bash
update user set authentication_string=password('你的密码') where user='root';
flush privileges;
exit;
```

#### 修改my.cnf
将/etc/my.cnf中的skip-grant-tables删除
### 登录再次设置密码
（不知道为啥如果不再次设置密码就操作不了数据库了）
```bash
mysql -u root -p
ALTER USER 'root'@'localhost' IDENTIFIED BY '修改后的密码';
exit;
```
### 方案二
**/etc/my.cnf中使用默认设置，不取消密码验证**
#### 登陆

```bash
mysql -u root -p
Enter password: 输入第6步的临时密码
```

#### 修改密码
```sql
ALTER USER 'root'@'localhost' IDENTIFIED BY '修改后的密码';
flush privileges;
exit;
```

## 10、允许远程连接
```bash
mysql -u root -p
use mysql;
update user set host='%' where user = 'root';
flush privileges;
exit;
```

## 11、添加快捷方式
```bash
ln -s /usr/local/mysql-5.7.24/bin/mysql /usr/bin
```

来源： https://www.cnblogs.com/daemon-/p/9009360.html