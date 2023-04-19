[TOC]

参考：https://www.cnblogs.com/shook/p/12409759.html

# 1. 本次安装简单介绍
## 1.1 Ambari
Ambari是一种基于Web的工具，支持Apache Hadoop集群的创建 、管理和监控。

Ambari已支持大多数Hadoop组件，包括HDFS、MapReduce、Hive、Pig、 Hbase、Zookeeper、Sqoop和Hcatalog等。Apache Ambari 支持HDFS、MapReduce、Hive、Pig、Hbase、Zookeepr、Sqoop和Hcatalog等的集中管理。也是5个顶级hadoop管理工具之一。

Ambari 自身也是一个分布式架构的软件，主要由两部分组成：Ambari Server 和 Ambari Agent。简单来说，用户通过 Ambari Server 通知 Ambari Agent 安装对应的软件；Agent 会定时地发送各个机器每个软件模块的状态给 Ambari Server，最终这些状态信息会呈现在 Ambari 的 GUI，方便用户了解到集群的各种状态，并进行相应的维护。

## 1.2 HDP

HDP是hortonworks的软件栈，里面包含了hadoop生态系统的所有软件项目，比如HBase,Zookeeper,Hive,Pig等等。

## 1.3 HDP-UTILS
HDP-UTILS是工具类库。

# 2. Ambari搭建前环境准备
## 2.1 版本介绍
截止到2020.03.03，Ambari的最新版本为2.7.5，HDP的最新版本为3.1.5

通过 [https://supportmatrix.hortonworks.com/](https://supportmatrix.hortonworks.com/) 可以查询Ambari和HDP各个版本支持情况

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-1024x530.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-1024x530.png)


我这里选择的是2.7.4版本，所以使用的HDP对应版本为3.1.4

## 2.2 搭建环境
### 2.2.1 所用到的环境列表
| 环境 | 版本 |
|---|---|
| Linux | Centos7物理机\*4（英文系统） |
| Ambari | 2.7.4 |
| HDP | 3.1.4 |
| HDP-UTILS | 1.1.0.2 |
| MySQL-MariaDB/ MySQL | 10.4.12/5.7 |
| OracleJDK8 | 1.8.0_201 |
| Nginx | 1.17.8(已有环境，选用，可不用) |

### 2.2.2 环境下载
Ambari在线安装特别慢，所以使用离线安装，建议使用迅雷下载

|名称|地址|
|:---|:---|
|ambari|http://public-repo-1.hortonworks.com/ambari/centos7/2.x/updates/2.7.4.0/ambari-2.7.4.0-centos7.tar.gz|
|HDP|http://public-repo-1.hortonworks.com/HDP/centos7/3.x/updates/3.1.4.0/HDP-3.1.4.0-centos7-rpm.tar.gz|
|HDP-UTILS| 	http://public-repo-1.hortonworks.com/HDP-UTILS-1.1.0.22/repos/centos7/HDP-UTILS-1.1.0.22-centos7.tar.gz|

JDK8环境
|名称|地址|
|:---|:---|
|OracleJDK|	https://www.oracle.com/java/technologies/javase-jdk8-downloads.html|

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-1-1024x546.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-1-1024x546.png)

## 2.4 集群节点划分

| 节点名称 | 局域网ip地址 | 服务 | 内存 | 硬盘 |
|:---|:---|:---|:---|:---|
| master（主节点） | 192.168.105.137 | Ambari/HDP<br>Ambari Server<br>MariaDB /MySql | 4G | 450G |
| slave1（子节点） | 192.168.105.191 | Compute node | 4G | 450G |
| slave2（子节点） | 192.168.105.192 | Compute node | 4G | 450G |
| slave3（子节点） | 192.168.105.193 | Compute node | 4G | 450G |

## 2.5 修改网络配置（所有节点）
用putty或者xshell连接执行

```bash
cd /etc/sysconfig/network-scripts
```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-3.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-3.png)

  

找到ifcfg-en开头，后面的数字由每台机器生成各有不同，直接vi编辑即可

修改前：

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-4.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-4.png)

修改后：

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-5.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-5.png)


```bash
TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=static
IPADDR=192.168.105.137
NETMASK=255.255.255.0
GATEWAY=192.168.105.1
DNS1=114.114.114.114
DNS2=8.8.8.8
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=stable-privacy
NAME=eno1
UUID=6c053b64-6581-4665-b3fc-c22f79c58848
DEVICE=eno1
ONBOOT=yes
```


重启网络服务：**service network start**

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-6.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-6.png)

ping一下局域网其他机器 通了即可

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-7.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-7.png)

# 3 系统环境配置
## 3.1 安装JDK（所有节点）
Linux自带的jdk或者是通过yum安装的jdk都是openjdk

最好是使用开源的oracle jdk，缺失部分功能，如果直接安装oracle的jdk，第三方的依赖包不会安装，我们这里是通过yum安装openjdk，并同时安装了第三方依赖包，然后卸载openjdk，通过自己来安装oracle的jdk，就能解决依赖问题。

### 3.1.1 安装openJDK

```bash
[root@master ~] yum -y install java
[root@master ~] java -version
openjdk version "1.8.0_212"
OpenJDK Runtime Environment (build 1.8.0_212-c061)
OpenJDK 64-Bit Server VM (build 25.212-c061, mixed mode)
[root@master ~] rpm -qa|grep java
javapackages-tools-3.4.1-11.el7.noarch
java-1.8.0-openjdk-1.8.0.212-0.c061.el7_7.x86_64
python-javapackages-3.4.1-11.el7_7.noarch
tzdata-java-2018d-1.el7_7.noarch
java-1.8.0-openjdk-headless-1.8.0.212-0.c061.el7_7.x86_64
```

### 3.1.2 卸载OpenJDK

```bash
[root@master ~] rpm -e --nodeps java-1.8.0-openjdk-headless-1.8.0.212-0.c061.el7_7.x86_64
[root@master ~] rpm -e --nodeps java-1.8.0-openjdk-1.8.0.212-0.c061.el7_7.x86_64

```

安装和卸载的时候注意一下open jdk的版本号

### 3.1.3 安装OracleJDK

将下载好的OracleJDk安装包（rpm格式）上传到linux
```bash
[root@master ~] rpm -ivh jdk-8u201-linux-x64.rpm 
准备中...                          ################################# [100%]
正在升级/安装...
   1:jdk1.8-2000:1.8.0_201-fcs        ################################# [100%]
Unpacking JAR files...
    tools.jar...
    plugin.jar...
    javaws.jar...
    deploy.jar...
    rt.jar...
    jsse.jar...
    charsets.jar...
    localedata.jar...
[root@master ~] java -version
java version "1.8.0_201"
Java(TM) SE Runtime Environment (build 1.8.0_201-b09)
Java HotSpot(TM) 64-Bit Server VM (build 25.201-b09, mixed mode)
[root@master ~] javac -version
javac 1.8.0_201
```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-9.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-9.png)

  
验证安装成功

## 3.2 修改节点主机名称（所有节点）

```
[root@master ~] hostnamectl set-hostname master
[root@master ~] hostname
master
```

### 3.2.1 修改/etc/hosts文件（所有节点）
方便通过名称来查找相应节点

```
[root@master ~] vi /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1             localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.105.137 master
192.168.105.191 slave1
192.168.105.192 slave2
192.168.105.193 slave3
```

### 3.2.2 修改/etc/sysconfig/network（所有节点）
各节点改成相对应的节点名即可

```bash
[root@master ~] vi /etc/sysconfig/network
# Created by anaconda
NETWORKING=yes
HOSTNAME=master
```

接下来通过测试ping各个节点名称是否调通

```
[root@master ~] ping  -C 3 slave1
PING slave1 (192.168.105.191) 56(84) bytes of data.
64 bytes from slave1 (192.168.105.191): icmp_seq=1 ttl=64 time=0.320 ms
64 bytes from slave1 (192.168.105.191): icmp_seq=2 ttl=64 time=0.282 ms
64 bytes from slave1 (192.168.105.191): icmp_seq=3 ttl=64 time=0.278 ms
--- slave1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 6999ms
rtt min/avg/max/mdev = 0.269/0.282/0.320/0.018 ms
```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-10.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-10.png)

## 3.3 跟换所有节点为阿里yum源
阿里巴巴开发者社区url： [https://developer.aliyun.com/mirror/](https://developer.aliyun.com/mirror/)

选择Centos

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-11-1024x523.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-11-1024x523.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-12.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-12.png)

可直接运行如下

```bash
[root@master ~] mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
[root@master ~] curl -o /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo
[root@master ~] yum makecache

```

## 3.4 同步实际NTP
### 3.4.1 安装ntp服务（所有节点）

```bash
[root@master ~] yum -y install ntp
```

启动服务，查看状态并设置开机自启

```
[root@master ~] systemctl start ntpd.service
 [root@master ~] systemctl status ntpd.service
● ntpd.service - Network Time Service
   Loaded: loaded (/usr/lib/systemd/system/ntpd.service; disabled; vendor preset: disabled)
   Active: active (running) since Mon 2020-03-02 15:41:28 CST; 23h ago
 Main PID: 3909 (ntpd)
   CGroup: /system.slice/ntpd.service
           └─3909 /usr/sbin/ntpd -u ntp:ntp -g

Mar 02 15:41:28 master ntpd[3909]: Listen normally on 4 lo ::1 UDP 123
Mar 02 15:41:28 master ntpd[3909]: Listen normally on 5 eno1 fe80::7b94:c6e6:5673:c105 UDP 123
Mar 02 15:41:28 master ntpd[3909]: Listening on routing socket on fd #22 for interface updates
Mar 02 15:41:28 master systemd[1]: Started Network Time Service.
Mar 02 15:41:28 master ntpd[3909]: 0.0.0.0 c016 06 restart
Mar 02 15:41:28 master ntpd[3909]: 0.0.0.0 c012 02 freq_set kernel 0.000 PPM
Mar 02 15:41:28 master ntpd[3909]: 0.0.0.0 c011 01 freq_not_set
Mar 02 15:41:35 master ntpd[3909]: 0.0.0.0 c614 04 freq_mode
Mar 02 15:58:21 master ntpd[3909]: 0.0.0.0 0612 02 freq_set kernel 8.697 PPM
Mar 02 15:58:21 master ntpd[3909]: 0.0.0.0 0615 05 clock_sync
[root@master ~] systemctl enable ntpd.service

```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-13.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-13.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-14.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-14.png)

## 3.5 关闭防火墙（所有节点）
**查看防火墙状态**

```bash
systemctl status firewalld.service
```

**关闭防火墙**

```bash
systemctl stop firewalld.service
```

**设置开机不启动**

```bash
systemctl disable firewalld.service
```

**查看是否成功**

```
[root@master ~] systemctl is-enabled firewalld.service
disabled
```

## 3.6 关闭Selinux和THP（所有节点）
### 3.6.1 关闭Selinux
查看Selinux状态

```bash
sestatus
```

关闭Selinux，提示没有vim用yum装一个或者用vi

```bash
[root@master ~] vim /etc/sysconfig/selinux
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#     enforcing - SELinux security policy is enforced.
#     permissive - SELinux prints warnings instead of enforcing.
#     disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of three two values:
#     targeted - Targeted processes are protected,
#     minimum - Modification of targeted policy. Only selected processes are protected.
#     mls - Multi Level Security protection.
SELINUXTYPE=targeted
```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-15.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-15.png)

### 3.6.2 关闭THP
查看状态

```bash
[root@yum ~] cat /sys/kernel/mm/transparent_hugepage/defrag 
[always] madvise never
[root@yum ~] cat /sys/kernel/mm/transparent_hugepage/enabled 
[always] madvise never
```

关闭THP并给予文件权限

```
[root@yum ~] vim /etc/rc.d/rc.local
#!/bin/bash
# THIS FILE IS ADDED FOR COMPATIBILITY PURPOSES
#
# It is highly advisable to create own systemd services or udev rules
# to run scripts during boot instead of using this file.
#
# In contrast to previous versions due to parallel execution during boot
# this script will NOT be run after all other services.
#
# Please note that you must run 'chmod +x /etc/rc.d/rc.local' to ensure
# that this script will be executed during boot.

touch /var/lock/subsys/local

if test -f /sys/kernel/mm/transparent_hugepage/enabled; then
        echo never > /sys/kernel/mm/transparent_hugepage/enabled
fi
if test -f /sys/kernel/mm/transparent_hugepage/defrag; then
        echo never > /sys/kernel/mm/transparent_hugepage/defrag
fi
[root@yum ~] chmod +x /etc/rc.d/rc.local
```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-16.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-16.png)

## 3.7 修改文件打开最大限制（所有节点）
设置并查看

```bash
[root@master ~] vim /etc/security/limits.conf 
# End of file 
* soft nofile 65536 
* hard nofile 65536 
* soft nproc 131072 
* hard nproc 131072
 
[root@master ~] ulimit -Sn 
[root@master ~] ulimit -Hn  

```
### 3.8 SSH 无密码登陆（主节点）
回车通过，输入密码等确认通过即可

```bash
[root@master ~] ssh-keygen -t rsa
[root@master ~] ssh-copy-id slave1
root@slave1's password: 
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'slave1'"
and check to make sure that only the key(s) you wanted were added.
[root@master ~] ssh-copy-id slave2
Are you sure you want to continue connecting (yes/no)? yes
root@slave2's password: 
Number of key(s) added: 1
[root@master ~] ssh-copy-id slave3
Are you sure you want to continue connecting (yes/no)? yes
root@slave3's password: 
Number of key(s) added: 1
[root@master ~] ssh-copy-id master
Are you sure you want to continue connecting (yes/no)? yes
root@master's password: 
Number of key(s) added: 1
Now try logging into the machine, with:   "ssh 'master'"
and check to make sure that only the key(s) you wanted were added
```

测试是否实现无密码登录 ，无输入密码即可通过

```bash
[root@master ~] ssh slave1 date ;ssh slave2 date;ssh slave3 date;ssh master date;
Tue Mar  3 16:32:07 CST 2020
Tue Mar  3 16:32:07 CST 2020
Tue Mar  3 16:32:07 CST 2020
Tue Mar  3 16:32:08 CST 2020

```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-17.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-17.png)

将刚刚创建的秘钥拷出来，后面ambari安装的时候需要上传这个秘钥。创建秘钥是在隐藏文件夹/root/.ssh/下面的，所以需要先把秘钥拷贝到可见区域，然后拷贝到本机上。

```bash
[root@master ~] cd /root/.ssh/
[root@master .ssh] ls
authorized_keys  id_rsa  id_rsa.pub  known_hosts
[root@master .ssh] cp id_rsa /root/
[root@master .ssh] ls /root/
ambari                               
ambari-2.7.4.0-centos7.tar.gz        id_rsa                              
anaconda-ks.cfg      install.sh  jdk-8u201-linux-x64.rpm

```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-18.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-18.png)

到这里我们先reboot重启一下

# 4 实现离线安装，跟换yum源
## 4.1 文件目录展示
由于我这里主机已经安装了Nginx，所以文件目录的方式我这里写了两种展示目录的方式 ，4.1.1和4.1.2选择一种即可

### 4.1.1 http服务方式

```bash
[root@master ~] yum -y install httpd
[root@master ~] service httpd restart
Redirecting to /bin/systemctl restart httpd.service
[root@master ~] chkconfig httpd on
```

安装完成后，会生成 /var/www/html目录（相当于Tomcat的webapps目录）

将之前下的Ambari、HDP、HDP-UTILS三个包放到 /var/www/html 下

```bash
[root@master ~] cd /var/www/html/
[root@master html] mkdir ambari
拷贝文件到ambari下面
[root@master html] cd ambari/
[root@master ambari] ls
ambari-2.7.4.0-centos7.tar.gz   HDP-3.1.4.0-centos7-rpm.tar.gz  HDP-UTILS-1.1.0.22-centos7.tar.gz
[root@master ambari] tar -zxvf ambari-2.7.4.0-centos7.tar.gz
[root@master ambari] tar -zxvf HDP-3.1.4.0-centos7-rpm.tar.gz 
[root@master ambari] mkdir HDP-UTILS
[root@master ambari] tar -zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz -C HDP-UTILS
[root@master ambari] rm -rf ambari--2.7.4.0-centos7.tar.gz HDP-2.6.3.0-centos7-rpm.tar.gz HDP-UTILS-1.1.0.22-centos7.tar.gz 
[root@master ambari] ls
ambari  HDP  HDP-UTILS
```

4.1.2 nginx服务方式
找到nginx的配置， 在nginx server中的location中增加：

```bash
 server {
        listen 8001;
        location / {
            root /www//html/ambari/;
            autoindex on;
            autoindex_localtime on;
            autoindex_exact_size off;
        }
     }
```

参数说明：  
- **root /data/image/**<br>
你需要开启浏览的目录，放访问http://IP 时候显示的就是/data/image目录下的内容
- **autoindex_localtime on;**<br>
默认为off，显示的文件时间为GMT时间。  
改为on后，显示的文件时间为文件的服务器时间
- **autoindex_exact_size off;**<br>
默认为on，显示出文件的确切大小，单位是bytes。  
改为off后，显示出文件的大概大小，单位是kB或者MB或者GB  
- **listen 8001;**<br>
访问端口号，

```bash
[root@master ~] cd /var/www/
[root@master html] mkdir ambari
拷贝文件到ambari下面
[root@master html] cd ambari/
[root@master ambari] ls
ambari-2.7.4.0-centos7.tar.gz   HDP-3.1.4.0-centos7-rpm.tar.gz  HDP-UTILS-1.1.0.22-centos7.tar.gz
[root@master ambari] tar -zxvf ambari-2.7.4.0-centos7.tar.gz
[root@master ambari] tar -zxvf HDP-3.1.4.0-centos7-rpm.tar.gz 
[root@master ambari] mkdir HDP-UTILS
[root@master ambari] tar -zxvf HDP-UTILS-1.1.0.22-centos7.tar.gz -C HDP-UTILS
[root@master ambari] rm -rf ambari--2.7.4.0-centos7.tar.gz HDP-2.6.3.0-centos7-rpm.tar.gz HDP-UTILS-1.1.0.22-centos7.tar.gz 
[root@master ambari] ls
ambari  HDP  HDP-UTILS

```

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-19.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-19.png)

访问 http://192.168.105.137/ambari/ 看是否能访问

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-21.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-21.png)

## 4.2 制作本地yum源
### 4.2.1 安装本地源制作相关工具（主节点）

```bash
[root@master ambari] yum install yum-utils createrepo yum-plugin-priorities -y
[root@master ambari]  createrepo  ./
```
### 4.2.2 修改文件里面的源地址（主节点）
注意文件路径，以自己为准

```bash
[root@master ambari] vi ambari/centos7/2.7.4.0-118/ambari.repo
#VERSION_NUMBER=2.7.4.0-118
[ambari-2.7.4.0]
#json.url = http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json
name=ambari Version - ambari-2.7.4.0
baseurl=http://192.168.105.137:8001/ambari/centos7/2.7.4.0-118
gpgcheck=1
gpgkey=http://192.168.105.137:8001/ambari/centos7/2.7.4.0-118/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
[root@master ambari] cp ambari/centos7/2.7.4.0-118/ambari.repo /etc/yum.repos.d/
[root@master ambari] vi HDP/centos7/3.1.4.0-315/hdp.repo
#VERSION_NUMBER=3.1.4.0-315
[HDP-3.1.4.0]
name=HDP Version - HDP-3.1.4.0
baseurl=http://192.168.105.137:8001/HDP/centos7/3.1.4.0-315
gpgcheck=1
gpgkey=http://192.168.105.137:8001/HDP/centos7/3.1.4.0-315/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1


[HDP-UTILS-1.1.0.22]
name=HDP-UTILS Version - HDP-UTILS-1.1.0.22
baseurl=http://192.168.105.137:8001/HDP-UTILS/HDP-UTILS/centos7/1.1.0.22
gpgcheck=1
gpgkey=http://192.168.105.137:8001/HDP-UTILS/HDP-UTILS/centos7/1.1.0.22/RPM-GPG-KEY/RPM-GPG-KEY-Jenkins
enabled=1
priority=1
[root@master ambari] cp HDP/centos7/3.1.4.0-315/hdp.repo /etc/yum.repos.d/

```

以上就已经创建好了

使用yum的命令清一下缓存

```bash
[root@master ambari] yum clean all
[root@master ambari] yum makecache
[root@master ambari] yum repolist
```

### 4.2.3 将repo文件发送到从节点（主节点）

```bash
[root@master ambari]#cd /etc/yum.repos.d
[root@master yum.repos.d] scp ambari.repo slave1:/etc/yum.repos.d/
ambari.repo                                                                                        100%  274    24.8KB/s   00:00    
[root@master yum.repos.d] scp ambari.repo slave2:/etc/yum.repos.d/
ambari.repo                                                                                        100%  274   294.9KB/s   00:00    
[root@master yum.repos.d] scp ambari.repo slave3:/etc/yum.repos.d/
ambari.repo                                                                                        100%  274   541.7KB/s   00:00    
[root@master yum.repos.d] scp hdp.repo slave1:/etc/yum.repos.d/
hdp.repo                                                                                           100%  482     603.6KB/s   00:00    
[root@master yum.repos.d] scp hdp.repo slave2:/etc/yum.repos.d/
hdp.repo                                                                                           100%  482   541.9KB/s   00:00    
[root@master yum.repos.d] scp hdp.repo slave3:/etc/yum.repos.d/
hdp.repo

```
# 5 安装ambari-server
这里介绍两种模式，一种是默认postgresql数据库的安装方式，这种不推荐生产环境使用，但是在安装中不会出现问题，还有就是mysql方式，这里我们用到的是mysql，大家根据自身选一种即可

无论是用哪种，首先都要安装ambari-server

```bash
[root@master yum.repos.d] yum -y install ambari-server
```

## 5.1默认数据库PostgreSQL安装方式（不推荐）（主节点）

```bash
[root@master yum.repos.d] ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'disabled'
Customize user account for ambari-server daemon [y/n] (n)? n
Adjusting ambari-server permissions and ownership...
Checking firewall status...
Checking JDK...
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
Enter choice (1): 2
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/java/jdk1.8.0_202
Validating JDK on Ambari Server...done.
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? n
Configuring database...
Default properties detected. Using built-in database.
Configuring ambari database...
Checking PostgreSQL...
Running initdb: This may take up to a minute.
Initializing database ... OK
About to start PostgreSQL
Configuring local database...
Configuring PostgreSQL...
Restarting PostgreSQL
Creating schema and user...
done.
Creating tables...
done.
Extracting system views...
ambari-admin-2.7.4.0\-118.jar
...........
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
```

启动ambari

```bash
[root@master yum.repos.d] ambari-server start
Using python  /usr/bin/python
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start...........................................
Server started listening on 8080
DB configs consistency check: no errors and warnings were found.
Ambari Server 'start' completed successfully.
```

成功启动后在浏览器输入Ambari地址：  
[http://192.168.105.137:8080](http://192.168.12.101:8080/) 即可看到

## 5.2 MySql安装方式（主节点）

### 5.2.1 安装MySql

```bash
[root@master ~] wget http://dev.mysql.com/get/mysql-community-release-el7-5.noarch.rpm
[root@master ~] rpm -ivh mysql-community-release-el7-5.noarch.rpm
[root@master ~] yum install mysql-community-serve
```

### 5.2.2 启动mysql，设置开机启动

```bash
[root@master ~] service mysqld start
[root@master ~] vi /etc/rc.local
#添加service mysqld start
```

### 5.2.3 登录进mysql，初始化设置root密码

```bash
[root@master ~] mysql -uroot 
设置登录密码
mysql> set password for 'root'@'localhost' = password('yourPassword');
添加远程登录用户
mysql> grant all privileges on \*.\* to 'root'@'%' identified by 'yourPassword';

远程登录
mysql -uroot -h ip(远程的ip地址) -p 
``` 

### 5.2.4登录mysql，执行以下的语句

```sql
CREATE DATABASE ambari;  
use ambari;  
CREATE USER 'ambari'@'%' IDENTIFIED BY 'ambari';  
GRANT ALL PRIVILEGES ON \*.\* TO 'ambari'@'%';  
CREATE USER 'ambari'@'localhost' IDENTIFIED BY 'ambar';  
GRANT ALL PRIVILEGES ON \*.\* TO 'ambari'@'localhost';  
CREATE USER 'ambari'@'master' IDENTIFIED BY 'ambari';  
GRANT ALL PRIVILEGES ON \*.\* TO 'ambari'@'master';  
FLUSH PRIVILEGES;  
source /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql  
show tables;  
use mysql;  
select Host User Password from user where user\='ambari';  
CREATE DATABASE hive;  
use hive;  
CREATE USER 'hive'@'%' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON \*.\* TO 'hive'@'%';  
CREATE USER 'hive'@'localhost' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON \*.\* TO 'hive'@'localhost';  
CREATE USER 'hive'@'master' IDENTIFIED BY 'hive';  
GRANT ALL PRIVILEGES ON \*.\* TO 'hive'@'master';  
FLUSH PRIVILEGES;  
CREATE DATABASE oozie;  
use oozie;  
CREATE USER 'oozie'@'%' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON \*.\* TO 'oozie'@'%';  
CREATE USER 'oozie'@'localhost' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON \*.\* TO 'oozie'@'localhost';  
CREATE USER 'oozie'@'master' IDENTIFIED BY 'oozie';  
GRANT ALL PRIVILEGES ON \*.\* TO 'oozie'@'master';  
FLUSH PRIVILEGES;
```

### 5.2.5 建立mysql与ambari-server的jdbc连接

```bash
[root@master yum.repos.d] yum install mysql-connector-java
[root@master yum.repos.d] ls /usr/share/java
[root@master yum.repos.d] cp /usr/share/java/mysql-connector-java.jar /var/lib/ambari-server/resources/mysql-jdbc-driver.jar
[root@master yum.repos.d] vi /etc/ambari-server/conf/ambari.properties
添加server.jdbc.driver.path=/usr/share/java/mysql\-connector-java.jar
```

### 5.2.6设置ambari-server

```bash
[root@master yum.repos.d] ambari-server setup
Using python  /usr/bin/python
Setup ambari-server
Checking SELinux...
SELinux status is 'enabled'
SELinux mode is 'permissive'
WARNING: SELinux is set to 'permissive' mode and temporarily disabled.
OK to continue [y/n] (y)? y
Customize user account for ambari-server daemon [y/n] (n)? y
Enter user account for ambari-server daemon (root):ambari #ambari-server 账号。如果直接回车就是默认选择root用户
如果输入已经创建的用户就会显示：
Adjusting ambari-server permissions and ownership...
Checking firewall status...
WARNING: iptables is running. Confirm the necessary Ambari ports are accessible. Refer to the Ambari documentation for more details on ports.
OK to continue [y/n] (y)? y
Checking JDK... #设置JDK。输入：2
[1] Oracle JDK 1.8 + Java Cryptography Extension (JCE) Policy Files 8
[2] Custom JDK
Enter choice (1): 2 #输入你自己的jdk位置/usr/java/jdk1.8.0_201\-amd64
WARNING: JDK must be installed on all hosts and JAVA_HOME must be valid on all hosts.
WARNING: JCE Policy files are required for configuring Kerberos security. If you plan to use Kerberos,please make sure JCE Unlimited Strength Jurisdiction Policy Files are valid on all hosts.
Path to JAVA_HOME: /usr/java/jdk1.8.0_201\-amd64
Validating JDK on Ambari Server...done.
Check JDK version for Ambari Server...
JDK version found: 8
Minimum JDK version is 8 for Ambari. Skipping to setup different JDK for Ambari Server.
Checking GPL software agreement...
GPL License for LZO: https://www.gnu.org/licenses/old-licenses/gpl-2.0.en.html
Enable Ambari Server to download and install GPL Licensed LZO packages [y/n] (n)? n
Completing setup...
Configuring database...
Enter advanced database configuration [y/n] (n)? y   #数据库配置。选择：y
Configuring database...
Choose one of the following options:
[1] - PostgreSQL (Embedded)
[2] - Oracle
[3] - MySQL / MariaDB
[4] - PostgreSQL
[5] - Microsoft SQL Server (Tech Preview)
[6] - SQL Anywhere
[7] - BDB
Enter choice (1): 3  #选择mysql数据库类型。输入：3
 设置数据库的具体配置信息，根据实际情况输入，如果和括号内相同，则可以直接回车。
Hostname (localhost): 
Port (3306): 
Database name (ambari): 
Username (ambari): ambari
Enter Database Password (bigdata): ambari
Re-enter password: ambari
Configuring ambari database...
Configuring remote database connection properties...
WARNING: Before starting Ambari Server, you must run the following DDL directly from the database shell to create the schema: /var/lib/ambari-server/resources/Ambari-DDL-MySQL-CREATE.sql
Proceed with configuring remote database connection properties [y/n] (y)? y
Extracting system views...
ambari-admin-2.7.4.0.118.jar
....
Ambari repo file contains latest json url http://public-repo-1.hortonworks.com/HDP/hdp_urlinfo.json, updating stacks repoinfos with it...
Adjusting ambari-server permissions and ownership...
Ambari Server 'setup' completed successfully.
[root@master yum.repos.d] ambari-server start
Using python  /usr/bin/python
Starting ambari-server
Ambari Server running with administrator privileges.
Organizing resource files at /var/lib/ambari-server/resources...
Ambari database consistency check started...
Server PID at: /var/run/ambari-server/ambari-server.pid
Server out at: /var/log/ambari-server/ambari-server.out
Server log at: /var/log/ambari-server/ambari-server.log
Waiting for server start...........................................
Server started listening on 8080
DB configs consistency check: no errors and warnings were found.
Ambari Server 'start' completed successfully.
```

如果启动失败 建议查看  
日志在/var/log/ambari-server/ambari-server.log里面  
重置 ambari-server

```bash
[root@master ~] ambari-server stop
[root@master ~] ambari-server reset
[root@master ~] ambari-server setup
```

如果选择的是mysql方式，就需要先执行上面的语句，然后手动将mysql里面创建的数据库进行删除

```sql
[root@master ~] mysql -uroot -p
mysql> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| ambari             |
| hive               |
| oozie              |
| performance_schema |
+--------------------+
rows in set (0.00 sec)
mysql> drop database ambari;
mysql> drop database hive;
mysql> drop database oozie;
```

如果在安装的过程中出现了错误，又想重新安装，可以在ambari-server开启的情况下，执行下面的语句来移除已安装的包，然后再通过不同的情况选择上面两种方式的一种对ambari-server进行重置

```python
python /usr/lib/python2.6/site-packages/ambari_agent/HostCleanup.py --silent
```

# 6. 安装配置部署HDP集群
## 6.1 登录过程

如果你以上安装成功<br>
输入主机ip：8080则会看到如下界面<br>
账户：admin 密码：admin

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-22-1024x402.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-22-1024x402.png)


## 5.2 安装向导

登录后点击进入流程

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-25-1024x530.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-25-1024x530.png)

### 6.2.1 配置及群名字

我这用到的是hadoop

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-26-1024x447.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-26-1024x447.png)

### 6.2.2 选择版本并修改为本地源地址

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-27-1024x535.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-27-1024x535.png)


放一张机翻的配置

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-28-1024x506.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-28-1024x506.png)

ambari确认过了就next继续了

### 6.2.3 安装配置

上传之前存好的秘钥文件`id_rsa`

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-29-1024x547.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-29-1024x547.png)

机翻页面

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-30-1024x590.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-30-1024x590.png)

### 6.2.4 确认安装ambari的agent

确认安装ambari的agent，并检查出现的问题，这个地方经常出现错误

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-31-1024x488.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-31-1024x488.png)

如图就出现了错误，点击Failed的查看错误日志

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-32-1024x467.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-32-1024x467.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-33-1024x527.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-33-1024x527.png)

像我这里的错误是ambari的8040等端口无法访问的问题，我放开了8000-9000的端口就可以了，我之前也遇到很多其他的问题，具体问题具体分析，多查谷歌，百度，国外的网站更容易解决问题

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-34-1024x464.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-34-1024x464.png)

机翻

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-36-1024x443.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-36-1024x443.png)

检查主机可能会发现之前漏下的问题，比如说我这里防火墙没关他就会出现提示

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-37-1024x498.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-37-1024x498.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-38-1024x470.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-38-1024x470.png)

检查无误，NEXT→通过即可

如果这个步骤失败了错误，记得多看日志，多找问题，如果还不行的话，回档咯

```
[root@master ~] ambari-server stop    #停止命令
[root@master ~] ambari-server reset   #重置命令
[root@master ~] ambari-server setup   #重新设置 
[root@master ~] ambari-server start   #启动命令
```

### 6.2.5 大数据服务组件安装

勾选你所需要的

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-39-1024x532.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-39-1024x532.png)

### 6.2.6 节点分配

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-40-1024x533.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-40-1024x533.png)

### 6.2.7 分配主从

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-41-1024x494.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-41-1024x494.png)

### 6.2.8 安装配置

hive和oozie的数据库用户密码填上之前创建好的

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-42-1024x476.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-42-1024x476.png)

如果安装了hive，ooize等，需要修改成我们本地建好的库，jdbc-mysql也要配置好

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-47-1024x510.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-47-1024x510.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-48-1024x492.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-48-1024x492.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-43-1024x498.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-43-1024x498.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-44-1024x493.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-44-1024x493.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-45-1024x480.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-45-1024x480.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-46-1024x480.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-46-1024x480.png)

### 6.2.9 概况部署

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-49-1024x478.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-49-1024x478.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-50-1024x486.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-50-1024x486.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-51-1024x472.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-51-1024x472.png)

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-52-1024x472.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-52-1024x472.png)

警告这里我这就忽略掉了，后期我们再修复

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-53-1024x471.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-53-1024x471.png)

哈哈，这红的心慌，把他们变绿吧

[![](http://www.shookm.com/wp-content/uploads/2020/03/image-54-1024x522.png)](http://www.shookm.com/wp-content/uploads/2020/03/image-54-1024x522.png)