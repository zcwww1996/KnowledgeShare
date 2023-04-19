[TOC]
# 1. date -d用法

date -d //显示字符串所指的日期与时间。字符串前后必须加上双引号

string可以为‘now' 、 ‘yesterday'、 ‘n days ago'、'tomorrow'

‘n days ago' 表示n天前的那一天

指定显示的日期格式：

==<font style="color: red;">+ 和格式之间没有空格</font>==


```shell
date <+时间日期格式>
例如：
date +"%Y-%m-%d" // 注意 ：+ 和格式之间没有空格
2016-11-30
```
## 1.1 常用命令
有可能用到的格式

```shell
%H 小时，24小时制（00~23） 
%I 小时，12小时制（01~12） 
%k 小时，24小时制（0~23） 
%l 小时，12小时制（1~12） 
%M 分钟（00~59） 
%p 显示出AM或PM 
%r 显示时间，12小时制（hh:mm:ss %p） 
%s 从1970年1月1日00:00:00到目前经历的秒数 
%S 显示秒（00~59） 
%T 显示时间，24小时制（hh:mm:ss） 
%X 显示时间的格式（%H:%M:%S） 
%Z 显示时区，日期域（CST） 
%a 星期的简称（Sun~Sat） 
%A 星期的全称（Sunday~Saturday） 
%h,%b 月的简称（Jan~Dec） 
%B 月的全称（January~December） 
%c 日期和时间（Tue Nov 20 14:12:58 2012） 
%d 一个月的第几天（01~31） 
%x,%D 日期（mm/dd/yy） 
%j 一年的第几天（001~366） 
%m 月份（01~12） 
%w 一个星期的第几天（0代表星期天） 
%W 一年的第几个星期（00~53，星期一为第一天） 
%y 年的最后两个数字（1999则是99）
```

输出当前日期

```shell
$date -d "now" +%Y-%m-%d
```

输出昨天日期

```shell
date -d "1 day ago" +"%Y-%m-%d"
date -d "yesterday" +%Y-%m-%d
2016-11-29
```

输出明天时间

```shell
date -d tomorrow +"%Y-%m-%d"
date -d +"1 day" +"%Y-%m-%d"
date -d day +"%Y-%m-%d"
2016-12-01
```

2秒后输出

```shell
date -d "2 second" +"%Y-%m-%d %H:%M.%S"
2016-11-30 10:46.04
```

时间戳类型的 输出对应的1234567890秒

```shell
date -d "1970-01-01 1234567890 seconds" +"%Y-%m-%d %H:%m:%S" 
2009-02-13 23:02:30
```

普通格式

```shell
date -d "2016-11-30" +"%Y/%m/%d %H:%M.%S" 
2016/11/30 00:00.00
```

apache格式转换：

```
date -d "Nov 30, 2016 12:00:37" +"%Y-%m-%d %H:%M.%S"
2016-11-30 12:00.37
```

格式转换后时间游走：

```
date -d "Nov 30, 2016 12:00:37 AM 2 year ago" +"%Y-%m-%d %H:%M.%S"
2014-11-30 00:00.37
```

加减操作

```
date +%Y%m%d //显示前天年月日 
date -d "+1 day" +%Y%m%d //显示前一天的日期 
date -d "-1 day" +%Y%m%d //显示后一天的日期 
date -d "-1 month" +%Y%m%d //显示上一月的日期 
date -d "+1 month" +%Y%m%d //显示下一月的日期 
date -d "-1 year" +%Y%m%d //显示前一年的日期 
date -d "+1 year" +%Y%m%d //显示下一年的日期
```

## 1.2 脚本
如果需要计算一组命令花费多少时间

```
start=$(date +"%d")//以时间戳类型保存当前时间
// 执行需要计算的命令
end=$(date +"%d")//以时间戳类型保存当前时间
difference=$(( end - start )) // 相减的结果就是命令执行完需要的时间
```

循环打印时间

```bash
#!/bin/bash

for ((i = 1; i < 5; i++)); do
        date1=$(date -d "20210101 $i days" "+%Y%m%d")
        echo "20210101 for the next $i days is $date1"
done
```

> **补充**<br>
> 20210101 3天前<br>
> date1=$(date -d "20210101 3 days ago" "+%Y%m%d")
> 
> 20210101 3天后<br>
> date2=$(date -d "20210101 3 days" "+%Y%m%d")
# 2. chrony时间同步 ★★★(推荐)

**推荐使用chrony进行时间同步，CentOS 7之类基于RHEL的操作系统上，已经默认安装有Chrony**

## 2.1 服务端
yum install chrony --RHEL7默认已安装chrony，而没有安装ntpd.

vim /etc/chrony.conf

注掉这几行

```bash
server 0.centos.pool.ntp.org
server 1.centos.pool.ntp.org
server 2.centos.pool.ntp.org
```

新增如下，采用本地主机时间

```bash
server 10.99.99.12 iburst
allow 10.99.99.0/24
local stratum 10
```

[![服务端](https://shop.io.mi-img.com/app/shop/img?id=shop_029c3cdb29644298a7342ec3e15f72ac.jpeg)](https://p.pstatp.com/origin/1372e00019372e20a244f)

[![服务端2](https://ae01.alicdn.com/kf/U0a84f45d5b5d41e09fb08eb6ed183f18i.jpg)](https://shop.io.mi-img.com/app/shop/img?id=shop_03923b8697ea3b2e54ca10642617be1e.jpeg)

> 补充chrony.conf解析
> ```bash
> # 要使用的时间同步服务器
> # Use public servers from the pool.ntp.org project.
> # Please consider joining the pool (http://www.pool.ntp.org/join.html).
> #server 0.centos.pool.ntp.org iburst
> #server 1.centos.pool.ntp.org iburst
> #server 2.centos.pool.ntp.org iburst
> #server 3.centos.pool.ntp.org iburst
> server 10.99.99.12 iburst
> # prefer表示优先 iburst 当初始同步请求时，采用突发方式接连发送8个报文，时间间隔为2秒
> 
> # 根据实际时间计算出服务器增减时间的比率，然后记录到一个文件中，在系统重启后为系统做出最佳时间补偿调整。
> # Record the rate at which the system clock gains/losses time.
> driftfile /var/lib/chrony/drift
> 
> # 如果系统时钟的偏移量大于1秒，则允许系统时钟在前三次更新中步进。
> # Allow the system clock to be stepped in the first three updates
> # if its offset is larger than 1 second.
> makestep 1.0 3
> 
> # 启用实时时钟（RTC）的内核同步。
> # Enable kernel synchronization of the real-time clock (RTC).
> rtcsync
> 
> # 通过使用 hwtimestamp 指令启用硬件时间戳
> # Enable hardware timestamping on all interfaces that support it.
> #hwtimestamp *
> 
> # Increase the minimum number of selectable sources required to adjust
> # the system clock.
> #minsources 2
> 
> 
> # 指定 NTP 客户端地址，以允许或拒绝连接到扮演时钟服务器的机器
> # 10.99.99.0/24代表10.99.99.0~10.99.99.255网段
> # Allow NTP client access from local network.
> allow 10.99.99.0/24
> 
> # 即使本机未能通过网络时间服务器同步到时间，也允许将本地时间作为标准时间授时给其它客户端
> # Serve time even if not synchronized to a time source.
> local stratum 10
> 
> # 指定包含 NTP 身份验证密钥的文件。
> # Specify file containing keys for NTP authentication.
> #keyfile /etc/chrony.keys
> 
> # 指定日志文件的目录。
> # Specify directory for log files.
> logdir /var/log/chrony
> 
> # 选择日志文件要记录的信息。
> # Select which information is logged.
> #log measurements statistics tracking
> ```

重启：
`systemctl restart chronyd.service`

## 2.2 客户端
vim /etc/chrony.conf

[![客户端](https://ae01.alicdn.com/kf/Ubf9baefc34154d80af34e2a02e6a49692.jpg)](https://pic.downk.cc/item/5f1fe82314195aa594c608a1.jpg)

重启：
`systemctl restart chronyd.service`

查看时间同步源：
**chronyc sources -v**

[![时间同步源](https://p.pstatp.com/origin/ff0300021262bb1a5acd)](https://pic.downk.cc/item/5f1ff6e814195aa594d69fac.png)

应该显示时间同步服务器的主机名和`*`号

**timedatectl status**<br>
客户端应该是两个yes

```bash
NTP enabled: yes
NTP synchronized: yes
```


## 2.3 常用命令
```bash
开机启动
systemctl enable chronyd.service

启用NTP时间同步：
timedatectl set-ntp yes

立即手工同步
chronyc -a makestep

查看时间同步源状态：
chronyc sourcestats -v

当前时间/日期/时区
timedatectl status
```


# 3. NTP时间服务器(非首选)
安装配置NTP服务端

简介

时间服务NTP：Network Time Protocol

作用：用来给其他主机提供时间同步服务，在搭建服务器集群的时候，需要保证各个节点的时间是一致的，时间服务器不失为一个好的选择。

**准备工作**

关闭防火墙、关闭selinux

系统版本：CentOS7.x，

NTP服务器IP：10.220.5.111，客户端IP：10.220.5.179

## 3.1 安装配置NTP服务器

### 3.1.1 安装ntp

```shell
[root@linux01 ~]# yum install ntp -y
```
### 3.1.2 修改ntp的配置文件
生成/etc/ntp.conf.bak文件（**文件名和{}无空格**）

[root@linux01 ~]# cp /etc/ntp.conf{,.bak}

[root@linux01 ~]# vim /etc/ntp.conf

**NTP服务器IP：10.220.5.111**
```shell
 server 210.72.145.44 prefer #中国国家授时中心服务器地址 prefer表示优先 iburst 当初始同步请求时，采用突发方式接连发送8个报文,时间间隔为2秒
 server 127.127.1.0 #无法连接授时中心时，以本机时间作为时间服务器
 fudge 127.127.1.0 startnum 10 #设置服务器层级 
 restrict 127.0.0.1 # 允许本机使用这个时间服务器
 restrict 10.220.5.0 netmask 255.255.255.0 nomodify #允许允许10.220.5.0/24网段（即所有网段，末尾ip：0-255）的所有主机使用该时间服务器校时，但是拒绝让他们修改服务器上的时间
 
 driftfile /var/lib/ntp/ #记录当前时间服务器，与上游服务器的时间差的文件
 logfile /var/log/ntp/ntp.log #指定日志文件位置，需要手动创建
```

**关于权限设定部分**

权限的设定主要以 restrict 这个参数来设定，主要的语法为：

restrict IP地址 mask 子网掩码 参数

其中 IP 可以是IP地址，也可以是 default ，default 就是指所有的IP

参数有以下几个：
- ignore　：关闭所有的 NTP 联机服务
- nomodify：客户端不能更改服务端的时间参数，但是客户端可以通过服务端进行网络校时。
- notrust ：客户端除非通过认证，否则该客户端来源将被视为不信任子网
- noquery ：客户端不能够使用 ntpq 与 ntpc 等来查询时间服务器，等于不提供 NTP 的网络校时
- notrap ：不提供 trap 这个远程事件登录(remote event logging)的功能，用于远程事件日志记录
- nopeer ：用于阻止主机尝试与服务器对等，并允许欺诈性服务器控制时钟
- kod ： 向不安全的访问者发送 Kiss-Of-Death 报文

**注意：如果参数没有设定，那就表示该 IP (或子网)没有任何限制！**

在/etc/ntp.conf文件中我们可以用restrict关键字来配置上面的要求

允许本机地址一切的操作
代码:
restrict 127.0.0.1

最后我们允许局域网内所有client连接到这台服务器同步时间.但是拒绝让他们修改服务器上的时间
代码:
restrict 10.220.5.0 mask 255.255.255.0 nomodify

### 3.1.3 创建日志文件

```shell
[root@linux01 ~]# mkdir /var/lib/ntp/
[root@linux01 ~]# touch /var/lib/ntp/ntp.log
```

### 3.1.4 启动服务

```shell
[root@linux01 ~]# systemctl start ntpd
[root@linux01 ~]# systemctl enable ntpd
```

### 3.1.5 查看状态

```shell
[root@linux01 ~]# ntpstat
synchronised to local net at stratum 6 
  time correct to within 11 ms
  polling server every 64 s
```

synchronised：表示时间同步完成（ntp可以正常工作了）

unsynchronised：表示时间同步尚未完成

**或者用`ntpq -p`查看状态**

```shell
[root@linux01 ~]# ntpq -p
     remote refid st t when poll reach delay offset jitter
==============================================================================
*LOCAL(0) .LOCL. 5 l 13 64 377 0.000 0.000 0.000
```

## 3.2 安装配置NTP客户端

### 3.2.1 安装

```
[root@Linux02 ~]# yum install ntp ntpdate -y
```

### 3.2.2 修改配置文件

**NTP服务器IP：10.220.5.179**

```
[root@Linux02 ~]# cp /etc/ntp.conf{,.bak}
[root@Linux02 ~]# vim /etc/ntp.conf
server 10.220.5.111 #设置以10.220.5.111做为本机的时间服务器
restrict 127.0.0.1
 
 logfile /var/log/ntp/ntp.log #指定日志文件位置，需要手动创建
 ```

### 3.2.3 创建日志文件

 <font style="background-color: yellow;">如未配置日志文件相关参数，跳过该步骤</font>

```
[root@Linux02 ~]# mkdir /var/log/ntp
[root@Linux02 ~]# touch /var/log/ntp/ntp.log
```

### 3.2.4 先执行一次ntpdate时间同步

```
[root@Linux02 ~]# ntpdate 10.220.5.111
```

### 3.2.5 启动ntpd

```
[root@Linux02 ~]# systemctl start ntpd
```

### 3.2.6 检查状态

```
[root@Linux02 ~]# ntpstat
unsynchronised
  time server re-starting
  polling server every 8 s
```


或者

```shell
[root@Linux02 ~]# ntpq -p
     remote refid st t when poll reach delay offset jitter
==============================================================================
 10.220.5.111 LOCAL(0) 6 u 11 64 1 0.502 0.009 0.000
```
说明：在工作中我们一般都是使用ntpdate+ntp来完成时间同步，因为单独使用ntpdate同步时间虽然简单快捷但是会导致时间不连续，而时间不连续在数据库业务中影响是很大的，单独使用ntp做时间同步时，当服务器与时间服务器相差大的时候则无法启动ntpd来同步时间。由于ntpd做时间同步时是做的顺滑同步（可以简单理解为时间走得快，以便将落后的时间赶过来），所以同步到时间服务器的的时间不是瞬间完成的，开启ntpd之后稍等三五分钟就能完成时间同步。

### 3.2.7 补充：ntpq -p命令
补充：用ntpq -p查看状态时的各种参数解释

|参数 |释义|
|---|---|
|remote|上游的时间服务器的ip或者主机名，如果是*表示本机就是做为上游服务器工作|
|refid|".LOCL."表示基于当前主机提供时间同步服务，如果是IP地址表示基于一个上游服务器提供时间同步服务。|
st|表示remote远程服务器的层级编号|
|t|“”|
|when|表示几秒之前做过一次时间同步|
|poll|表示每隔多少秒做一次时间同步|
|reach|表示向上游服务器成功请求时间同步的次数|
|delay|从本地机发送同步要求到ntp服务器的时间延迟|
|offset|主机通过NTP时钟同步与所同步时间源的时间偏移量，单位为毫秒（ms）。offset越接近于0,主机和ntp服务器的时间越接近|
|jitter|这是一个用来做统计的值. 它统计了在特定个连续的连接数里offset的分布情况。简单地说这个数值的绝对值越小，主机的时间就越精确|



## 3.3 ntp时间同步
检查ntp的版本：`ntpq -c version`

- service ntpd status  #查看ntpd服务状态
- service ntpd start   #启动ntpd服务
- service ntpd stop    #停止ntpd服务
- service ntpd restart #重启ntpd服务

**配置/etc/ntp.conf**

```
server 172.16.100.221 prefer
restrict 172.16.100.0 mask 255.255.255.0 nomodify
restrict default ignore
server 127.127.1.0
fudge 127.127.1.0 stratum 8
```


https://blog.csdn.net/zhaomax/article/details/82458283

## 3.4 解决ntp的错误 no server suitable for synchronization found 

 
当用ntpdate -d 来查询时报错：
 "no server suitable for synchronization found "
错误的原因有以下2个：
  
**1.Server dropped: Strata too high**

在ntp客户端运行ntpdate serverIP，呈现no server suitable for synchronization found的错误。 

在ntp客户端用ntpdate –d serverIP查看，发明有“Server dropped: strata too high”的错误，并且显示“stratum 16”。而正常景象下stratum这个值得局限是“0~15”。

这是因为NTP server还没有和其自身或者它的server同步上。

以下的定义是让NTP Server和其自身对峙同步，若是在/etc/ntp.conf中定义的server都不成用时，将应用local时候作为ntp办事供给给ntp客户端。 

server 127.127.1.0 

fudge 127.127.1.0 stratum 8 
 
原因分析：

在ntp server上从头启动ntp服务后，ntp server自身或者与其server的同步的须要一个时候段，这个过程可能是5分钟，在这个时候之内涵客户端运行ntpdate号令时会产生no server suitable for synchronization found的错误。 
 
 
**那么如何知道何时ntp server完成了和自身同步的过程呢？** 

在ntp server上应用命令： 


```bash
watch ntpq -p
```


呈现画面： 


```
＃ ntpq -p                                                                                                             
 Thu  Jul 10 02:28:32 2008 
     remote           refid      st t when poll reach   delay   offset jitter 
============================================================================== 
192.168.30.22   LOCAL(0)         8 u   22   64    1    2.113 179133.   0.001 
LOCAL(0)        LOCAL(0)        10 l   21   64    1    0.000   0.000  0.001
```
重视LOCAL的这个就是与自身同步的ntp server。

重视reach这个值，在启动ntp server办过后，这个值就从0开端络续增长，当增长到17的时辰，从0到17是5次的变革，每一次是poll的值的秒数，是64秒*5=320秒的时候。

若是之后从ntp客户端同步ntp server还失败的话，用ntpdate –d来查询具体错误信息，再做断定。

**错误2.Server dropped: no data**

从客户端履行netdate –d时有错误信息如下： 

..... 

28 Jul 17:42:24 ntpdate[14148]: no server suitable for synchronization found

呈现这个题目的原因可能有2： 

1。搜检ntp的版本，若是你应用的是ntp4.2（包含4.2）之后的版本，在restrict的定义中应用了notrust的话，会导致以上错误。 检查ntp的版本：ntpq -c version

2。搜检ntp server的防火墙。可能是server的防火墙拦截了upd 123端口。 ：

`service iptables stop`
来关掉iptables办过后再测验测验从ntp客户端的同步，若是成功，证实是防火墙的题目，须要更改iptables的设置。 

来源： https://blog.51cto.com/xiahongyuan/939815

# 4. crontab定时任务同步时间

ntpdate调整时间的方式就是我们所说的“**跃变**”：在获得一个时间之后，ntpdate使用set time of day设置系统时间


```bash
安装ntpdate
yum install ntpdate -y

date
查看当前时间，结果如下：Tue Mar 4 01:36:45 CST 2018

date -s 09:38:40 
设置当前时间，结果如下：Tue Mar 4 09:38:40 CST 2018
```

```bash
crontab -l

40 0 * * * /usr/sbin/ntpdate -u 210.72.145.44;/sbin/hwclock -w;

//每天0点40同步时间并写入硬件
```

若不加上-u参数,会出现以下提示：`no server suitable for synchronization found`

- u：`-u参数`可以越过防火墙与主机同步；
- 210.72.145.44：中国国家授时中心的官方服务器
- 通过`hwclock -w`将时间写入硬件中。ntpdate同步的时间并不会写入到硬件去。如果机器重启，会出现时间恢复到未同步之前。

查看硬件时间
`hwclock -show`
