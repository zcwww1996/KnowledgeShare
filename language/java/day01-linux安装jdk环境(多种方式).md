[TOC]
# 1. linux系统通用安装
## 1.1 通过tar.gz压缩包安装

此方法适用于绝大部分的linux系统
### 1.1.1 下载tar.gz的压缩包，这里使用官网下载。

进入：
http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html<br>
[![JAVA安装](https://z3.ax1x.com/2021/11/12/IDJoFg.png "JAVA安装")](https://b-ssl.duitang.com/uploads/item/201906/10/20190610152514_8veV4.png "JAVA安装")<br>
勾选接受许可协议后选择对应的压缩包，下载完成后上传的linux服务器上，这里是上传到/root 目录下。
也可以通过wget直接下载，注意这里的下载地址是有认证的。
### 1.1.2 下载完成后解压到指定文件下

先创建java文件目录，如果已存在就不用创建

```shell
[root@linux01 ~] # mkdir -p /usr/local/java
解压到java文件目录
[root@linux01 ~] # tar -vzxf jdk-8u161-linux-x64.tar.gz -C /usr/local/java/
```

### 1.1.3 添加环境变量，编辑配置文件

[root@linux01 ~] # vi /etc/profile

在文件最下方或者指定文件添加
```shell
export JAVA_HOME=/usr/local/java/jdk1.8.0_161
export CLASSPATH=$:CLASSPATH:$JAVA_HOME/lib/
export PATH=$PATH:$JAVA_HOME/bin
```
保存退出
（保存退出的命令是，Shift+:后输入wq回车）

### 1.1.4 重新加载配置文件
[root@linux01 ~] # <font color="red">**source /etc/profile**</font>

### 1.1.5 测试
[root@linux01 ~] # java -version

可以看到一下信息则表示配置成功

java version “1.8.0_161”

Java™ SE Runtime Environment (build 1.8.0_161-b12)

Java HotSpot™ 64-Bit Server VM (build 25.161-b12, mixed mode)

### 1.1.6 java版本不一致问题
`java -version`显示openjdk，`javac -version`显示自己安装的jdk

使用一下命令切换jdk版本
```bash
[root@localhost ~]# alternatives --install /usr/bin/java java /usr/local/java/jdk1.8.0_161/bin/java 100

[root@localhost ~]# alternatives --config java

There are 2 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
*+ 1           java-1.8.0-openjdk.x86_64 (/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.181-7.b13.el7.x86_64/jre/bin/java)
   2           /usr/local/java/jdk1.8.0_161/bin/java

Enter to keep the current selection[+], or type selection number: 2
```

> 如果添加错误可以使用以下命令删除:
> 
> ```bash
> alternatives --remove java  /usr/local/java/jdk1.8.0_161/bin
> ```


# 2. ubuntu系统
## 2.1 使用apt-get 命令安装
### 2.1.1 添加ppa

[root@linux01 ~] # `sudo add-apt-repository ppa:webupd8team/java`
[root@linux01 ~] # `sudo apt-get update`
### 2.1.2 安装oracle-java-installer

[root@linux01 ~] # `sudo apt-get install oracle-java8-installer`
安装器会提示你同意 oracle 的服务条款,选择 ok

然后选择yes 即可，如果你因为网络或者其他原因,导致installer 下载速度很慢或无法下载,可以中断操作.然后下载好相应jdk的tar.gz 包,放在:/var/cache/oracle-jdk8-installer下面,然后安装一次installer，installer则会默认使用你下载的tar.gz包。
### 2.1.3 测试

[root@linux01 ~] # java -version

可以看到一下信息则表示配置成功

java version “1.8.0_161”

Java™ SE Runtime Environment (build 1.8.0_161-b12)

Java HotSpot™ 64-Bit Server VM (build 25.161-b12, mixed mode)
# 3. red hat 或centos
## 3.1 使用rpm命令

1、通过官网下载选定版本的rpm包，然后放在指定目录下（这里是/tmp）

进入指定目录下cd /tmp

2、添加执行权限

```
[root@linux01 ~] # chmod +x /tmp/jdk-8u161-linux-x64.rpm
```

3、rpm安装

```
[root@linux01 ~] # rpm -ivh /tmp/jdk-8u161-linux-x64.rpm
```

4、查看版本信息

```
[root@linux01 ~] # java -version
可以看到一下信息则表示配置成功
java version “1.8.0_161”
Java™ SE Runtime Environment (build 1.8.0_161-b12)
Java HotSpot™ 64-Bit Server VM (build 25.161-b12, mixed mode)
```

## 3.2 使用yum源

这里需要注意yum源的配置

1、查看yum库中都有哪些jdk版本

[root@linux01 ~] # `yum search java|grep jdk`

2、选择指定的版本安装，注意最后的 *
以及yum源安装的是openjdk，注意openjdk的区别。

```shell
[root@linux01 ~] # yum install java-1.8.0-openjdk*
```
3、安装完成后查看版本信息

[root@linux01 ~] # java -version

# 4. 切换JDK
在有openJDK的情况下，安装官方JDK，并改为默认
[https://blog.csdn.net/u014648759/article/details/84699958](https://blog.csdn.net/u014648759/article/details/84699958)

将我们装好的SUN JDK装到系统里，

```shell
sudo update-alternatives --install /usr/bin/java java /usr/local/java/jdk1.8.0_181/bin/java 1100
sudo update-alternatives --install /usr/bin/javac javac /usr/local/java/jdk1.8.0_181/bin/javac 1100
```

蓝色部分，就是我们自己安装的jdk的地方，按情况改就是了。

最后将SUN JDK设为默认就可以了，

```shell
sudo update-alternatives --config java
```

这时如果有多个jdk的话（比如openJDK和SUN JDK），就会出来一个列表，当前默认的会在列表前面有一个" * " 号，这时我们就要选择我们刚装的SUN JDK的java的那个序号，输入这个序号，回车就行了。

# 5. Linux更新jar配置文件

有时候我们需要更新jar程序，但是又只有一点小改动，如果重新打包上传的话很费时间，我们可以对某个文件进行更新，步骤如下：

1) 列出指定文件路径：`jar -tvf test.jar|grep ipConfig`
2) 解压指定路径下的文件：`jar -xvf test.jar com/ctsi/ipConfig.properties`
3) 更新该文件 `vim ipConfig.properties`
4) 更新到jar包中 `jar -uvf test.jar com/test/ipConfig.properties`