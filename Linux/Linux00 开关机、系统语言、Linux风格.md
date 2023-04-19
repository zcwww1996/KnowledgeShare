[TOC]
# 关机、重启
## 1.reboot命令
reboot 通知系统重启。


```
# reboot        //重启机器

# reboot --halt //停止机器

# reboot -p     //关闭机器
```


## 2.poweroff命令

```
# poweroff          //关闭机器

# poweroff --halt   // 停止机器

# poweroff --reboot // 重启机器
```


## 3.halt命令
halt通知硬件来停止所有的 CPU 功能，但是仍然保持通电。你可以用它使系统处于低层维护状态。

用halt命令来关机时，实际调用的是shutdown -h。

halt 执行时将杀死应用进程，执行sync系统调用文件系统写操作完成后就会停止内核。

注意在有些情况会它会完全关闭系统。

halt 参数说明:
- [-n] 防止sync系统调用，它用在用fsck修补根分区之后，以阻止内核用老版本的超级块〔superblock〕覆盖修补过的超级块。 
- [-w] 并不是真正的重启或关机，只是写wtmp〔/var/log/wtmp〕纪录。 
- [-d] 不写wtmp纪录〔已包含在选项[-n]中〕。 
- [-f] 没有调用shutdown而强制关机或重启。 
- [-i] 关机〔或重启〕前关掉所有的网络接口。 
- [-p] 该选项为缺省选项。就是关机时调用poweroff。
下面是 halt 命令示例：


```
# halt           //停止机器

# halt -p        //关闭机器

# halt --reboot  // 重启机器
```


## 4.shutdown命令
shutdown会给系统计划一个时间关机。它可以被用于停止、关机、重启机器。

你可以指定一个时间字符串（通常是now或者用hh:mm指定小时/分钟）作为第一个参数。额外地，你也可以设置一个广播信息在系统关闭前发送给所有已登录的用户。

重要：如果使用了时间参数，系统关机前 5 分钟，会创建/run/nologin文件。以确保没有人可以再登录。

shutdown 参数说明:
- [-t] 在改变到其它runlevel之前，告诉init多久以后关机。 
- [-r] 重启计算器。 
- [-k] 并不真正关机，只是送警告信号给每位登录者〔login〕。 
- [-h] 关机后关闭电源〔halt〕。 
- [-n] 不用init而是自己来关机。不鼓励使用这个选项，而且该选项所产生的后果往往不总是你所预期得到的。 
- [-c] cancel current process取消目前正在执行的关机程序。所以这个选项当然没有时间参数，但是可以输入一个用来解释的讯息，而这信息将会送到每位使用者。 
- [-f] 在重启计算器〔reboot〕时忽略fsck。
- [-F] 在重启计算器〔reboot〕时强迫fsck。 
- [-time] 设定关机〔shutdown〕前的时间。 

```
# shutdown -c        //取消正在关机状态

# shutdown now       //立即关机

# shutdown -h 10     //10分钟后 停止机器

# shutdown 13:20     //13:20时关机

# shutdown -p now    // 关闭机器

# shutdown -H now    // 停止机器 

# shutdown -r now    //立刻重启(root用户使用)
# shutdown -r 10     //过10分钟自动重启(root用户使用)
# shutdown -r 09:35  // 在 09:35am 重启机器
```


如果是通过shutdown命令设置关机的话，可以用shutdown -c 命令取消重启

**那么为什么说shutdown命令是安全地将系统关机呢**？

实际中有些用户会使用直接断掉电源的方式来关闭linux，这是十分危险的。因为linux与windows不同，其后台运行着许多进程，所以强制关机可能会导致进程的数据丢失使系统处于不稳定的状态。甚至在有的系统中会损坏硬件设备。而在系统关机前使用shutdown命令，系统管理员会通知所有登录的用户系统将要关闭。并且login指令会被冻结，即新的用户不能再登录。直接关机或者延迟一定的时间才关机都是可能的，还有可能是重启。这是由所有进程〔process〕都会收到系统所送达的信号〔signal〕决定的。

shutdown执行它的工作是发送信号〔signal〕给init程序，要求它改变 runlevel。runlevel 0被用来停机〔halt〕，runlevel 6是用来重新激活〔reboot〕系统，而runlevel 1则是被用来让系统进入管理工作可以进行的状态，这是预设的。假定没有-h也没有-r参数给shutdown。要想了解在停机〔halt〕或者重新开机〔reboot〕过程中做了哪些动作？你可以在这个文件/etc/inittab里看到这些runlevels相关的资料。

## 5.init
init是所有进程的祖先，他是Linux系统操作中不可缺少的程序之一。它的进程号始终为1，所以发送TERM信号给init会终止所有的用户进程，守护进程等。shutdown就是使用这种机制。init定义了8个运行级别(runlevel)，init 0为关机，init 6为重启。

# 修改Linux系统语言
## 1、首先查看当前系统的语言
### 1.1、查看当前操作系统的语言

echo $LANG

中文：zh_CN.UTF-8

英文：en_US.UTF-8

### 1.2 查看支持的编码

在linux系统的终端中输入命令：locale，打印系统当前的编码信息

```
[root@Hadoop01 ~]# locale
LANG=en_US.UTF-8
LC_CTYPE="en_US.UTF-8"
LC_NUMERIC="en_US.UTF-8"
LC_TIME="en_US.UTF-8"
LC_COLLATE="en_US.UTF-8"
LC_MONETARY="en_US.UTF-8"
LC_MESSAGES="en_US.UTF-8"
LC_PAPER="en_US.UTF-8"
LC_NAME="en_US.UTF-8"
LC_ADDRESS="en_US.UTF-8"
LC_TELEPHONE="en_US.UTF-8"
LC_MEASUREMENT="en_US.UTF-8"
LC_IDENTIFICATION="en_US.UTF-8"
LC_ALL=
```


## 2、安装中文编码
### 2.1 查linux的支持系统编码

检查linux的系统编码，确定系统是否支持中文。在linux系统的终端中输入命令：locale -a，就会看到打印出的系统支持的编码信息。
### 2.2 系统安装中文语言包


```
[root@Hadoop01 ~]# yum -y groupinstall chinese-support
```

## 3、修改系统默认语言
### 3.1 临时更改

临时更改默认语言，当前立即生效 重启失效
export LANG=en_US.UTF-8

### 3.2 永久生效

修改配置文件

```
centos7/rhel7之前版本：vim /etc/sysconfig/i18n
centos7/rhel7版本：vim /etc/locale.conf
修改：LANG="en_US.UTF-8"
```


### 3.3 使其立即生效

```
source /etc/sysconfig/i18n
source /etc/locale.conf
```

# System V风格、BSD风格
**Linux命令参数前加-、--和不加-的区别**
## 1.单- 和双- -的区别

1.1 参数前单-表示后面参数为字符形式，如tar -zxvf；

1.2 参数前加- - 表示后面参数为单词，如rm - -help；
## 2.加-和不加-的区别

在这里插入代码片加不加-执行命令的结果是相同的，区别主要涉及Linux风格，System V和BSD。

2.1 参数前不加-属于System V风格；

2.2 加-属于BSD风格。
## 两种风格的区别
系统启动过程中 kernel 最后一步调用的是 init 程序，init 程序的执行有两种风格，即 System V 和 BSD。

System V 风格中 init 调用 /etc/inittab，BSD 风格调用 /etc/rc，它们的目的相同，都是根据 runlevel 执行一系列的程序。

# 查看开机启动项

**servicename 就是你要查的服务名**

```bash
# 查看某服务当前状态
service servicename status

# 查看某服务是否开机自动启动
chkconfig --list servicename
# 查看所有开机自启服务
chkconfig --list
```
如果service和chkconfig 找不到，可以试试`/sbin/service`和`/sbin/chkconfig`<br>
如果用ubuntu好像是要用`/etc/init.d/servicename status`查看当前状态

# 查看Linux系统版本


## `lsb_release -a`

适用于所有的linux，包括Redhat、SuSE、Debian等发行版，但是在debian下要安装lsb

```
[root@hdp03 ~]# lsb_release -a
LSB Version:    :core-4.1-amd64:core-4.1-noarch:cxx-4.1-amd64:cxx-4.1-noarch:desktop-4.1-amd64:desktop-4.1-noarch:languages-4.1-amd64:languages-4.1-noarch:printing-4.1-amd64:printing-4.1-noarch
Distributor ID: CentOS
Description:    CentOS Linux release 7.6.1810 (Core) 
Release:        7.6.1810
Codename:       Core
```

## `cat /etc/redhat-release`
适用于Redhat、CentOS系统

```bash
[root@hdp03 ~]# cat /etc/redhat-release
CentOS Linux release 7.6.1810 (Core)
```
