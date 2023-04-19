[TOC]
# 1. 安装一个HDFS集群
 
# 1.1  准备工作：
规划：

要有一台机器安装namenode

要有N台datanode服务器
 
 
准备4台虚拟机：
1. 在vmware中创建一个虚拟机
2. 安装linux系统
3. 安装jdk
4. 主机名、ip地址、hosts映射
5. 安装ssh工具
6. 关闭防火墙
 
 
关闭配置好的这一台虚拟，克隆出另外3台      
克隆出来的机器需要：
1. 修改主机名
2. 修改ip地址
3. 修改物理网卡删掉eth0，改eth1为eth0，

## 1.2 安装HDFS软件
Hdfs/YARN是两个分布式服务系统，mapreduce是运算程序，这些软件通通包含在HADOOP安装包中。那么，我们在安装时，其实对每一台机器都是安装一个整体的HADOOP软件，只是将来会在不同的机器上启动不同的程序。
### 1.2.1 上传安装包
hadoop安装压缩包tar.gz到hdp26-01上，解压到/root/apps/,
`alt+p`开启sftp，拖动Hadoop安装包，上传到/root

### 1.2.2. 认识hadoop软件的目录结构

![clip_image002a0f9ca62-796a-498d-afd5-58186724b131.jpg](https://i0.wp.com/i.loli.net/2019/11/15/Nr1jdXGVei3Sfwh.jpg 'hadoop目录结构')

### 1.2.3 修改配置文件
Hadoop安装文件下的hadoop-2.8.1/etc/hadoop/下面有hadoop-env.sh、core-site.xml、hdfs-site.xml三个配置文件

#### hadoop-env.sh

- 配置一个JAVA_HOME目录即可

```shell
export JAVA_HOME=/usr/local/jdk1.8.0_60
```

#### core-site.xml

- 最核心配置：HDFS的URI
- 参数名： fs.defaultFS
- 参数值： hdfs://hdp26-01:9000/
- 
**单台namenode默认使用9000端口，高可用使用8020端口**

```shell
<configuration>
 <property>
   <name>fs.defaultFS</name>
   <value>hdfs://linux01:9000/</value>
 </property>
</configuration>
```


**<font style="background-color: yellow;color: red;">知识补充</font>**

> URI:全球统一资源定位，用于描述一个“资源”（一个网页/一个文件/一个服务）的访问地址；
> 
> 举例：我有一个数据库，别人想访问，可以这样指定：
> 
>     jdbc:mysql://192.168.33.44:3306/db1
> 
> jdbc:mysql: 资源的类型或者通信协议
> 
> 192.168.33.44:3306: 服务提供者的主机名和端口号
> 
> /db1: 资源体系中的某一个具体资源（库）
> 
>  
> 
> 我有一个web系统，别人想访问，可以这样指定：
> 
> http://www.ganhoo.top:80/index.php
> 
> http:  资源的访问协议
> 
> www.ganhoo.top:80  服务提供者的主机名和端口号
> 
> /index.php  资源体系中的某一个具体资源（页面）
> 
> 我要访问本地磁盘文件系统中的文件
> 
>     file://d:/aa/bb/cls.avi
>     file:///usr/local/..

#### hdfs-site.xml
核心配置：

1.datanode服务软件在存储文件块时，放在服务器的磁盘的哪个目录

- 参数名：dfs.datanode.data.dir
- 参数值：/root/hdp-data/data/ 
 
2.namenode 服务软件在记录文件位置信息数据时，放在服务器的哪个磁盘目录
- 参数名：dfs.namenode.name.dir
- 参数值：/root/hdp-data/name/


```xml
<configuration>
 
 <property>
   <name>dfs.namenode.name.dir</name>
   <value>/root/hdp-data/name/</value>
 </property>
 
 <property>
   <name>dfs.datanode.data.dir</name>
   <value>/root/hdp-data/data/</value>
 </property>
 
 <property>
   <name>dfs.namenode.secondary.http-address</name>
   <value>linux02:50090</value>
 </property>
 
</configuration>
```

### 1.2.3 拷贝安装包到其他机器
在linux01上：

```shell
scp -r /usr/local/hadoop-2.8.1 hdp26-02:/usr/local/
scp -r /usr/local/hadoop-2.8.1 hdp26-03:/usr/local/
scp -r /usr/local/hadoop-2.8.1 hdp26-04:/usr/local/
```

### 1.2.4 手动启动HDFS系统

**<font style="background-color: yellow;color: red;">知识补充</font>**

> 为了能够在任意地方执行hadoop的脚本程序，可以将hadoop的安装目录及脚本所在目录配置到环境变量中：
> 
>  
> 
> 在之前的配置文件中添加如下内容：
> 
> vi /etc/profile
> 
>     export HADOOP_HOME=/usr/local/hadoop-2.8.1
>     export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin 
> 
> 然后
> 
> source /etc/profile

#### 格式化namenode

首先，需要给namenode机器生成一个初始化的元数据存储目录

```shell
hadoop namenode -format
```
![namenode格式化.jpg](https://i0.wp.com/i.loli.net/2019/11/15/6GypJte9TciU8qg.jpg "namenode格式化")

#### 启动namenode
在哪里启动namenode：

在你配的fs.defaultFS=hdfs://linux01:9000中的主机上启namenode

**启动命令**：

```shell
hadoop-daemon.sh start namenode
```

**启动完后检查**:

```shell
[root@liunx01]# jps
2112 NameNode
[root@liunx01]# netstat -nltp | grep 50070
```

使用浏览器访问linux01的50070端口，能打开一个namenode状态信息的网页

#### 启动datanode

**DATANODE的存储目录不需要事先初始化，datanode软件在启动时如果发现这个存储目录不存在，则会自动创建**

在哪里启动datanode：

在你想让它成为datanode服务器的机器上启：

（linux01，linux02，linux03，linux04）

**启动命令**：

```shell
hadoop-daemon.sh start datanode
```

**启动完后检查**:

```shell
root@liunx01] jps
2112 DataNode
```


<使用浏览器访问linux01的50075端口，能打开一个datanode状态信息的网页>

刷新namenode的网页，看看是否识别出这一台datanode

启完之后，浏览namenode页面，会发现namenode已经认出4个datanode服务器

![4台datanode.jpg](https://i0.wp.com/i.loli.net/2019/11/15/nMgDGZ16joEaR42.jpg "4台datanode")

### 1.2.5  脚本自动启动HDFS系统 ★★★

**<font style="background-color: yellow;color: red;">知识补充</font>**

> ssh不仅可以远程登录一个linux服务器进行shell会话，还可以远程发送一条指令到目标机器上执行
> 
> `ssh linux04 "mkdir /root/test"`
> 
> `ssh linux04 "source /etc/profile; hadoop-daemon.sh start datanode"`
> 
> 但是，远程发送指令执行需要经过ssh的安全验证：
> 
> 验证分为两种方式：
> 
> 1、密码验证方式
> 
> 2、密钥验证方式
> 
> 需要在linux01上生成一对密钥：
> 
> ssh-keygen  
> 
> 然后将公钥发送到目标机器上：
> 
>     ssh-copy-id linux01
>     ssh-copy-id linux02
>     ssh-copy-id linux03
>     ssh-copy-id linux04

接着修改 hadoop安装目录/etc/hadoop/slaves文件，列入想成为datanode服务器的主机：

```shell
linux01
linux02
linux03
linux04
```

然后，就可以：

用`start-dfs.sh`自动启动整个集群；

用`stop-dfs.sh`自动停止整个集群。

#### 1.2.5.1 三种启动方式

三种启动方式介绍
方式一：逐一启动（实际生产环境中的启动方式）

```bash
hadoop-daemon.sh  start|stop  namenode|datanode|journalnode

yarn-daemon.sh  start|stop  esourcemanager|nodemanager
```

方式二：分开启动

```bash
start-dfs.sh

start-yarn.sh
```

方式三：一起启动


```bash
start-all.sh
```

**说明**：start-all.sh实际上是调用sbin/start-dfs.sh脚本和sbin/start-yarn.sh脚本

**脚本解读**<br>
start-dfs.sh脚本：

1. 通过命令bin/hdfs getconf –namenodes查看namenode在那些节点上

2. 通过ssh方式登录到远程主机，启动hadoop-deamons.sh脚本 

3. hadoop-deamon.sh脚本启动slaves.sh脚本

4. slaves.sh脚本启动hadoop-deamon.sh脚本，再逐一启动


注意：为什么要使用SSH??

当执行start-dfs.sh脚本时，会调用slaves.sh脚本，通过ssh协议无密码登陆到其他节点去启动进程。

 三种启动方式的关系
- `start-all.sh`：其实调用`start-dfs.sh`和 `start-yarn.sh`
- `start-dfs.sh`：调用`hadoop-deamon.sh`
- `start-yarn.sh`：调用`yarn-deamon.sh`

[![image](https://pic.imgdb.cn/item/60caaff5844ef46bb2d7de01.png)](https://p.pstatp.com/origin/1385e00008100972701d2)

**`hadoop-daemon.sh` 和`Hadoop-daemons.sh` 的区别**

`Hadoop-daemon.sh`:用于启动当前节点的进程<br>
例如：`Hadoop-daemon.sh start namenode` 用于启动当前的名称节点
 
`Hadoop-daemons.sh`：用于启动所有节点的进程<br>
例如：`Hadoop-daemons.sh start datanode` 用于启动所有节点的数据节点

## 1.3 windows安装Hadoop
参考：

[windows 10 上编译 Hadoop](https://www.cnblogs.com/jhxxb/p/10765815.html)<br>
[winutils](https://github.com/cdarlint/winutils)<br>
[hadoop下载路径](http://hadoop.apache.org/releases.html)

### 1.3.1 安装步骤
一、安装JDK

**建议：安装时不要安装到有空格的目录路径中，这样Hadoop在找JAVA_HOME的时候会找不到**

二、配置Java环境变量

1、JAVA\_HOME : `d:\Java\jdk1.8.0_291`

2、CLASSPATH : `.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;`

3、path : 添加 `%JAVA_HOME%\bin;%JAVA_HOME%\jre\bin;` 到最前面

4、测试 : 打开命令行cmd ,在任意路径下输入以下命令，返回一下结果即配置正确

```
命令：java -version 

返回：
   java version "jdk1.8.0_291"
   Java(TM) SE Runtime Environment (build jdk1.8.0_291-b13)
   Java HotSpot(TM) 64-Bit Server VM (build 25.291-b13, mixed mode)

命令：javac -version

返回：javac 1.8.0_291
```

三、下载Hadoop

1、下载路径：http://hadoop.apache.org/releases.html

2、解压到 D:\NativeLibraries\Hadoop\hadoop-2.8.5

四、配置Hadoop环境变量

1、HADOOP_HOME : D:\NativeLibraries\Hadoop\hadoop-2.8.5

2、path : 添加 %HADOOP_HOME%\bin;

3、下载[winutils](https://github.com/cdarlint/winutils)对应版本的bin，加压并覆盖HADOOP_HOME下的bin目录

4、测试：打开命令行cmd ,在任意路径下输入hadoop version命令，返回一下结果即配置正确

### 1.3.2 问题
1、 java -version可以正常查询到版本，但是hadoop -version无法查询到版本。`JAVA_HOME is incorrectly set`

```bash
C:\Users\Legion>hadoop version
系统找不到指定的路径。
Error: JAVA_HOME is incorrectly set.
       Please update D:\NativeLibraries\Hadoop\hadoop-3.2.2\etc\hadoop\hadoop-env.cmd
'-Xmx512m' 不是内部或外部命令，也不是可运行的程序
或批处理文件。
```
问题为**JAVA\_HOME路径不对，包含了空格**，比如Java安装到了`D:\Program Files\Java\jdk1.8.0_291`

修改 `%HADOOP_HOME%\etc\hadoop\hadoop-env.cmd`下的`set JAVA_HOME=%JAVA_HOME%`,修改为`set JAVA_HOME=D:\Progra~1\Java\jdk1.8.0_291`

> 备注:
> 
> ==使用`Progra~1`替代`Program Files`==

2、`hadoop checknative`不包含snappy

使用hadoop.dll、snappy.dll替换`%HADOOP_HOME%\bin`下文件

# 2. HDFS分布式文件系统的使用

**命令行客户端**：

客户端可以运行在任何机器上（前提：这台机器能够跟hdfs集群的机器联网；这台机器上需要有hadoop安装包）

（引申点：hdfs集群中的任意一台机器，都可以运行hdfs客户端）

**具体使用命令**：

参考：https://blog.csdn.net/weixin_42951763/article/details/120587862

**查看帮助**
```bash
命令：hdfs dfs -help  [cmd ...]
参数：
	cmd... 需要查询的一个或多个命令
```



## 2.1 查看目录或文件

```bash
命令：hdfs dfs -ls [-C] [-d] [-h] [-R] [-t] [-S] [-r] [-u] path
参数：
	-C 仅显示文件和目录的路径
	-d 目录列为普通文件
	-h 以人类可读的方式格式化文件大小，而不是按字节数
	-R 递归地列出目录的内容
	-t 按修改时间对文件排序（最近的第一次）
	-S 按大小对文件进行排序
	-r 颠倒排序顺序
	-u 使用上次访问的时间而不是修改来显示和排序
```


## 2.2 上传机器本地的一个文件到HDFS中

```shell
# put与copyFromLocal本质是进行复制操作，moveFromLocal本质是剪切操作
hadoop fs -put 本地文件 hdfs的目录
hdfs dfs -moveFromLocal <localsrc> ... <dst>

参数：
	-f 如果目标已存在，则覆盖该目标
	-p 保留访问和修改时间、所有权和模式
	-d 跳过临时文件的创建
```
(-copyFromLocal、-moveFromLocal)

比如：

```shell
hadoop fs  -put /root/a.txt   /

# 将本地的 a.txt上传到hdfs根目录，再删除本地的 a.txt
hdfs dfs -moveFromLocal /root/a.txt /
```


## 2.3 从HDFS中下载一个文件到本地机器

```shell
#  注意：下载多个文件时，目标必须是目录
hadoop fs -get /hdfs路径 /本地路径
参数：
	-f 如果目标已存在，则覆盖该目标
	-ignoreCrc 忽略CRC校验
	-crc 使用CRC校验
```
(-copyToLocal)

比如：


```shell
hadoop fs -get /a.txt /root
```


**<font style="color: red;">将hdfs指定目录下所有内容保存为一个文件，同时down至本地</font>**，文件不存在时会自动创建，文件存在时会覆盖里面的内容

```bash
hdfs dfs -getmerge -nl  < hdfs dir >  < local file >

参数：
    -nl：合并到local file中的hdfs文件之间会空出一行
```

例子：
```shell
hdfs dfs -getmerge /user/test/ /root/ab.txt
```

![hdfs批量下载.png](https://i0.wp.com/i.loli.net/2019/11/15/fY5My4PmL3CujT2.png "hdfs批量下载")

## 2.4 在HDFS中创建目录

```shell
hadoop fs -mkdir -p /aaa/bbb
```

## 2.5 在hadoop指定目录下新建一个空文件

```shell
hdfs dfs -touchz /aaa/test03.txt
```

## 2.6 查看HDFS中指定目录下的信息

```shell
hadoop  fs -ls  /
```

## 2.7 删除HDFS中的一个文件/文件夹

1. **删除HDFS中的一个文件**
```shell
hadoop fs -rm /spark-2.2.0-bin-hadoop2.7.tgz

参数：
	-f 如果文件不存在，不显示诊断消息或修改退出状态以反映错误
	-[rR] 递归删除目录
	-skipTrash 删除时不经过回收站，直接删除
	-safely 需要安全确认
```

2. **删除HDFS中的一个空文件夹**
```shell
hadoop fs -rmdir /aaa/bbb
```

3. **删除HDFS中的一个文件夹（可以是非空）**
```shell
hadoop fs -rm -r  /aaa
hdfs dfs -rmr /aaa/bbb
```

## 2.8 复制HDFS的文件到其他目录

```shell
hdfs dfs -cp /aaa/bbb /123/bbb
参数：
	-f 如果目标已存在，则覆盖该目标
	-p | -p[topax] 保留状态，参数[topax]分别表示（时间戳、所有权、权限、ACL、XAttr），如果指定了-p且没有arg，则保留时间戳，所有权和权限
	-d 跳过临时文件的创建
```

**复制HDFS的文件到其他集群目录**

```shell
# dscp会启动一个MapReduce程序，为每一个block单独启动一个线程，进行数据的读写操作

hdfs dfs -dscp hdfs集群1的路径 hdfs集群2的路径
```

## 2.9 移动HDFS中的文件或目录到另一个路径； 或者改名

```shell
hadoop fs -mv /aaa/bbb  /ccc
hadoop fs -mv /a.txt  /aaa/a.txt.o
```

## 2.10 查看HDFS中文件的内容（文本文件、gz压缩包）

文本文件：**cat与text作用相同**

压缩文件：文件为压缩格式（gzip以及hadoop的二进制序列文件格式）时，text会先解压缩

```shell
hadoop fs -cat  /aaa/a.txt.o
hadoop fs -text  /aaa/c.txt.gz
hadoop fs -cat /aaa/a.txt.o | less 查看a.txt.o中的全部内容(支持前后翻页)

hadoop fs -tail  /aaa/a.txt.o    查看a.txt.o中的末尾27（？）行
hadoop fs -tail -f  /aaa/a.txt.o 实时监视a.txt.o中新增的内容
```

## 2.11 追加一个本地文件内容到HDFS中已存在的文件中

```shell
hadoop fs -appendToFile  /root/a.txt  /aaa/a.txt.o
```

## 2.12 修改HDFS中的文件权限

```shell
修改所属用户和所属组
hadoop fs -chown -R angelababy:mygirls /aaa

修改权限：
hadoop fs -chmod 777 /aaa
```

## 2.13 查看HDFS ls后某一列数据（创建时间、大小、文件路径）

查看HDFScheckLines下面所有文件路径
```shell
awk -F " " '{print $8}' 按空格切分，取第8列
hdfs dfs -ls checkLines | awk -F " " '{print $8}'
```

## 2.14 检查是否有坏点
如果所有的文件满足最小副本的要求，那么就认为文件系统是健康的。
fsck工具只会列出有问题的文件和block，但是它并不会对它们进行修复。

```shell
hdfs fsck /
hdfs dfsadmin -report
```

## 2.15 shell脚本删除指定日期前的hdfs数据

```shell
#!/bin/bash
old_version=$(hdfs dfs -ls /user/hadoop/checkLines/2019/02 | awk '{ if($6<"2019-02-05"){printf "%s\n", $8} }')
arr=(${old_version// / })
for version in ${arr[@]}; do
	hdfs dfs -rm -r -f $version
done
```


## 2.16 shell命令判断hdfs文件是否存在

在Linux文件系统中，判断本地某个文件是否存在：

```bash
# 这里的-f参数判断$file是否存在 
if [ ! -f "$file" ]; then
　　echo "文件不存在!"
fi
```

同样hadoop内置了提供了判断某个文件是否存在的命令

```bash
hdfs dfs -test 

-d 判断<path>是否是目录

-e 判断<path>是否存在

-f 判断<path>是否是个文件

-s 判断内容是否大于0bytes ，大于0为真

-z 判断内容是否等于0bytes，为0真
```

脚本为：

```bash
hdfs dfs -test -e /path/exist
if [ $? -eq 0 ] ;then 
    echo 'exist' 
else 
    echo 'Error! path is not exist' 
fi
```

除此之外，还可以判断某个文件是否是文件夹、是否是文件、某个文件的大小是否大于0或者等于0


```bash
hdfs dfs -test -d /path/exist
if [ $? -eq 0 ] ;then 
    echo 'Is a directory' 
else 
    echo 'Is not a directory' 
fi
  
hdfs dfs -test -f /path/exist
if [ $? -eq 0 ] ;then 
    echo 'Is a file' 
else 
    echo 'Is not a file' 
fi
 
hdfs dfs -test -s /path/exist
if [ $? -eq 0 ] ;then 
    echo 'Is greater than zero bytes in size' 
else 
    echo 'Is not greater than zero bytes in size' 
fi
 
 
hdfs dfs -test -z /path/exist
if [ $? -eq 0 ] ;then 
    echo 'Is zero bytes in size.' 
else 
    echo 'Is not zero bytes in size. '
fi
```




## 2.17 hdfs shell补充
补充：
- hadoop fs：通用的文件系统命令，针对任何系统，比如本地文件、HDFS文件、HFTP文件、S3文件系统等。
- hadoop dfs：特定针对HDFS的文件系统的相关操作，但是已经不推荐使用。
- hdfs dfs：与hadoop dfs类似，同样是针对HDFS文件系统的操作，官方推荐使用。


```shell
hdfs dfs  查看Hadoop HDFS支持的所有命令
hdfs dfs -ls  列出目录及文件信息
hdfs dfs -lsr  循环列出目录、子目录及文件信息
hdfs dfs -tail /user/sunlightcs/test.txt  查看最后1KB的内容

hdfs dfs -copyFromLocal test.txt /user/sunlightcs/test.txt  从本地文件系统复制文件到HDFS文件系统，等同于put命令
hdfs dfs -copyToLocal /user/sunlightcs/test.txt test.txt  从HDFS文件系统复制文件到本地文件系统，等同于get命令

hdfs dfs -chgrp [-R] /user/sunlightcs  修改HDFS系统中/user/sunlightcs目录所属群组，选项-R递归执行，跟linux命令一样
hdfs dfs -chown [-R] /user/sunlightcs  修改HDFS系统中/user/sunlightcs目录拥有者，选项-R递归执行
hdfs dfs -chmod [-R] MODE /user/sunlightcs  修改HDFS系统中/user/sunlightcs目录权限，MODE可以为相应权限的3位数或+/-{rwx}，选项-R递归执行

hdfs dfs -count [-q] PATH  查看PATH目录下，子目录数、文件数、文件大小、文件名/目录名
hdfs dfs -cp SRC [SRC …] DST 将文件从SRC复制到DST，如果指定了多个SRC，则DST必须为一个目录
hdfs dfs -du PATH  显示该目录中每个文件或目录的大小
hdfs dfs -dus PATH  类似于du，PATH为目录时，会显示该目录的总大小

hdfs dfs -expunge  清空回收站，文件被删除时，它首先会移到临时目录.Trash/中，当超过延迟时间之后，文件才会被永久删除

hdfs dfs -getmerge SRC [SRC …] LOCALDST [addnl]获取由SRC指定的所有文件，将它们合并为单个文件，并写入本地文件系统中的LOCALDST，选项addnl将在每个文件的末尾处加上一个换行符

hdfs dfs -test -[ezd] PATH  对PATH进行如下类型的检查：-e PATH是否存在，如果PATH存在，返回0，否则返回1；-z 文件是否为空，如果长度为0，返回0，否则返回1； -d 是否为目录，如果PATH为目录，返回0，否则返回1  

hdfs dfs -text PATH  显示文件的内容，当文件为文本文件时，等同于cat；文件为压缩格式（gzip以及hadoop的二进制序列文件格式）时，会先解压缩 

hdfs dfs -help ls  查看某个[ls]命令的帮助文档
```

# 3.HDFS安全模式
## 3.1 简介
安全模式是HDFS所处的一种特殊状态，在这种状态下，文件系统只接受读数据请求，而不接受删除、修改等变更请求。在NameNode主节点启动时，HDFS首先进入安全模式，DataNode在启动的时候会向namenode汇报可用的block等状态，当整个系统达到安全标准时，HDFS自动离开安全模式。如果HDFS出于安全模式下，则文件block不能进行任何的副本复制操作，因此达到最小的副本数量要求是基于datanode启动时的状态来判定的，启动时不会再做任何复制（从而达到最小副本数量要求）

**系统什么时候才离开安全模式，需要满足哪些条件？**

当收到来自datanode的状态报告后，namenode根据配置，确定 1）可用的block占总数的比例、2）可用的数据节点数量符合要求之后，离开安全模式。如果有必要，也可以通过命令强制离开安全模式。与安全模式相关的主要配置在hdfs-site.xml文件中，主要有下面几个属性

**dfs.namenode.replication.min:** 最小的文件block副本数量，默认为1.

**dfs.namenode.safemode.threshold-pct:** 副本数达到最小要求的block占系统总block数的百分比，当实际比例超过该配置后，才能离开安全模式（但是还需要其他条件也满足）。默认为0.999f，也就是说符合最小副本数要求的block占比超过99.9%时，并且其他条件也满足才能离开安全模式。如果为小于等于0，则不会等待任何副本达到要求即可离开。如果大于1，则永远处于安全模式。

**dfs.namenode.safemode.min.datanodes:** 离开安全模式的最小可用（alive）datanode数量要求，默认为0.也就是即使所有datanode都不可用，仍然可以离开安全模式。

**dfs.namenode.safemode.extension:** 当集群可用block比例，可用datanode都达到要求之后，如果在extension配置的时间段之后依然能满足要求，此时集群才离开安全模式。单位为毫秒，默认为1.也就是当满足条件并且能够维持1毫秒之后，离开安全模式。 这个配置主要是对集群的稳定程度做进一步的确认。避免达到要求后马上又不符合安全标准。

总结一下，要离开安全模式，需要满足以下条件：

1. **达到副本数量要求的block比例满足要求**； 
2. **可用的datanode节点数满足配置的数量要求**； 
3. (1、2)两个条件满足后维持的时间达到配置的要求。

## 3.2 操作命令

Hadoop提供脚本用于对安全模式进行操作，主要命令为：

```shell
hdfs dfsadmin -safemode <command>
```
command的可用取值如下：

|command|功能|
|---|---|
|get|查看当前状态|
|enter|进入安全模式|
|leave|强制离开安全模式|
|wait|一直等待直到安全模式结束|

## 3.3 HDFS block丢失过多进入安全模式（safe mode）的解决方法
**背景及现象描述(Background and Symptom)**

因磁盘空间不足，内存不足，系统掉电等其他原因导致dataNode datablock丢失，出现如下类似日志：

*    The number of live datanodes 3 has reached the minimum number 0.    Safe mode will be turned off automatically once the thresholds have been reached.    Caused by: org.apache.hadoop.hdfs.server.namenode.SafeModeException: Log not rolled.     Name node is in safe mode.    The reported blocks 632758 needs additional 5114 blocks to reach the threshold 0.9990    of total blocks 638510.    The number of live datanodes 3 has reached the minimum number 0.    Safe mode will be turned off automatically once the thresholds have been reached.    at org.apache.hadoop.hdfs.server.namenode.FSNamesystem.checkNameNodeSafeMode    (FSNamesystem.java:1209)        ... 12 more*

**原因分析(Cause Analysis)**

由于系统断电，内存不足等原因导致dataNode丢失超过设置的丢失百分比，系统自动进入安全模式

**解决办法(Solution)**

安装HDFS客户端，并执行如下命令：


```shell
步骤 1 执行命令退出安全模式：
hdfs dfsadmin -safemode leave

步骤 2 执行健康检查，删除损坏掉的block。
hdfs fsck / -delete
```


注意: 这种方式会出现数据丢失，损坏的block会被删掉

## 3.4 通过命令来查看NameNode的状态（是Active还是Standby)


```shell
[root@hadoop01 ~]# hdfs haadmin -getServiceState nn1
active
[root@hadoop01 ~]# hdfs haadmin -getServiceState nn2
standby
```

# 4. 管理HDFS集群
HDFS为集群管理者提供了一系列管理工具，通过这些管理工具，管理员可以及时了解集群运行状态。下面对几个常用的工具进行介绍：

## 4.1 dfsadmin
dfsadmin是Hadoop提供的命令行形式的多功能管理工具，通过该工具可以查看Hadoop集群的运行状态、管理Hadoop集群。该工具的调用方式为:

```shell
$ bin/hdfs dfsadmin <command> [params...]
```
dfsadmin提供的功能如表3-2所示。

表3-2 dfsadmin命令


|命令|格式|说明|
|---|---|---|
|-report|-report|显示文件系统的统计信息及集群的运行状态。|
|-saveNamespace|	-saveNamespace|将当前内存中的文件系统映像保持为一个新的fsimage文件，重置edits文件。 <br>*• 该操作仅在安全模式下进行<br>• 需要superusesr权限*|
|-restoreFailedStorage|-restoreFailedStorage <true/false/check>|设置/取消/检查NameNode中恢复失败存储的可用副本的标记。<br>*• 需要superusesr权限*|
|-refreshNodes|-refreshNodes|更新允许连接到namenode的datanode列表。该操作会使namenode重新读取dfs.hosts、dfs.host.exclude的配置。<br>*• 详细情况，请参见3.2.3日常维护中的"加入和接触节点"一节*|
|-finalizeUpgrade|-finalizeUpgrade|移除datanode和namenode存储目录下的旧版本数据。一般在升级完成后进行。<br>*• CDH v1.0是第一个CDH发行版，关于系统升级的说明将在CDH v2.0的说明文档中阐述*|
|-upgradeProgress|-upgradeProgress <status/details/force>|	获取HDFS的升级进度，或强制升级。<br>*• 关于系统升级的说明将在CDH v2.0的说明文档中阐述*|
|-metasave|-metasave filename|将HDFS部分信息存储到Hadoop日志目录下的指定文件中。这些信息包括: <br>*• 正在被复制或删除的块信息<br>• 连接上的datanode列表*| 
|-refreshServiceAcl|-refreshServiceAcl|刷新namenode的服务级授权策略文件|
|-refreshUserToGroupsMappings|-refreshUserToGroupsMappings|刷新用户组信息|
|-refreshSuperUserGroupsConfiguration|-refreshSuperUserGroupsConfiguration	|刷新超级用户代理组信息|
|-printTopology|-printTopology|显示集群拓扑结构|
|refreshNamenodes|-refreshNamenodes <datanodehost:port>	|使给定datanode重新载入配置文件，停止管理旧的block-pools，并开始管理新的block-pools|
|-deleteBlockPool|-deleteBlockPool <datanode-host:port blockpoolId> [force] |	删除指定datanode下给定的blockpool。<br>*• force参数将删除blockpool对于的数据目录，无论他是否为空 <br>• 该命令只有在blockpool已下线的情况下执行，可使用refreshNamenodes命令来停止一个blockpool*|
|-setQuota|-setQuota <quota> <dirname>...<dirname>|设置目录配额，即该目录下最多包含多少个目录和文件。该配置主要用于防止用户创建大量的小文件，以保护namenode的内存安全。|
|-clrQuota|-clrQuota <dirname>...<dirname>|清除指定目录的配额。|
|-setSpaceQuota|-setSpaceQuota <quota> <dirname>...<dirname>|设置目录的空间配额，以限制目录的规模。一般用户用于给多个用户分配存储空间。|
|-clrSpaceQuota|-clrSpaceQuota <dirname>...<dirname>|清理目录的空间配额。|
|-setBalancerBandwidth|-setBalancerBandwidth <bandwidth in bytes per second>|设置负载均衡器的带宽限制。限制负载均衡器的带宽十分有必要。|
|-fetchImage|-fetchImage <local directory>|	下载最新的fsimage文件到指定的本地目录下。B34:C39B31:C39A26:C39C15B36:C3A13:C39|

## 4.2 fsck
HDFS提供fsck工具来查找HDFS上的文件缺失的块、副本缺失的块和副本过多的块。使用方式如下：


```shell
$ bin/hdfs fsck <path> [-move| -delete|-openforwrite] [-files[-blocks[-locations|-racks]]]
```


表3-3 fsck命令选项

|命令选项|说明|
|---|---|
|<path>|要检查的目录或文件的HDFS路径|
|-move|将损坏的文件移动到 /lost+found|
|-delete|删除损坏的文件|
|-openforwrite|显示正在写入的文件|
|-files|显示被检查的文件|
|-blocks|显示文件中的各个块信息|
|-locations|显示每一块的位置|
|-racks|显示每个块的机架位置和DataNode地址|

来源： https://www.cnblogs.com/warmingsun/p/5056393.html

# 5. HDFS目录限额

 在多人共用HDFS的环境下，配置设置非常重要。特别是在Hadoop处理大量资料的环境，如果没有配额管理，很容易把所有的空间用完造成别人无法存取。Hdfs的配额设定是针对目标而不是针对账号，所有在管理上最好让每个账号仅操作某一个目录，然后对目录设置配置。

设定方法有两种：

- Name Quotas：设置某一个目录下文件总数
- Space Quotas：设置某一个目录下可使用空间大小

默认情况下Hdfs没有任何配置限制，可以使用 `hadoop fs -count` 来查看配置情况

- -h 自动换算单位，友好展示
- -v 显示标题行

```bash
hdfs dfs -count -q -h /user

以下是结果，依次表示为(none和inf表示没有设置配额):

文件数限额  可用文件数  空间限额    可用空间      目录数      文件数     总大小     文件/目录名
none          inf      none         inf         42          59      174.8 M     /user
```


**hdfs指令中count和ls的区别**<br>
1. count目录数的统计，不仅会向下递归统计，还会包括根目录的
2. ls统计，如果下面没有文件夹或文件，则数量为0；如果有文件夹或文件，则数量为文件夹和文件数量加1


## 5.1 Name Quotas(文件数量限额)

计算公式：`QUOTA – (DIR_COUNT + FILE_COUNT) = REMAINING_QUOTA`

这里的 10000 是指 `DIR_COUNT + FILE_COUNT = 10000`，最大值为 `Long.Max_Value` 

限制hdfs的/user/seamon文件夹下只能存放1000个文件
```bash
启用设定：hdfs dfsadmin -setQuota 1000 /user/seamon

清除设定：hdfs dfsadmin -clrQuota /user/seamon
```


## 5.2 Space Quotas(文件大小限额)

计算公式：`SPACE_QUOTA – CONTENT_SIZE = REMAINING_SPACE_QUOTA`

可以使用 M, F, T 代表 MB, GB, TB


```bash
启用设定：hdfs dfsadmin -setSpaceQuota 1G /user/seamon/

清除设定：hdfs dfsadmin -clrSpaceQuota /user/seamon
```


这里需要特别注意的是 **“Space Quota”的设置不是hdfs目录可存储的文件大小，而是写入Hdfs所有block块的大小** ，假设一个文件被切分为2个blocks，在core-site.xml里面设置 dfs.block.size=64MB，dfs.replication=3，那么该文件所需要的存储空间为：2 * 64M * 3 =  384MB

如果一个小文件（例如，1k大小的文件）被上传到hdfs，该文件并不能占满一整个blok，但是按照hdfs配置规则也需要按照一个blok计算，即存储空间为：1 x 64MB x 3 = 192MB

==**设置空间限额的时候必须将副本份数也考虑在里面**==

## 5.3 其它事项

hdfs的配额管理是跟着目录走，如果目录被重命名，配额依然有效。

麻烦的是，在设置完配额以后，**如果超过限制**，虽然**文件不会写入到hdfs**，但是文件名依然会存在，**只是文件size为0**。当加大配额设置后，**还需要将之前的空文件删除**才能进一步写入。

如果**新设置的quota值，小于**该目录现有的Name Quotas 及 Space Quotas，系统并不会给出错误提示，但是该目录的**配置会变成最新设置的quota**