[TOC]

# 1. 部署方案
部署高可用mysql，安装版本：mysql-5.7.30<br>
部署ip：10.99.99.12、10.99.99.223<br>
模式：主主互备<br>

# 2. 单机部署步骤
2台主机 安装mysql,根据事先规划好的,在相应的机器上安装mysql

两台mysql服务器，安装过程除 ==**my.cnf中[mysqld]**== 内容外，均一致

## 2.1 基础环境配置
1) 添加mysql组和mysql用户，用于设置mysql安装目录文件所有者和所属组。

```
groupadd mysql
useradd -r -g mysql -s /bin/false mysql
```

2) 将二进制文件解压到指定的安装目录，我们这里指定为/usr/local，创建出mysql的软连接

```
cd /usr/local
tar -zxvf mysql-5.7.30-linux-glibc2.12-x86_64.tar
ln -s mysql-5.7.30-linux-glibc2.12-x86_64 mysql
```
 
3) 进入mysql文件夹，也就是mysql所在的目录，并更改所属的组和用户。

```
cd /usr/local/mysql
chown -R mysql:mysql *
```

4) 编写/etc/my.cnf

**清空重写 /etc/my.cnf**


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

    auto_increment_increment=2
    auto_increment_offset=1
    
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

> 注意点：检查配置文件中配置的目录必须存在且目录的属组用户应是mysql，如果不存在则手工创建，否则在启动mysql时会报错
> 先查看/data/mysql 有没有文件，有的话 全删了
> 
> ```bash
> mkdir -p /data/mysql/mysqldata
> chown mysql:mysql  /data/mysql/mysqldata
> mkdir -p /data/mysql/log
> chown mysql:mysql  /data/mysql/log
> ```

第一台mysql，my.cnf中[mysqld]内容为

```bash
    auto_increment_increment=2
    auto_increment_offset=1
```
第二台mysql，my.cnf中[mysqld]内容为

```bash
    auto_increment_increment=2
    auto_increment_offset=2
```

5)	配置mysql环境变量

```bash
vim /etc/profile
最后一行添加
MYSQL_HOME=/usr/local/mysql
export PATH=$PATH:$MYSQL_HOME/bin
 
source /etc/profile
```

## 2.2 安装mysql
1) 初始化MySQL

```bash
touch /data/mysql/log/mysqld.log
chmod 666 /data/mysql/log/mysqld.log   

bin/mysqld --initialize --user=mysql
bin/mysql_ssl_rsa_setup
```

**以上操作会生成root用户的临时密码,需记住!!**
（由于我们先配置了配置文件所以临时密码生成在/data/mysql/log/mysqld.log）

查看用户临时mysql root密码，并记录下来,用于第一次登入<br>
`grep 'temporary password' /data/mysql/log/mysqld.log`


2) 将mysql/目录下除了data/目录的所有文件，改回root用户所有，mysql用户只需作为/data/mysql/mysqldata目录下所有文件的所有者。

```bash
cd /usr/local/mysql
chown -R root:root .  (注意后面的 .  这个点)
```

3) 添加mysql服务

```bash
cd /usr/local/mysql
cp support-files/mysql.server /etc/init.d/mysql.server
chown mysql:mysql /etc/init.d/mysql.server
```

4) 启动服务

```bash
sudo – u mysql service mysql.server start
```

 

5) 登录测试

```bash
mysql -uroot -p -h 10.99.99.221 -P 13306
```

首次登录会提示重设密码:<br>
`set password=password('mysql13306');`

**授权root用户可远程访问**<br>
root用户登入mysql后<br>
执行 `use mysql;` 切换mysql库<br>
执行`update user set Host='%' where Host='localhost' and user='root'; `

6) 重启mysql

```bash
执行命令：
service mysql.server restart
```

# 3. mysql服务器互相配置备份

## 3.1 两台数据库互相创建备用用户
1) 在10.99.99.12上执行

```bash
进入mysql 
mysql -uroot -p --port=13306
执行sql：
CREATE USER 'backupuser'@'10.99.99.223' IDENTIFIED BY 'mysql13306';
GRANT REPLICATION SLAVE ON *.* TO 'backupuser'@'10.99.99.223' IDENTIFIED BY 'mysql13306';
FLUSH PRIVILEGES;
```


2) 在10.99.99.223上执行

```bash
进入mysql 
mysql -uroot -p --port=13306
CREATE USER 'backupuser'@'10.99.99.12' IDENTIFIED BY 'mysql13306';
GRANT REPLICATION SLAVE ON *.* TO 'backupuser'@'10.99.99.12' IDENTIFIED BY 'mysql13306';
FLUSH PRIVILEGES;
```

## 3.2 数据库互相告知备份

`10.99.99.12：show master status;`<br>
 [![10.99.99.12_status](https://shop.io.mi-img.com/app/shop/img?id=shop_1ec80741dee41a5d51d414c805c4d5e5.jpeg)](https://ae02.alicdn.com/kf/Ue831d2482005426e9d98477c2ad68f83p.jpg)
 
`10.99.99.223: show master status;`<br>
[![10.99.99.223_status](https://shop.io.mi-img.com/app/shop/img?id=shop_95309d888baf2f21e27b11da2d9fee05.jpeg)](https://img04.sogoucdn.com/app/a/100520146/3efc24d0fc00e22ac337de883a5a1ba3)

10.99.99.12：
```sql
change master to master_host='10.99.99.223',master_user='backupuser',master_password='mysql13306',master_port=13306,master_log_file='mysql-bin.000003',master_log_pos=1929;
```

10.99.99.223:
```sql
change master to master_host='10.99.99.12',master_user='backupuser',master_password='mysql13306',master_port=13306,master_log_file='mysql-bin.000004',master_log_pos=1928;
```

10.99.99.12：

```sql
start slave;
 
show slave  status\G

Slave_IO_Running：Yes
Slave_SQL_Running：Yes
正确即可
```

[![10.99.99.12_Slave](https://shop.io.mi-img.com/app/shop/img?id=shop_fccf4063fec462f63d12051805999f01.jpeg)](https://s1.ax1x.com/2020/07/29/ae36ne.jpg)

同理10.99.99.223:

```sql
start slave;

show slave  status\G
```

# 4. nginx动态代理与负载均衡

需求<br>
1. 需要云数据库开放外网地址，但是mysql服务器未开放外网端口，需要通过代理访问
2. 两台mysql服务器为主主备份，需要负载均衡，不能只往一台分发任务

部署机器：10.99.99.12<br>
版本：nginx-1.17.10<br>
nginx下载连接 http://nginx.org/download <br>

## 4.1 ngxin解压编译

```bash
tar -zxvf ginx-1.17.10.tar.gz
mv nginx-1.17.10 /usr/local/
 
编译：
cd /usr/local/nginx
./configure --prefix=/usr/local/nginx --with-stream
 
make
make  install
```

> - nginx编译时添加`--with-stream`模块可以把内网ip端口映射到外网地址去
> - `sbin/nginx -V`命令可以查看编译的nginx模块
> 
> ```bash
> [root@ecs-hdp-1 nginx] /usr/local/nginx/sbin/nginx -V
> nginx version: nginx/1.17.10
> built by gcc 4.8.5 20150623 (Red Hat 4.8.5-36) (GCC) 
> configure arguments: --prefix=/usr/local/nginx --with-stream
> ```


## 4.2 配置mysql消息转发

```bash
vim /usr/local/nginx/conf/nginx.conf
清空后，添加：
 
#user  nginx;
worker_processes  1;
events {
    worker_connections  1024;
}
stream{
  include /usr/local/nginx/conf/mysql.conf;
 }
```



```bash
vim /usr/local/nginx/conf/mysql.conf
 
server {
	listen 23306;
	proxy_connect_timeout 10s;
	proxy_timeout 300s;
	proxy_pass mysql;
}

upstream mysql {
	server 10.99.99.12:13306 weight=5 max_fails=3 fail_timeout=30s;
	server 10.99.99.223:13306 weight=5 max_fails=3 fail_timeout=30s;
}
```


- 测试Nginx：<br>
`sbin/nginx -t`
- 启动Nginx：<br>
`sbin/nginx -c /usr/local/nginx/conf/nginx.conf`
