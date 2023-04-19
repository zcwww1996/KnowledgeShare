[TOC]
# 1. 增大文件描述符nofile和用户最大进程nproc
（查看当前的lsof |wc -l）

**1. 调整Linux的最大文件打开数和进程数**。

```bash
vi /etc/security/limits.conf
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
```

**2. RHEL6下引入了90-nproc.conf配置文件**

```bash
vi /etc/security/limits.d/90-nproc.conf
* soft nproc 65535
```


**3. pam_limits.so 文件被加入到启动文件**

```bash
vi /etc/pam.d/login
session required /lib/security/pam_limits.so
session required pam_limits.so
```

**4. 重启**

```bash
reboot
```

---
只针对当前session会话,快速设置

```
[hdfs@hadoop01 ~]$  ulimit -u
1024
[hdfs@hadoop01 ~]$  ulimit -u 65535
[hdfs@hadoop01 ~]$  ulimit -u 65535

-n <文件数目> 　指定同一时间最多可打开的文件数。
-u <进程数目> 　用户最多可启动的进程数目。
```

# 2. 网络
(两个网络相关的参数可以影响Hadoop的性能。net.core.somaxconn Linux内核设置能够支持NameNode和JobTracker的大量爆发性的HTTP请求,设置txqueuelen到4096及以上能够更好地适应在Hadoop集群中的突发流量)

## 2.1 net.core.somaxconn
net.core.somaxconn是listen()的默认参数,挂起请求的最大数量.默认是128.对繁忙的服务器,增加该值有助于网络性能，当前已经被调整到32768

```bash
more /etc/sysctl.conf |grep net.core.somaxconn
sysctl -w net.core.somaxconn=32768 

echo net.core.somaxconn=32768 >> /etc/sysctl.conf
执行命令
sysctl -p
```


## 2.2 txqueuelen
设置txqueuelen到4096及以上能够更好地适应在Hadoop集群中的突发流量， txqueuelen代表用来传输数据的缓冲区的储存长度，

通过下面的命令可以对该参数进行设置为4096。

```bash
[hdfs@hadoop01 conf]# ifconfig
eth0      Link encap:Ethernet  HWaddr 00:16:3E:02:00:2B  
 inet addr:xx.xxx.xx.x  Bcast:xx.xxx.xx.xxx  Mask:255.255.255.0
 UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
 RX packets:55072078 errors:0 dropped:0 overruns:0 frame:0
 TX packets:33328184 errors:0 dropped:0 overruns:0 carrier:0
 collisions:0 txqueuelen:1000 
 RX bytes:23381014283 (21.7 GiB)  TX bytes:4464530654 (4.1 GiB)
```
发现当前的eth0的txqueuelen值为1000,设置为4096

```bash
[hdfs@hadoop01 conf]# ifconfig eth0 txqueuelen 4096
```

# 3. 关闭swap分区

```bash
more /etc/sysctl.conf | vm.swappiness
echo vm.swappiness = 0 >> /etc/sysctl.conf
```

# 4. 设置合理的预读取缓冲区大小
预读取缓冲区(readahead buffer)

调整linux文件系统中预读缓冲区地大小，可以明显提高顺序读文件的性能。默认buffer大小为256 sectors，可以增大为1024或者2408 sectors（注意，并不是越大越好）。可使用blockdev命令进行调整。

```bash
[hdfs@hadoop01 conf ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        40G  7.1G   31G  19% /
tmpfs           3.9G     0  3.9G   0% /dev/shm
/dev/vdb1       197G   36G  152G  19% /data/01

[hdfs@hadoop01 conf ~]# blockdev --report
RO    RA   SSZ   BSZ   StartSec            Size   Device
rw   256   512  4096          0     42949672960   /dev/vda
rw   256   512  4096       2048     42947575808   /dev/vda1
rw   256   512  4096          0    214748364800   /dev/vdb
rw   256   512  4096         63    214748029440   /dev/vdb1
```

**修改/dev/vdb1的readahead buffer**,因为hadoop的dfs nn等等文件夹是在这个目录下

```bash
[hdfs@hadoop01 conf ~]# blockdev --setra 1024 /dev/vdb1
```

     
# 5. I/O调度器选择
(一般不调整,只会在mapreduce中调整)

主流的Linux发行版自带了很多可供选择的I/O调度器。在数据密集型应用中，不同的I/O调度器性能表现差别较大，
管理员可根据自己的应用特点启用最合适的I/O调度器

# 6. vm.overcommit_memory设置
进程通常调用malloc()函数来分配内存，内存决定是否有足够的可用内存，并允许或拒绝内存分配的请求。Linux支持超量分配内存，以允许分配比可用RAM加上交换内存的请求。

vm.overcommit_memory参数有三种可能的配置：
1. 表示检查是否有足够的内存可用，如果是，允许分配；如果内存不够，拒绝该请求，并返回一个错误给应用程序。
2. 表示根据vm.overcommit_ratio定义的值，允许分配超出物理内存加上交换内存的请求。vm.overcommit_ratio参数是一个百分比，加上内存量决定内存可以超量分配多少内存。

例如，vm.overcommit_ratio值为50，而内存有1GB，那么这意味着在内存分配请求失败前，加上交换内存，内存将允许高达1.5GB的内存分配请求。
3. 表示内核总是返回true。

除了以上几个常见的Linux内核调优方法外，还有一些其他的方法，管理员可根据需要进行适当调整。
         
**【查看当前值】**

```bash
sysctl -n vm.overcommit_memory
```

**【永久性修改内核参数】**

在/etc/sysctl.conf文件里面加入或者直接删除也可以，因为它缺省值就是0

```bash
vm.overcommit_memory = 0
```

运行使之生效

```bash
sysctl -p
```

# 7. Transparent Huge Page
已启用“透明大页面”，它可能会导致重大的性能问题。版本为“CentOS release 6.3 (Final)”且版本为“2.6.32-279.el6.x86_64”的 Kernel 已将 enabled 设置为“[always] never”，并将 defrag 设置为“[always] never”。

请运行`echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag`以禁用此设置，然后将同一命令添加到一个 init 脚本中，如 /etc/rc.local，这样当系统重启时就会设置它。

或者，升级到 RHEL 6.4 或以上版本，它们不存在此错误。将会影响到RHEL 6.4以下主机。

```bash
hdfs@hadoop01 conf ~]# cat /sys/kernel/mm/redhat_transparent_hugepage/defrag
[always] madvise never
[hdfs@hadoop01 conf ~]# echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag 
[hdfs@hadoop01 conf ~]# echo 'echo never > /sys/kernel/mm/redhat_transparent_hugepage/defrag' >> /etc/rc.local
```
