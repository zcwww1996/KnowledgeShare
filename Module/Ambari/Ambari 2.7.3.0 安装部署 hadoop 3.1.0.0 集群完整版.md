[Ambari 2.7 安装配置教程](https://mp.weixin.qq.com/s?__biz=MzU3MTc1NzU0Mg==&mid=2247483940&idx=1&sn=7c35e51f61458ac5f92909a05012fb99&scene=19#wechat_redirect)

[B站：Ambari 2.7.3.0 安装部署 hadoop 3.1.0.0 集群完整版，附带移除 SmartSense 服务及 FAQ](https://www.bilibili.com/video/av80315899)
#### 一、配置说明

##### 1\. 硬件环境

![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDPDNfyu0uNYlvOoTQBtwCic4ajMJffIicZFZODib77I0zlq0A1YEeZMbAA/640?wx_fmt=png)

##### 2\. 软件环境

![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDiciaQjrepKM36zibR2Kkw37PXQicXnjqQdS1vSibZ6O2ogAtfRZbywvVyBQ/640?wx_fmt=png)

#### 二、修改主机名和hosts文件

##### 1\. 修改主机名（三台主机分别修改主机名）

```
# 使用hostnamectl命令修改主机名，执行该命令后立即生效，但必须需要重启Xshell连接# 以其中一台为例，代码如下hostnamectl set-hostname node1.ambari.com# 其余的机器也使用hostnamectl命令修改主机名...(略)
```

##### 2\. 修改hosts文件（三台主机的hosts文件均修改为下图所示）

```
# 添加机器ip与主机名映射vim /etc/hosts
```

![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDGF33Wm1nyz5BdqaQwzxKCcCibbN6k3pOXcADVnED4u4VUYYWlmxVbag/640?wx_fmt=png)

#### 三、关闭防火墙和selinux

##### 1\. 防火墙设置

```
# 查看防火墙状态systemctl status firewalld# 查看开机是否启动防火墙服务systemctl is-enabled firewalld# 关闭防火墙systemctl stop firewalldsystemctl disable firewalld# 再次查看防火墙状态和开机防火墙是否启动systemctl status firewalldsystemctl is-enabled firewalld
```

##### 2\. 禁用selinux

```
# 永久性关闭selinux（重启服务器生效）sed -i 's/SELINUX=enforcing/SELINUX =disabled/' /etc/selinux/config# 临时关闭selinux（立即生效，重启服务器失效）setenforce 0# 查看selinux状态getenforce# disabled为永久关闭，permissive为临时关闭，enforcing为开启
```

#### 四、免密登陆

各个主机均执行以下操作：

```
## 生成密钥对ssh-keygen -t rsa   ## 一路回车即可## 进入.ssh目录，如果目录不存在则创建cd ~/.ssh## 将公钥导入至authorized_keyscat id_rsa.pub >> authorized_keys## 修改文件权限chmod 700 ~/.sshchmod 600 authorized_keys
```

在node1.ambari.com上执行：

```
## 配置主从互相免密登陆[root@node1 ~]# cat ~/.ssh/id_rsa.pub | ssh root@node2.ambari.com 'cat - >> ~/.ssh/authorized_keys'[root@node1 ~]# cat ~/.ssh/id_rsa.pub | ssh root@node3.ambari.com 'cat - >> ~/.ssh/authorized_keys'ssh node2.ambari.com ssh node3.ambari.com # 验证主机点是否可以免密登陆从节点，执行exit命令退出即可。
```

**备注：**要想实现多主机互相免密，可参考文章：《[Linux多台主机互相免密](http://mp.weixin.qq.com/s?__biz=MzU3MTc1NzU0Mg==&mid=2247483660&idx=1&sn=7b10b29eea5932e48a8dccfcbb6867b7&chksm=fcda0785cbad8e93d38bb81633cdb6372333ab9ff27f2d85c8148336c02abd543d268a86b5c0&scene=21#wechat_redirect)》

#### 五、安装JDK

下载链接: https://pan.baidu.com/s/1rlqZejpZZqio9RPzgnGOEg 提取码: j47n ；内有`jdk-8u151-linux-x64.tar.gz`和`mysql-connector-java.jar`文件。

*   mkdir /usr/java；将下载的压缩包上传到java文件夹内
    
*   解压压缩包：tar zxvf jdk-8u151-linux-x64.tar.gz
    
*   配置jdk环境变量：
    

```
# 编辑/etc/profile,文末插入以下内容：# set javaexport JAVA_HOME=/usr/java/jdk1.8.0_151export PATH=$JAVA_HOME/bin:$PATH
```

*   使环境变量生效：source /etc/profile
    
*   安装验证：java -version
    

#### 六、安装mysql

mysql5.7 centos7:

https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm

mysql5.7 centos6:

https://dev.mysql.com/get/mysql57-community-release-el6-11.noarch.rpm

mysql5.6 centos7:

https://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm

mysql5.6 centos6:

https://dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm

##### 1\. 检查本地资源库中是否有mysql的rpm包

```
rpm -qa | grep mysql# 删除相关rpm包rpm -ev <rpm包名> --nodeps
```

##### 2\. 搭建mysql5.7的yum源

```
# 下载mysql5.7的rpm包wget https://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm# 安装第一步下载的rpm文件，安装成功后/etc/yum.repos.d/目录下会增加两个文件yum -y install mysql57-community-release-el7-11.noarch.rpm# 查看mysql57的安装源是否可用，如不可用请自行修改配置文件（/etc/yum.repos.d/mysql-community.repo）使mysql57下面的enable=1# 若有mysql其它版本的安装源可用，也请自行修改配置文件使其enable=0yum repolist enabled | grep mysql
```

##### 3\. 安装mysql

```
yum install mysql-community-server
```

##### 4\. 设置mysql

```
# 启动mysql服务service mysqld start# 查看root密码grep "password" /var/log/mysqld.log# 登陆mysqlmysql -u root -pEnter password: # 为了可以设置简单密码set global validate_password_policy=0;set global validate_password_length=4;# 立即修改密码，执行其他操作报错：SET PASSWORD FOR 'root'@'localhost' = PASSWORD('newpass');# 我们创建密码为root123
```

##### 5\. 新增ambari用户并增加权限

```
mysql -uroot -proot123CREATE USER 'ambari'@'%' IDENTIFIED BY 'ambari';GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'%';CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'ambari';GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'localhost';CREATE USER 'ambari'@'node1.ambari.com' IDENTIFIED BY 'ambari';GRANT ALL PRIVILEGES ON *.* TO 'ambari'@'node1.ambari.com';  //本地主机名FLUSH PRIVILEGES;
```

**注：**删除用户命令：

```
Delete FROM user Where User='your_user' and Host='your_host';FLUSH PRIVILEGES;
```

##### 6\. 使用ambari用户登陆并创建数据库

```
mysql -uambari -pambariCREATE DATABASE ambari;exit;
```

#### 七、设置时钟同步

请参考我写的另一篇文章：《[Linux NTP时钟同步](http://mp.weixin.qq.com/s?__biz=MzU3MTc1NzU0Mg==&mid=2247483777&idx=1&sn=94f1ae87e32a18ee5df693564e3db7ff&chksm=fcda0708cbad8e1e4d335b6293474c13dcc2d0ee645562e93b89ad7ae58c5251a953e646cb3f&scene=21#wechat_redirect)》

#### 八、搭建yum本地源

##### 1\. 安装httpd和wget服务

```
# 安装httpdyum -y install httpd.x86_64systemctl enable httpd.servicesystemctl start httpd.service# 安装wgetyum -y install wget
```

##### 2\. 下载ambari和hdp包

```
# 将tar包下载到/var/www/htmlcd /var/www/htmlwget http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.1.0/ambari-2.7.1.0-centos7.tar.gzwget http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.0.1.0/HDP-3.0.1.0-centos7-rpm.tar.gzwget http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz# 解压上面三个包tar zxvf ambari-2.7.1.0-centos7.tar.gztar zxvf HDP-3.0.1.0-centos7-rpm.tar.gztar zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz
```

##### 3\. 新建repo文件

*   新建ambari.repo文件
    

```
[ambari]name=ambaribaseurl=http://node1.ambari.com/CentOS-7/ambari-2.6.0.0enabled=1gpgcheck=0
```

*   新建HDP.repo文件
    

```
[HDP]name=HDPbaseurl=http://node1.ambari.com/CentOS-7/HDPpath=/enabled=1gpgcheck=0
```

*   新建HDP-UTILS.repo文件
    

```
[HDP-UTILS]name=HDP-UTILSbaseurl=http://liuyzh1.xdata/CentOS-7/HDP-UTILSpath=/enabled=1gpgcheck=0
```

将以上文件放入`/etc/yum.repos.d/`目录下。

#### 九、在主节点安装ambari-server

##### 1\. 安装

```
yum -y install ambari-server
```

##### 2\. 将mysql-connector-java.jar包拷贝到/usr/share/java目录下

##### 3\. 修改配置文件

```
echo server.jdbc.driver.path=/usr/share/java/mysql-connector-java.jar >> /etc/ambari-server/conf/ambari.properties
```

##### 4\. 安装ambari-server

```
ambari-server setup --jdbc-db=mysql --jdbc-driver=/usr/share/java/mysql-connector-java.jar
```

![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDI2hob9eRk6qXhJh3iciaqCS24tichPo1nrEpubalNo2YsGibcUFWlVP1YQ/640?wx_fmt=png)
![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDRPnydVP1BBvC8yYicnzwmkJsZlRTVLMqg6agcMMial4XeqLkBgHE5Iiag/640?wx_fmt=png)

##### 5\. 初始化数据库

```
mysql -uambari -pambariuse ambari;source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql
```

##### 6\. 启动ambari-server

```
ambari-server start
```

如果启动过程失败，可以去_/var/log/ambari-server/ambari-server.log_查看报错信息，一般是由于数据库配置不好导致ambari启动失败。

如果解决不了，可点击**阅读全文**进行留言，我会尽力解决。

登陆浏览器访问: http://192.168.162.41:8080 ，利用界面部署集群。

![](https://mmbiz.qpic.cn/mmbiz_png/BQQRPo0PQq53HfNpESn5H56p7KhsickEDIGVSKnS6kw6eROfhyGkfeicp3hIxgicaB5uC0A8FcF2eqH2JaaRG0bJQ/640?wx_fmt=png)

默认登陆账号/密码为：admin/admin。