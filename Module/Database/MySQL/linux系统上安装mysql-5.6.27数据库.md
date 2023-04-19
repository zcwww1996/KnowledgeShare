[TOC]
# 一、环境
CentOS 7.2
MySQL 5.6.43
离线tar方式安装
root用户安装


下载地址：http://dev.mysql.com/downloads/mysql/
下载说明：上边的下载地址是最新版的，如果想下载老版本可以点击页面中的超链接“Looking for previous GA versions?”
# 二、安装
## 1.卸载老版本MySQL
查找并删除mysql有关的文件
```shell
find / -name mysql|xargs rm -rf
```
## 2.解压，重命名
找到软件包，解压到某文件夹（/home/hadoop/installs）下，并重命名，如图所示：
```shell
tar -zxf mysql-5.6.43-linux-glibc2.12-x86_64.tar.gz
mv mysql-5.6.43-linux-glibc2.12-x86_64 mysql-5.6.43
```
[![解压](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_zNzTM.png "解压")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_zNzTM.png "解压")


## 3.添加用户和用户组
先检查是否有mysql用户组和mysql用户
```shell
groups mysql
```
[![判断用户组是否存在](https://b-ssl.duitang.com/uploads/item/201906/10/20190610161632_UrhJy.png "判断用户组是否存在")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610161632_UrhJy.png "判断用户组是否存在")<br>
若无，则添加；


```shell
#添加用户组
groupadd mysql
#添加用户mysql到用户组mysql
useradd -r -m -g mysql mysql
```
## 5.修改当前目录权限为mysql
```shell
cd /home/hadoop/installs/mysql-5.6.43
chown -R mysql:mysql ./
```
[![修改目录权限](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_LjwNG.png "修改目录权限")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_LjwNG.png "修改目录权限")


## 6.创建数据库目录
```shell
mkdir /data/mysql_data
chown -R mysql:mysql /data/mysql_data
```
[![数据库目录](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_AWlCS.png "数据库目录")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_AWlCS.png "数据库目录")
## 7.安装并指定用户和data文件夹位置


```shell
./scripts/mysql_install_db --user=mysql --basedir=/home/hadoop/installs/mysql-5.6.43 --datadir=/data/mysql_data
```
[![初始化Mysql](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_GMlUw.png "初始化Mysql")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_GMlUw.png "初始化Mysql")
## 8.复制mysql到服务自动启动里面
```shell
cp support-files/mysql.server /etc/init.d/mysqld
```
## 9.修改权限为755 也就是root可以执行
```shell
chmod 755 /etc/init.d/mysqld
```
## 10.复制配置文件到etc下，因为默认启动先去etc下加载配置文件，选择覆盖
```shell
cp support-files/my-default.cnf /etc/my.cnf
```
[![复制my.cnf](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_FwnWA.png "复制my.cnf")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_FwnWA.png "复制my.cnf")
## 11.修改启动脚本，修改basedir路径和datadir路径
```shell
vi /etc/init.d/mysqld
```
修改内容：
```shell
basedir=/home/hadoop/installs/mysql-5.6.43
datadir=/data/mysql_data
```
## 12.启动服务
如图所示表示开启成功
```shell
service mysqld start
```
[![启动MySQL服务](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_wyfWx.png "启动MySQL服务")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_wyfWx.png "启动MySQL服务")
## 13.加入环境变量
编辑 /etc/profile，这样可以在任何地方用mysql命令了
```shell
export PATH=$PATH:/usr/local/mysql/bin
```
注：

①修改了环境变量，要使用source /etc/profile命令，重新部署环境变量

②可能有些资料PATH是放到前面的，这里放到后面为了解决新旧版本冲突问题
## 14.登录到mysql数据库
mysql -u root
注：开始安装的时候不需要密码，直接就可以登录
熟悉的界面又出现了：
[![登陆mysql](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_zYxfQ.png "登陆mysql")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_zYxfQ.png "登陆mysql")
## 15. 关闭服务
```shell
service mysqld stop
```
[![关闭MySQL服务](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_SJ5UH.png "关闭MySQL服务")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_SJ5UH.png "关闭MySQL服务")
## 16. 为数据库设置密码
```shell
set password for mysql@localhost = password（‘123456’）
flush privileges;
```
实例：
[![设置密码](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_3ACVA.png "设置密码")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610160752_3ACVA.png "设置密码")
# 附加知识:(可能你不会遇到)
 <font color="red">可能会遇到的问题：</font>安装完成后自动进行root用户密码修改和相关用户配置完成后，用工具远程连接报错，是由于没有给远程连接用户，连接权限问题
## 解决1：修改内部user表
更改“mysql”数据库 ‘user’表 ‘host’项，从‘localhost’改成‘%’
```shell
use mysql;
select user,host from user; 
update user set host = '%' where user ='mysql';
grant all privileges on *.* to root@'%' identified by 'root'; flush privileges;
flush privileg
```


## 解决2：直接授权
```shell
GRANT ALL PRIVILEGES ON *.* TO ‘root'@'%' IDENTIFIED BY ‘youpassword' WITH GRANT OPTION;
flush privileges;
```
**说明:**

 <font style="background-color: yellow;">GRANT privileges ON databasename.tablename TO ' <font color="red">username'@'host</font>'</font>
- privileges：用户的操作权限，如SELECT，INSERT，UPDATE等，如果要授予所的权限则使用ALL
- databasename：数据库名
- tablename：表名，如果要授予该用户对所有数据库和表的相应操作权限则可用*表示，如*.*

来源： https://blog.csdn.net/lg_49/article/details/80231535

参考：https://www.cnblogs.com/wangdaijun/p/6132632.html