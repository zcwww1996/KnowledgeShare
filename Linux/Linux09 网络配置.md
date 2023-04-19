[TOC]

# 1. 网络通信命令
## 1.1 ping
命令路径：/bin/ping

执行权限：所有用户

作用：测试网络的连通性

语法：**ping [选项] IP地址**
- -c 指定发送次数   

ping 命令使用的是icmp协议，不占用端口
eg:
```shell
ping -c 3 127.0.0.1
```
## 1.2 ifconfig
命令路径：/sbin/ifconfig

**执行权限：root**

作用：查看和设置网卡网络配置

语法：

ifconfig [-a] [网卡设备标识]  
- -a：显示所有网卡信息

ifconfig [网卡设备标识] IP地址 修改ip地址

## 1.3 netstat
命令路径：/bin/netstat

执行权限：所有用户

作用：主要用于检测主机的网络配置和状况
- -a  all显示所有连接和监听端口
- -t (tcp)仅显示tcp相关选项
- -u (udp)仅显示udp相关选项
- -n 使用数字方式显示地址和端口号
- -l （listening）  显示监控中的服务器的socket
- -p或--programs：显示正在使用Socket的程序识别码和程序名称
eg:
```shell
# netstat -tlnu      查看本机监听的端口
tcp 0 0 0.0.0.0:111 0.0.0.0:* LISTEN
协议  待收数据包  待发送数据包  本地ip地址：端口 远程IP地址：端口
# netstat -au 列出所有 udp 端口
# nestat -at 列出所有tcp端口
# netstat -an  查看本机所有的网络连接
# netstat -anp  |grep   端口号 查看XX端口号使用情况
# netstat -nultp （此处不用加端口号）该命令是查看当前所有已经使用的端口情况
```
如下，我以3306为例，netstat  -anp  |grep  3306（此处备注下，我是以普通用户)

[![3306端口是否被占用.png](https://www.helloimg.com/images/2022/07/08/ZTlnnt.webp)](https://i0.wp.com/i.loli.net/2019/09/16/ByIDxCdL5O2K7vQ.png)

图1中主要看监控状态为LISTEN表示已经被占用，最后一列显示被服务mysqld占用，查看具体端口号，只要有如图这一行就表示被占用了。

netstat  -anp  |grep 82

查看82端口的使用情况，如图2：

[![82端口使用情况.png](https://ae01.alicdn.com/kf/Hc249dbd96efc448fadab569ba068c385t.jpg "82端口使用情况") ](https://imageproxy.pimg.tw/resize?url=https://i0.wp.com/i.loli.net/2019/09/16/WE6lc7wiT1n94RK.png)

可以看出并没有LISTEN那一行，所以就表示没有被占用。此处注意，图中显示的LISTENING并不表示端口被占用，不要和LISTEN混淆哦，查看具体端口时候，**必须要看到tcp，端口号，LISTEN那一行，才表示端口被占用了**

**统计机器中网络连接各个状态个数**

```bash
netstat -an | awk '/^tcp/ {++S[$NF]}  END {for (a in S) print a,S[a]} '
```

**查看连接某服务端口最多的的IP地址**
```bash
netstat -ant|awk '{print $5}'|grep "172.16*"|sort -nr|uniq -c
```

# 2. 防火墙
## 2.1 Centos 7 firewall
### 2.1.1 放行ssh的端口
1、	配置防火墙，首先放行ssh的端口，默认为22端口（本例为52222端口）

```bash
vim /etc/firewalld/zones/public.xml

添加：
<port protocol="tcp" port="52222"/>
```

[![ssh端口.jpg](https://www.helloimg.com/images/2022/07/08/ZTlmvb.png)](https://ae01.alicdn.com/kf/U33ee35373ed9421c80afed597c4bf76bd.jpg)

2、	开启防火墙

```bash
systemctl start firewalld
查看开放端口：firewall-cmd --list-ports
```


### 2.1.2 firewalld的基本使用

启动： systemctl start firewalld

关闭： systemctl stop firewalld

查看状态： systemctl status firewalld

开机禁用 ： systemctl disable firewalld

开机启用 ：systemctl enable firewalld

### 2.1.3 systemctl
systemctl是CentOS7的服务管理工具中主要的工具，它融合之前service和chkconfig的功能于一体。

- 启动一个服务：`systemctl start firewalld.service`
- 关闭一个服务：`systemctl stop firewalld.service`
- 重启一个服务：`systemctl restart firewalld.service`
- 显示一个服务的状态：`systemctl status firewalld.service`
- 在开机时启用一个服务：`systemctl enable firewalld.service`
- 在开机时禁用一个服务：`systemctl disable firewalld.service`
- 查看服务是否开机启动：`systemctl is-enabled firewalld.service`
- 查看已启动的服务列表：`systemctl list-unit-files|grep enabled`
- 查看启动失败的服务列表：`systemctl --failed`

### 2.1.4 配置firewalld-cmd

- 查看版本： firewall-cmd --version

- 查看帮助： firewall-cmd --help

- 显示状态： firewall-cmd --state

- 查看所有打开的端口： firewall-cmd --zone=public--list-ports

- 更新防火墙规则： firewall-cmd --reload

- 查看区域信息: firewall-cmd --get-active-zones

- 查看指定接口所属区域： firewall-cmd --get-zone-of-interface=eth0

- 拒绝所有包：firewall-cmd --panic-on

- 取消拒绝状态： firewall-cmd --panic-off

- 查看是否拒绝： firewall-cmd --query-panic

那怎么开启一个端口呢

添加

```bash
firewall-cmd--zone=public--add-port=80/tcp--permanent  （--permanent永久生效，没有此参数重启后失效）
```

重新载入


```
firewall-cmd--reload
```


查看

```
firewall-cmd--zone=public--query-port=80/tcp
```


删除

```
firewall-cmd--zone=public--remove-port=80/tcp--permanent
```

调整默认策略（默认拒绝所有访问，改成允许所有访问）：

```
firewall-cmd --permanent --zone=public --set-target=ACCEPT
firewall-cmd--reload
```



对某个IP开放多个端口：

```
firewall-cmd --permanent --add-rich-rule="rule family="ipv4" source address="10.159.60.29" port protocol="tcp" port="1:65535" accept"
firewall-cmd--reload
```


## 2.2 Centos 6 iptables
### 2.2.1 iptables的基本使用


```bash
启动： service iptables start

关闭：service iptables stop

查看状态：service iptables status

开机禁用 ： chkconfig iptables off

开机启用 ：chkconfig iptables on
```


### 2.2.2 开放指定的端口

`-A`和`-I`参数分别为添加到规则末尾和规则最前面。

```
`#允许本地回环接口(即运行本机访问本机)
iptables -A INPUT -i lo -j ACCEPT
# 允许已建立的或相关连的通行
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT
#允许所有本机向外的访问  
iptables -P INPUT ACCEPT
iptables -A OUTPUT -j ACCEPT
# 允许访问22端口
iptables -A INPUT -p tcp --dport 22 -j ACCEPT  
iptables -A INPUT -p tcp -s 10.159.1.0/24 --dport 22 -j ACCEPT     
注：-s后可以跟IP段或指定IP地址
#允许访问80端口
iptables -A INPUT -p tcp --dport 80 -j ACCEPT
#允许FTP服务的21和20端口
iptables -A INPUT -p tcp --dport 21 -j ACCEPT
iptables -A INPUT -p tcp --dport 20 -j ACCEPT
#如果有其他端口的话，规则也类似，稍微修改上述语句就行
#允许ping
iptables -A INPUT -p icmp -m icmp --icmp-type 8 -j ACCEPT
#禁止其他未允许的规则访问
iptables -A INPUT -j REJECT  #（注意：如果22端口未加入允许规则，SSH链接会直接断开。）
iptables -A FORWARD -j REJECT` 
```

### 2.2.3 屏蔽IP

```
iptables -I INPUT -s 123.45.6.7 -j DROP

iptables -I INPUT -s 123.0.0.0/8 -j DROP

iptables -I INPUT -s 124.45.0.0/16 -j DROP

iptables -I INPUT -s 123.45.6.0/24 -j DROP
```

### 2.2.4 查看已添加的iptables的规则

`iptables -L -n`

_N：只显示IP地址和端口号，不将IP解析为域名_

### 2.2.5 删除已添加的iptables的规则

将所有iptables以序号标记显示，执行：

`iptables -L -n --line-numbers`

比如要删除INPUT里序号为8的规则，执行：

`iptables -D INPUT 8`

### 2.2.6 编辑配置文件，添加iptables规则

iptables的配置文件为/ etc / sysconfig / iptables

编辑配置文件：

`vi /etc/sysconfig/iptables`

文件中的配置规则与通过的iptables命令配置，语法相似：

如，通过iptables的命令配置，允许访问80端口：

`iptables -A INPUT -p tcp --dport 80 -j ACCEPT`

那么，在文件中配置，只需要去掉句首的iptables，添加如下内容：

`-A INPUT -p tcp --dport 80 -j ACCEPT`

保存退出。

### 2.2.7 两种方式添加规则

`iptables -A` 和`iptables -I`

iptables -A 添加的规则是添加在最后面。如针对INPUT链增加一条规则，接收从eth0口进入且源地址为192.168.0.0/16网段发往本机的数据。


```
iptables -A INPUT -i eth0 -s 192.168.0.0/16 -j ACCEPT
```


iptables -I 添加的规则默认添加至第一条。

如果要指定插入规则的位置，则使用iptables -I 时指定位置序号即可。

1. 删除规则

如果删除指定则，使用iptables -D命令，命令后可接序号。效果请对比上图。

或iptables -D 接详细定义；

如果想把所有规则都清除掉，可使用iptables -F。

2. 备份iptabes rules

使用iptables-save命令，如：

iptables-save > /etc/sysconfig/iptables.save

3. 恢复iptables rules

使用iptables命令，如：

iptables-restore < /etc/sysconfig/iptables.save

4. iptables 配置保存

以上做的配置修改，在设备重启后，配置将丢失。可使用service iptables save进行保存。

service iptables save

5. 重启iptables的服务使其生效：

```
service iptables save 添加规则后保存重启生效。

service iptables restart
```


### 2.2.8 后记


关于更多的iptables的使用方法可以执行：

`iptables --help`

# 3. Linux端口应用
## 3.1 开启端口

（以80端口为例）
### 方法一：
`/sbin/iptables -I INPUT -p tcp --dport 80 -j ACCEPT` 写入修改

/etc/init.d/iptables save 保存修改

service iptables restart 重启防火墙，修改生效

### 方法二：
vi /etc/sysconfig/iptables

打开配置文件加入如下语句:


```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j ACCEPT
```

重启防火墙，修改完成


## 3.2 关闭端口

### 方法一：
/sbin/iptables -I INPUT -p tcp --dport 80 -j DROP 写入修改

/etc/init.d/iptables save 保存修改

service iptables restart 重启防火墙，修改生效
### 方法二：
vi /etc/sysconfig/iptables

打开配置文件加入如下语句:


```
-A INPUT -p tcp -m state --state NEW -m tcp --dport 80 -j DROP
```

重启防火墙，修改完成

## 3.3 查看端口状态

`/etc/init.d/iptables status`

## 3.4 端口转发
### 3.4.1 功能场景
192.168.134.111(linux01)和192.168.134.112(linux02)在内网可以连接，但是只有192.168.134.111这台机器有另外一个网口配置了外网可访问的地址，外网不能直接访问和192.168.134.112这台机器。

我客户端要直接访问和192.168.134.112的数据库或者网页，怎么办？

### 3.4.2 具体需求
192.168.134.112:8080是tomcat默认页面，外部无法访问，通过linux01这台机器的外网网口转发。

本机(192.168.134.111)监听来自对本机全部网口9999端口发起连接的请求，然后把数据全部转发到192.168.134.112的8080端口去

```bash
ssh -C -f -N -g -L 9999:192.168.134.112:8080 root@192.168.134.111
```

> 补充: ssh的三个强大的端口转发命令：
> 
> **转发到远端：ssh -C -f -N -g -L 本地端口:目标IP:目标端口 用户名@目标IP**
> 
> **转发到本地：ssh -C -f -N -g –R 本地端口:目标IP:目标端口 用户名@目标IP**
> 
> - -C：压缩数据传输。
> - -f ：后台认证用户/密码，通常和-N连用，不用登录到远程主机。
> - -N ：不执行脚本或命令，通常与-f连用。
> - -g ：在-L/-R/-D参数中，允许远程主机连接到建立的转发的端口，如果不加这个参数，只允许本地主机建立连接。
> - -L 本地端口:目标IP:目标端口<br/>
将**本地机(客户机)的某个端口转发到远端**指定机器的指定端口
> - -R本地端口:目标IP:目标端口<br/>
将**远程主机(服务器)的某个端口转发到本地**端指定机器的指定端口
> - -p ：被登录的ssd服务器的sshd服务端口。
> - -D port 指定一个本地机器 “动态的'’ 应用程序端口转发

### 3.4.4 场景演示
#### 3.4.4.1 网页连接前
12机器：

```bash
[root@linux01 ~]# ssh -C -f -N -g -L 9999:192.168.134.112:8080 root@192.168.134.111
root@192.168.134.111's password:
[root@linux01 ~]# netstat -an |grep 9999
tcp        0      0 0.0.0.0:9999                0.0.0.0:*                   LISTEN
tcp        0      0 :::9999                     :::*                        LISTEN
```

以上会发现，linux01这台机器起了9999端口，侦听外面发起的连接请求

那么在其他机器发起对9999连接后，可以看到会话建立了。

linux02机器:

```bash
[root@linux02 ~]# netstat -an |grep 8080
tcp        0      0 :::8080                     :::*                        LISTEN
```
[![tomcat网页访问](https://sc01.alicdn.com/kf/H06d54e8e06c64f5aab4632b613d97236r.jpg)](https://ww1.yunjiexi.club/2020/04/28/JjLNz.jpg)

实际上，浏览器中地址栏显示是linux01的9999端口,但是内容实质是linux02那边开启的8080服务。


#### 3.4.4.2 网页连接后
linux01机器：

```bash
[root@linux01 ~]# netstat -an |grep 9999
tcp        0      0 0.0.0.0:9999                0.0.0.0:*                   LISTEN      
tcp        0      0 192.168.134.111:9999        192.168.134.1:15515         ESTABLISHED 
tcp        0      0 192.168.134.111:9999        192.168.134.1:15516         ESTABLISHED 
tcp        0      0 192.168.134.111:9999        192.168.134.1:15514         ESTABLISHED 
tcp        0      0 :::9999                     :::*                        LISTEN
```


> wireshark抓包：linux01上抓eth0的包
> 
> ```bash
> tcpdump -i eth0 host 192.168.134.112 -w /root/9999.cap
> ```

### 3.4.5 关停端口转发
用完，直接停掉进程即可
```bash
[root@linux01 ~]# netstat -an |grep 9999
tcp        0      0 0.0.0.0:9999                0.0.0.0:*                   LISTEN
tcp        0      0 :::9999                     :::*                        LISTEN
[root@linux01 ~]# netstat -tlnp |grep 9999
tcp        0      0 0.0.0.0:9999                0.0.0.0:*                   LISTEN      1142/ssh            
tcp        0      0 :::9999                     :::*                        LISTEN      1142/ssh

[root@linux01 ~]# ps -f 1142
UID         PID   PPID  C STIME TTY      STAT   TIME CMD
root       1142      1  0 15:04 ?        Ss     0:00 ssh -C -f -N -g -L 9999:192.168.134.112:8080 root@192.168.134.111
[root@linux01 ~]# kill -9 1142
```

### 3.4.5 问题
1、**有些机器SSH设置不允许端口转发，需要设置**

```bash
vi /etc/ssh/sshd_config
gatewayports yes
```
