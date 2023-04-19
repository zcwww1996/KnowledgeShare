[TOC]

挂载
mount -o remount,rw /dev/cdrom /mnt/cdrom
# 1. 配置yum源
## 1.1 ISO镜像文件
### 1.1.1 挂载光盘

**光盘挂载，常见于虚拟机**<br>
```shell
# mkdir /mnt/cdrom
# mount  /dev/cdrom  /mnt/cdrom
```
卸载挂载:umount /mnt/cdrom

**iso文件挂载**<br>
一般情况下，提供的均是iso文件，例如CentOS-7-x86_64-DVD-1810.iso

```bash
mount -o loop iso文件路径 /mnt/cdrom
-o loop参数表示：通过loop设备来挂载，这个参数必不可少
```

### 1.1.2 让网络yum源文件失效

```shell
cd /etc/yum.repos.d/
  rename  .repo  .repo.bak  *  #重命名所有的.repo文件
 cp  CentOS-Media.repo.bak  CentOS-Media.repo  #配置一个.repo文件
```

### 1.1.3 修改光盘yum源文件

vim CentOS-Media.repo 

```shell
[c6-media]
name=CentOS-$releasever - Media
baseurl=file:///mnt/cdrom
# 这里的地址为自己光盘挂载地址，并把不存在的地址注释掉，在行首注释
# file:///mnt/cdrom
gpgcheck=1
enabled=1
# 把原来的0改为1，让这个yum源配资文件生效
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-6
```
可使用 yum repolist 查看 当前可用yum源

```
yum clean all
yum repolist
```


想要让yum源配置永久生效：
在/etc/fstab里添加一行
`/dev/sr0 /mnt/cdrom/ iso9660 ro 0 0`
可以实现开机自动挂载

## 1.2 RPM包自行制作yum仓库
### 1.2.1 createrepo创建镜像
将下载的rpm包上传到centos服务器(10.1.88.101)上（比如/data/rpm目录下），然后进入存放rpm包的目录，执行以下命令：

```bash
cd /data/rpm

createrepo -o ./ /data/rpm
```

rpm包存放的目录就可以作为yum源目录使用了（后面说明如何使用），可以将这个目录打包后，放到其他地方也可以使用。
如上例打包 : 
```bash
cd /data
tar -zcvf rpm.tar.gz rpm/
```

【注】：如果提示找不到createrepo命令，可以使用`yum install createrepo`安装该程序。如果无法联网安装，需要自行到网上下载rpm包安装，尤其是还要下载一些依赖包，例如createrepo-0.9.9-28.el7.noarch版本就依赖于以下包：

```bash
[root@hps105 test]# yum deplist createrepo-0.9.9-28.el7.noarch
Loaded plugins: fastestmirror
Loading mirror speeds from cached hostfile
 * base: mirrors.163.com
 * epel: mirror01.idc.hinet.net
 * extras: mirrors.btte.net
 * updates: mirrors.163.com
软件包：createrepo.noarch 0.9.9-28.el7
   依赖：/bin/sh
   provider: bash.x86_64 4.2.46-33.el7
   依赖：/usr/bin/python
   provider: python.x86_64 2.7.5-86.el7
   依赖：deltarpm
   provider: deltarpm.x86_64 3.6-3.el7
   依赖：libxml2-python
   provider: libxml2-python.x86_64 2.9.1-6.el7_2.3
   依赖：pyliblzma
   provider: pyliblzma.x86_64 0.5.3-11.el7
   依赖：python >= 2.1
   provider: python.x86_64 2.7.5-86.el7
   依赖：python(abi) = 2.7
   provider: python.x86_64 2.7.5-86.el7
   依赖：python-deltarpm
   provider: python-deltarpm.x86_64 3.6-3.el7
   依赖：rpm >= 4.1.1
   provider: rpm.x86_64 4.11.3-40.el7
   依赖：rpm-python
   provider: rpm-python.x86_64 4.11.3-40.el7
   依赖：yum >= 3.4.3-4
   provider: yum.noarch 3.4.3-163.el7.centos
   依赖：yum-metadata-parser
   provider: yum-metadata-parser.x86_64 1.1.4-10.el7
```


```bash
依次安装
rpm -ivh python-deltarpm-3.6-3.el7.x86_64.rpm
rpm -ivh createrepo.noarch 0.9.9-28.el7.noarch.rpm

```
### 1.2.2 配置repo文件
（如果yum源是执行createrepo的本机(10.1.88.101)，不需要配置1，2步）

1、上传rpm.tar.gz文件到需要配置yum源的服务器</br>
假设上传到/data下

2、解压

```bash
cd /data
tar -zxvf rpm.tar.gz -C /data/rpm
```

3、配置yum

```bash
cd /etc/yum.repos.d/
mkdir bak
mv *repo bak/
vim local_rpm.repo
```

local_rpm.repo文件内容如下：

```bash
[rpm]
name=rpm_package
baseurl=file:///data/rpm
gpgcheck=0
enabled=1
```

4、然后验证即可：

```bash
yum clean all
yum makecache
```

## 1.3 yum服务器搭建
https://blog.csdn.net/huangjin0507/article/details/51351807

上述步骤及配置，都只能在本地使用离线yum仓库，如果希望其他服务器(例如10.1.88.102)也能使用该服务器（例如10.1.88.101）的离线yum仓库，就需要在该服务器上通过http服务或者是ftp服务将yum仓库共享出去，这里提供的方法是http方式。

注：这里提供的http方式需要占用80端口，其他服务器也不能将这个端口防火墙过滤掉。

1、搭建http服务器（按上例10.1.88.101，如果已搭建，可以继续下一步）

httpd服务的配置及应用：https://www.jianshu.com/p/687b915766b6

```shell
yum install -y httpd
systemctl enable httpd #httpd开机自启
systemctl start httpd #启动httpd
```

```bash
service httpd status #查看httpd的运行状态
service httpd stop #停止httpd
service httpd start #启动httpd

ps -ef | grep httpd #查看httpd的进程
httpd -v #查看已经安装的httpd的版本
```

通过yum安装的httpd服务的主配置路径通常为`/etc/httpd/conf/httpd.conf`

注：如果无法通过yum方式安装，请依次下载以下包进行安装（centos7.0+系统为例）：

```shell
 rpm -ivh apr-1.4.8-3.el7.x86_64.rpm
 rpm -ivh apr-util-1.5.2-6.el7.x86_64.rpm
 rpm -ivh httpd-tools-2.4.6-31.el7.x86_64.rpm
 rpm -ivh mailcap-2.1.41-2.el7.noarch.rpm
 rpm -ivh httpd-2.4.6-31.el7.x86_64.rpm
```

2、按照如上方式启动的httpd服务，占用端口80，默认访问路径是/var/www/html/，因此需要将上例中创建的/data/rpm、/data/iso目录做个软连接到这个目录下：

```shell
mkdir -p /var/www/html/
ln -s /data/rpm /var/www/html/rpm
ln -s /data/iso /var/www/html/iso
```


3、在其他服务器(按上例，即10.1.88.102)上配置yum源：

```shell
cd /etc/yum.repos.d/
mkdir bak
mv *repo bak/
vim http.repo
```

http.repo文件内容如下：

```shell
[http_iso] 
name=iso_105 
baseurl=http://10.1.88.101/iso 
gpgcheck=0 
enabled=1
[http_rpm] 
name=rpm_105 
baseurl=http://10.1.88.101/rpm 
gpgcheck=0 
enabled=1
```

4、然后验证即可：

```shell
yum clean all
yum makecache
```

看是否有报错。

## 1.4 查看yum安装的软件路径

1、首先安装一个redis

```
[root@linux01 ~]# yum install redis
```

2、查找redis的安装包

```
[root@linux01 ~]# rpm -qa|grep redis
redis-3.2.10-2.el7.x86_64
[root@linux01 ~]# 
```

3、查找安装包的安装路径

```
[root@linuxp01 ~]# rpm -ql redis-3.2.10-2.el7.x86_64
/etc/logrotate.d/redis
/etc/redis-sentinel.conf
/etc/redis.conf
/etc/systemd/system/redis-sentinel.service.d
/etc/systemd/system/redis-sentinel.service.d/limit.conf
/etc/systemd/system/redis.service.d
/etc/systemd/system/redis.service.d/limit.conf
/usr/bin/redis-benchmark
/usr/bin/redis-check-aof
/usr/bin/redis-check-rdb
/usr/bin/redis-cli
```

4、ok，现在就找到了！

## 1.5 yum本地安装与卸载软件


1、先查看是否有版本：

```bash
rpm -qa|grep jdk
```


2、再卸载

```bash
yum remove java-*-openjdk -y

yum remove java-1.8.0-openjdk-headless-1.8.0.91-1.b14.el6.x86_64 -y
```


3、使用yum安装本地jdk的rpm包

```bash
yum  localinstall  /opt/soft/jdk-8u45-Linux-x64.rpm  -y
```

# 2. 模板机配置
## 2.1 ip地址与hostname对应关系：
192.168.134.111 linux01

192.168.134.112 linux02

192.168.134.113 linux03

### 2.1.1 配置主机名


```bash
1) 临时修改：echo hostName > /proc/sys/kernel/hostname

2) 永久修改hostname
echo "HOSTNAME=$(hostname)" >> /etc/sysconfig/network
或 vi /etc/sysconfig/network
```

方法二：
永久性的修改主机名称，重启后能保持修改后的。

```bash
hostnamectl set-hostname xxx
```

方法三：
方法4：永久生效

通过nmtui修改，之后重启hostnamed

```bash
nmcli general hostname new-hostname
```

重启`systemctl restart systemd-hostnamed`服务

注意：如果配置正确后，主机名显示不正常。可通过以下命令修改

```shell
# sysctl kernel.hostname=linux01
```

### 2.1.2 配置映射关系，可添加别名

vi /etc/hosts

建议：可提前规划集群, 在hosts中添加各节点的ip和主机名。

规划如下：

```shell
[root@hadoop241 ~]# vim /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.134.111 linux01
192.168.134.112 linux02
192.168.134.113 linux03
```





### 2.1.3 配置ip地址

```shell
vi /etc/sysconfig/network-scripts/ifcfg-eth0
```

注意：

minimal版中，网卡默认是不开机自启动的，需要修改ONBOOT为yes

![网卡开机自启.jpg](https://i0.wp.com/i.loli.net/2019/09/18/gcGZhxkjV7tR35M.jpg "网卡开机自启")

修改后的ip配置：删除mac地址信息

![静态ip.jpg](https://i0.wp.com/i.loli.net/2019/09/18/g7v3MC68U2jn4VB.jpg "Linux静态IP")


```bash
[root@linux01 ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
TYPE=Ethernet            #网卡类型
DEVICE=eth0              #网卡接口名称
ONBOOT=yes               #系统启动时是否自动加载
BOOTPROTO=static         #启用地址协议 --static:静态协议 --bootp协议 --dhcp协议
IPADDR=192.168.134.111   #网卡IP地址
NETMASK=255.255.255.0    #网卡网络地址
GATEWAY=192.168.134.2    #网卡网关地址
DNS1=114.114.114.114     #网卡DNS地址

# 扩展配置。可省略
BROADCAST=192.168.1.255  #网卡广播地址(可省略)
USERCTL=no               #非root用户不允许控制该网络接口
IPV6INIT=yes             #是否支持IPv6
```


 <font style="background-color: yellow;color: red;">**可以直接删除映射关系配置文件，避免mac地址的冲突。**</font>


```shell
rm -rf  /etc/udev/rules.d/70-persistent-net.rules
```

**执行<font color="red">`service network restart`</font>命令，重启网络服务**

## 2.2 防火墙
### CentOS6
- 查看防火墙状态
  - service iptables status
- 开启防火墙
  - service iptables start
- 临时关闭防火墙
  - service iptables stop
- 永久关闭防火墙
  - chkconfig iptables off
- 重启防火墙
  - service iptables restart


或者

开启：/etc/init.d/iptables start

关闭：/etc/init.d/iptables stop

重启：/etc/init.d/iptablesrestart

#查看防火墙开机启动状态

chkconfig iptables -list
#开机启动

chkconfig  iptables  on
#关闭防火墙开机启动***

chkconfig iptables off
关闭防火墙的自动运行

/sbin/chkconfig --level 2345 iptables off
### CentOS7
- 查看默认防火墙状态（2选1）
  - firewall-cmd --state
  - systemctl status firewalld.service
- 启动防火墙
  - systemctl start firewalld.service
- 停止firewall
  - systemctl stop firewalld.service
- 重启防火墙（2选1）
  - firewall-cmd --reload
  - systemctl restart firewalld.service
- 禁止firewall开机启动
  - systemctl disable firewalld.service

Centos 7 firewall 命令：

```shell
查看已经开放的端口：
firewall-cmd --list-ports
开启端口
firewall-cmd --zone=public --add-port=80/tcp --permanent

命令含义：
-zone #作用域
-add-port=80/tcp #添加端口，格式为：端口/通讯协议
-permanent #永久生效，没有此参数重启后失效
```

学习阶段，建议关闭防火墙。

把该虚拟机当做模板机。用于生成快速生成其他机器。

## 2.3 scp & ssh
### 2.3.1 scp节点之间拷贝文件

命令格式：
`scp file  远程用户名@远程服务器IP:~/`
（注意：冒号和目录之间不能有空格）

**注意事项**：
1. 远程用户名@可省略，默认使用当前操作系统的用户名。
2. **如果拷贝目录，需要加-r 选项。** 
3. 使用`pwd`或者$PWD 默认到当前目录
4. ~:到当前用户的宿主目录
5. scp默认连接的远端主机22端口，使用-P（P大写）指定其他端口


使用root用户


```
#复制文件夹
scp /etc/profile root@node2:/etc
scp -r /usr/jdk1.8 node2:/usr/java

#当前目录
scp hello.log node2:'pwd'
scp hello.log node2:$PWD

#指定端口号
scp -P 16022 local_file user@host:/dir
```

**从远端主机将文件复制到另一台远端主机**

三台hdp主机，hdp01和hdp02、hdp02和hdp03网络相通但hdp01和hdp03网络不通(不要求网络通)，要求把文件从hdp01复制到hdp03上。

在hdp02上执行：

```bash
scp userA@hdp01:fileA userC@hdp03:fileC
```


**批量发送**
```shell
scp /etc/services  node2:/root/service.hard

for i in {2..9}; do scp /etc/services  node$i:/root/service.hard;done

for ((i=2;i<10;i++)) do scp /etc/services  node${i}:/root/service.hard; done
```
可以通过这种方式**修改拷贝的文件名**。


### 2.3.2 ssh切换到其他节点

1. 切换到其他节点系统

sh 到指定端口

`ssh -p port user@ip`

- port为端口号
- user为用户名
- ip为要登陆的ip

```shell
ssh -p 10022 hdp@node2 
```
退出输入exit

2. **在主节点远程命令其他节点执行命令**


```shell
#在node1机上，为node2机创建一个spark用户，命令建议使用“双引号”括起来
ssh node2 “useradd spark”

# 以root用户，让hadoop01到hadoop09打印主机名
for ((i=1;i<6;i++)) do ssh root@hadoop0${i} "echo $HOSTNAME"; done

for ((i=1;i<6;i++)) do ssh root@10.37.73.${i} "hostname"; done
```

3. **sftp远程登陆**


```bash
sftp -P 22 ftp@192.168.134.112
```


sftp默认端口号22，可以通过-P来指定要通过哪个端口号连接

> 补充：
>
> 不同服务器可能sftp版本不一样,-P 指定的不是port 而是 `sftp_server_path`，可以通过sftp --help查看指定端口参数
> 
> **可以尝试如下方式指定端口**
> ```bash
> sftp -oPort=10022 ftp@192.168.134.112
> ```


### 2.3.3 免密登录

**1、在第一台机器上生成一对钥匙，公钥和私钥**

-t rsa可省略

```shell
ssh-keygen -t rsa
```
![生成公钥与私钥.png](https://i0.wp.com/i.loli.net/2019/09/18/1GxnZORa9uMq5cf.png "生成公钥与私钥")

当前用户的宿主目录下的.ssh目录多了两个文件

![公钥与私钥.png](https://i0.wp.com/i.loli.net/2019/09/18/OvxgAKoGNz1PMX5.png "公钥与私钥")

**2、将公钥拷贝给要免密码登录的机器**

注意：主机名和ip都可以（确保配置了主机名 ip的映射）

还需要输入密码

```shell
ssh-copy-id node2
```

![拷贝公钥给其他节点.png](https://i0.wp.com/i.loli.net/2019/09/18/vA8tGYJs1P4ydo3.png "拷贝公钥到其他节点")


拷贝完成之后，会在要免密登录的机器上生成授权密码文件
![授权密码文件.png](https://i0.wp.com/i.loli.net/2019/09/18/NatpZSvAKO4erh1.png "授权密码文件")

**3、验证免密码登录**

![验证免密登陆.png](https://i0.wp.com/i.loli.net/2019/09/18/YveBjCIyJPV1zTD.png "验证免密登陆")

**4、注意**：

1. 免密码登录是<font color="red">单向</font>的
2. ssh、scp、ssh-copy-id都可以省略用户名，默认为当前主机的登陆用户
   - scp test.txt root@linux01:$PWD/test.doc
   - ssh root@linux02 "ll"
   - ssh-copy-id root@linux02
