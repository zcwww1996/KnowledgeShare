[TOC]
# 系统
## 清屏

1) system clear;
2) \! clear
3) 快捷键:Ctrl+L

## 登陆
### 连接到本机上的MYSQL

```sql
mysql -u root -p
```
回车后提示你输密码.注意用户名前可以有空格也可以没有空格，但是密码前必须没有空格，否则让你重新输入密码

### 连接到远程主机上的MYSQL
假设远程主机的IP为：192.168.110.110，用户名为root,密码为abcd123。则键入以下命令：

```sql
mysql -u root -p abcd123 -h192.168.110.110
```

## 修改表名

执行重命名操作，把test修改为test1

```sql
mysql> rename table test to test1;
Query OK, 0 rows affected (0.08 sec)
```

# 导入导出
## 导入
### 一、首先建空数据库

格式：

`mysql>create database 数据库名;`

举例：

`mysql>create database abc;`

### 二、导入数据库

- 方法一：

1、选择数据库

`mysql>use abc;`

2、设置数据库编码

`mysql>set names utf8;`

3、导入数据（注意sql文件的路径）

`mysql>source /home/abc/abc.sql;`

- 方法二（常用）：

格式：`mysql -u用户名 -p密码 数据库名 < 数据库名.sql`

举例：mysql -uabc_f -p abc < abc.sql

## 导出
### 一、导出数据和表结构：

格式： `mysqldump -u用户名 -p密码 数据库名 > 数据库名.sql`

举例： /usr/local/mysql/bin/ mysqldump -uroot -p abc > abc.sql

敲回车后会提示输入密码

### 二、只导出表结构
格式：`mysqldump -u用户名 -p密码 -d 数据库名 > 数据库名.sql`

举例：/usr/local/mysql/bin/ mysqldump -uroot -p -d abc > abc.sql 

注：/usr/local/mysql/bin/ —> mysql的data目录

# 删除
## 删除数据库
命令：`drop database <数据库名>`

例如：删除名为 t_data的数据库<br/>
mysql> drop database t_data;

例子1：删除一个已经确定存在的数据库

```sql
mysql> drop database t_data;
   Query OK, 0 rows affected (0.00 sec)
```

例子2：删除一个不确定存在的数据库

```sql
mysql> drop database t_data;
ERROR 1008 (HY000): Can't drop database 't_data'; database doesn't exist
    //发生错误，不能删除't_data'数据库，该数据库不存在。
mysql> drop database if exists t_data;
Query OK, 0 rows affected, 1 warning (0.00 sec)//产生一个警告说明此数据库不存在
mysql> create database t_data;
Query OK, 1 row affected (0.00 sec)
mysql> drop database if exists t_data;//if exists 判断数据库是否存在，不存在也不产生错误
Query OK, 0 rows affected (0.00 sec)
```
