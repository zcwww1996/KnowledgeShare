[TOC]

本文介绍了linux下的压缩程式tar、gzip、gunzip、bzip2、bunzip2、compress、uncompress、zip、unzip、rar、unrar等程式，以及如何使用它们对`.tar`、`.gz`、`.tar.gz`、`.tgz`、`.bz2`、`.tar.bz2`、`.Z`、`.tar.Z`、`.zip`、`.rar`这10种压缩文件进行操作

# tar
Linux下最常用的打包程序就是tar了，使用tar程序打出来的包我们常称为tar包，tar包文件的命令通常都是以.tar结尾的。生成tar包后，就可以用其它的程序来进行压缩了，所以首先就来讲讲tar命令的基本用法：
## tar命令的基本用法
tar命令的选项有很多(用man tar可以查看到)，但常用的就那么几个选项，下面来举例说明一下：

```shell
tar
-c: 建立压缩档案
-x：解压
-t：查看内容
-r：向压缩归档文件末尾追加文件
-u：更新原压缩包中的文件
```

这五个是独立的命令，压缩解压都要用到其中一个，可以和别的命令连用但只能用其中一个。

下面的参数是根据需要在压缩或解压档案时可选的。

```shell
-z：有gzip属性的
-j：有bz2属性的
-Z：有compress属性的
-v：显示所有过程
-O：将文件解开到标准输出
```

下面的参数-f是必须的

-f: 使用档案名字，切记，这个参数是最后一个参数，后面只能接档案名。
**1. 压缩成新tar包**
```shell
# tar -cf all.tar *.jpg
这条命令是将所有.jpg的文件打成一个名为all.tar的包。-c是表示产生新的包 ，-f指定包的文件名。
```

**2. 添加文件到tar包**
```shell
# tar -rf all.tar *.gif
这条命令是将所有.gif的文件增加到all.tar的包里面去。-r是表示增加文件的意思
```

**3. 更新tar包中的文件**
```shell
# tar -uf all.tar logo.gif
这条命令是更新原来tar包all.tar中logo.gif文件，-u是表示更新文件的意思。
```


**4. 显示tar包中的文件**
```shell
# tar -tf all.tar
这条命令是列出all.tar包中所有文件，-t是列出文件的意思。
```

**5. 解压tar包**
```shell
# tar -xf all.tar

这条命令是解出all.tar包中所有文件，-x是解开的意思。
```



以上就是tar的最基本的用法。为了方便用户在打包解包的同时可以压缩或解压文件，tar提供了一种特殊的功能。这就是tar可以在打包或解包的同时调用其它的压缩程序，比如调用gzip、bzip2等。

### 1) tar调用gzip

gzip是GNU组织开发的一个压缩程序，.gz结尾的文件就是gzip压缩的结果。与gzip 相对的解压程序是gunzip。tar中使用-z这个参数来调用gzip。下面来举例说明一下：

```shell
# tar -czf all.tar.gz *.jpg
```

这条命令是将所有.jpg的文件打成一个tar包，并且将其用gzip压缩，生成一个gzip压缩过的包，包名为all.tar.gz

```shell
# tar -xzf all.tar.gz
```

这条命令是将上面产生的包解开。

### 2) tar调用bzip2

bzip2是一个压缩能力更强的压缩程序，.bz2结尾的文件就是bzip2压缩的结果。

与bzip2相对的解压程序是bunzip2。tar中使用-j这个参数来调用gzip。下面来举例说明一下：

```shell
# tar -cjf all.tar.bz2 *.jpg
```

这条命令是将所有.jpg的文件打成一个tar包，并且将其用bzip2压缩，生成一个bzip2压缩过的包，包名为all.tar.bz2

```shell
# tar -xjf all.tar.bz2
```

这条命令是将上面产生的包解开。

### 3) tar调用compress

compress也是一个压缩程序，但是好象使用compress的人不如gzip和bzip2的人多。.Z结尾的文件就是bzip2压缩的结果。与 compress相对的解压程序是uncompress。tar中使用-Z这个参数来调用compress。下面来举例说明一下：

```shell
# tar -cZf all.tar.Z *.jpg
```

这条命令是将所有.jpg的文件打成一个tar包，并且将其用compress压缩，生成一个uncompress压缩过的包，包名为all.tar.Z

```shell
# tar -xZf all.tar.Z
```

这条命令是将上面产生的包解开

### 补充：`tar: 从成员名中删除开头的“/”`

**a. 在打包和压缩的过程中，我们有时候会看到这样的语句：tar: 从成员名中删除开头的“/”，这个并不是报错，是因为没有加上-P选项，没有保留原来的绝对路径去打包或者压缩，提取打包的内容跟解压一样**

**压缩**

例子：将/root/目录以gzip的方式压缩为root.tar.gz压缩文件：

1. 没有加-P选项：

[![压缩_no_P.png](https://www.helloimg.com/images/2020/12/29/_no_P2c50fd4879ebd81e.png)](https://img-blog.csdnimg.cn/20190816181744301.png)

2. 加上-P选项：

[![压缩_P.png](https://www.helloimg.com/images/2020/12/29/_P713137b91f892203.png)](https://img-blog.csdnimg.cn/20190816181754945.png)

解压的时候同理，如果在压缩文件的时候使用了-P选项，那么在解压的时候也要加上-P选项，不然也会出现”tar: 从成员名中删除开头的“/”“，如下图：


**解压**

1. 不加-P选项解压使用了-P选项压缩/root/后的root.tar.gz文件：

[![解缩_no_P.png](https://www.helloimg.com/images/2020/12/29/_no_Pc15d32d9e5faf6a8.png)](https://img-blog.csdnimg.cn/20190816181851738.png)

2. 加上-P选项解压使用了-P选项压缩/root/后的root.tar.gz文件：

[![解缩_P.png](https://www.helloimg.com/images/2020/12/29/_P69b209dd6077cc1e.png)](https://img-blog.csdnimg.cn/20190816181905583.png)

**b. 在使用tar压缩或者打包的时候，可以通过增加--exclude来达到排除指定的文件的目的**

将/root/目录下的harry目录打包，但是不打包harry目录下的ha.txt文件，如下图：

[![exclude.png](https://www.helloimg.com/images/2020/12/29/exclude8764b66184fa306f.png)](https://img-blog.csdnimg.cn/20190816181945333.png)

压缩文件也是同理，想要排除指定的目录压缩或者打包也是同理

## tar系列的压缩文件作一个小结
### 1)对于.tar结尾的文件

```shell
tar -xf all.tar
```

### 2)对于.gz结尾的文件
解压
```shell
gzip -d all.gz

gunzip all.gz
```
压缩

```
#默认不保留源文件
gzip ce.txt
#保留源文件
gzip -c ce.txt > ce.txt.gz
#压缩目录
gzip -r directory
```

### 3)对于.tgz或.tar.gz结尾的文件
解压
```shell
tar -xzf all.tar.gz

tar -xzf all.tgz
# 解压到指定目录
tar zxvf all.tgz -C /source/Linux/
```
压缩
```shell
# 比如将/linux/test目录压缩到/home/test.tar.gz
tar zcvf /home/test.tar.gz /linux/test
```

### 4)对于.bz2结尾的文件
解压
```shell
bzip2 -d all.bz2

bunzip2 all.bz2
```
压缩
```
bzip2 -z FileName
```

### 5)对于tar.bz2结尾的文件

```shell
tar -xjf all.tar.bz2
```

### 6)对于.Z结尾的文件
解压
```shell
uncompress all.Z
```
压缩

```
compress FileName
```

### 7)对于.tar.Z结尾的文件

```shell
tar -xZf all.tar.z
```
# gzip

".gz"格式是 Linux 中最常用的压缩格式，使用 gzip 命令进行压缩，其基本信息如下：

命令名称：gzip。

英文原意：compress or expand files。

所在路径：/bin/gzip。

执行权限：所有用户。

功能描述：压缩文件或目录。

命令格式

```bash
[root@localhost ~]# gzip [选项] 源文件
```

选项：
- -c：将压缩数据输出到标准输出中，可以用于保留源文件；
- -d：解压缩；
- -r：对目录下的所有文件递归压缩；
- -v：显示压缩文件的信息；
- -数字：用于指定压缩等级，-1 压缩比最差；-9 压缩比最高。默认压缩比是 -6；
- -t：测试，检查压缩文件是否完整。
- -l：列出压缩文件的内容信息。

### 1)基本压缩。
gzip 压缩命令非常简单，甚至不需要指定压缩之后的压缩包名，只需指定源文件名即可。我们来试试：

```bash
[root@localhost ~]# gzip install.log
#压缩instal.log 文件
[root@localhost ~]# ls
anaconda-ks.cfg install.log.gz install.log.syslog
```

压缩文件生成，但是源文件也消失了

把test目录下的每个文件都单独压缩成.gz文件
```bash
[root@cxf test]# touch {1..6}.txt
[root@cxf test]# ls
1.txt  2.txt  3.txt  4.txt  5.txt  6.txt

[root@cxf test]# gzip *.txt
[root@cxf test]# ls
1.txt.gz  2.txt.gz  3.txt.gz  4.txt.gz  5.txt.gz  6.txt.gz
```


### 2)保留源文件压缩/解压

**压缩保留源文件**：

在使用 gzip 命令压缩文件时，源文件会消失，从而生成压缩文件。能不能在压缩文件的时候，不让源文件消失？好吧，也是可以的，不过很别扭。

```bash
[root@localhost ~]# gzip -c anaconda-ks.cfg > anaconda-ks.cfg.gz
```

使用-c选项，但是不让压缩数据输出到屏幕上，而是重定向到压缩文件中，这样可以缩文件的同时不删除源文件

```bash
[root@localhost ~]# ls
anaconda-ks.cfg anaconda-ks.cfg.gz install.log.gz install.log.syslog
```

可以看到压缩文件和源文件都存在


**解压缩保留源文件**： 

```bash
gunzip -c filename.gz > filename
```

### 3)压缩目录。
我们可能会想当然地认为 gzip 命令可以压缩目录。 

```bash
[root@localhost ~]# mkdir test
[root@localhost ~]# touch test/test1
[root@localhost ~]# touch test/test2
[root@localhost ~]# touch test/test3 #建立测试目录，并在里面建立几个测试文件
[root@localhost ~]# gzip -r test/
#压缩目录，并没有报错
[root@localhost ~]# ls
anaconda-ks.cfg anaconda-ks.cfg.gz install.log.gz install.log.syslog test
#但是査看发现test目录依然存在，并没有变为压缩文件
[root@localhost ~]# ls test/
testl .gz test2.gz test3.gz
```

**原来`gzip -r`命令不会打包目录，而是把目录下所有的子文件分别压缩**

在 Linux 中，打包和压缩是分开处理的。而 gzip 命令只会压缩，不能打包，所以才会出现没有打包目录，而只把目录下的文件进行压缩的情况。



另外对于Window下的常见压缩文件.zip和.rar，Linux也有相应的方法来解压它们：

# zip
zip是linux和windows等多平台通用的压缩格式。zip比gzip更强的是**zip命令压缩文件不会删除源文件，还可以压缩目录**。

linux下提供了zip和unzip程序，如果没有，使用`yum install -y unzip zip`安装。

zip是压缩程序，unzip是解压程序。它们的参数选项很多，这里只做简单介绍，依旧举例说明一下其用法：

- -r：将指定目录下的所有文件和子目录一并压缩
- -x：压缩文件时排除某个文件
- -q：不显示压缩信息

### 1) 压缩文件
```bash
zip all.zip *.jpg
#这条命令是将所有.jpg的文件压缩成一个zip包
```


### 2) 压缩目录
```bash
# 压缩tmp目录
zip -r tmp.zip ./tmp/

#把/home目录下面的abc文件夹和123.txt压缩成为abc123.zip
zip -r abc123.zip abc 123.txt
```

### 3) 排查压缩

```bash
# 压缩tmp目录，并排除tmp目录里面的services.zip文件
zip -r tmp1.zip ./tmp/ -x tmp/services.zip
```

### 4) 解压文件
```bash
# 将all.zip中的所有文件解压到当前目录
unzip all.zip

# 把mydata.zip解压到/home/mydatabak目录里面
unzip mydata.zip -d /home/mydatabak
```

# rar
## 安装
要在linux下处理.rar文件，需要安装RAR for Linux，可以从网上下载，但要记住，RAR for Linux 不是免费的；可从http://www.rarsoft.com/download.htm下载 RAR 3.71 for Linux
0，然后安装：
```shell
# tar -xzpvf rarlinux-3.2.0.tar.gz 
# cd rar
# make
```
这样就安装好了，安装后就有了rar和unrar这两个程序
## 使用
rar是压缩程序，unrar 是解压程序。它们的参数选项很多，这里只做简单介绍，依旧举例说明一下其用法：
### 1) 压缩文件
```shell
# rar a all *.jpg
```
这条命令是将所有.jpg的文件压缩成一个rar包，名为all.rar，该程序会将.rar 扩展名将自动附加到包名后。

### 2) 解压文件
```shell
# unrar e all.rar
```
这条命令是将all.rar中的所有文件解压出来

到此为至，我们已经介绍过linux下的tar、gzip、gunzip、bzip2、bunzip2、compress 、 uncompress、 zip、unzip、rar、unrar等程式，你应该已经能够使用它们对.tar 、.gz、.tar.gz、.tgz、.bz2、.tar.bz2、. Z、.tar.Z、.zip、.rar这10种压缩文
件进行解压了，以后应该不需要为下载了一个软件而不知道如何在Linux下解开而烦恼了。而且以上方法对于Unix也基本有效。


## 以下补充


压缩

    tar –cvf jpg.tar *.jpg //将目录里所有jpg文件打包成tar.jpg
    tar –czf jpg.tar.gz *.jpg //将目录里所有jpg文件打包成jpg.tar后，并且将其用gzip压缩，生成一个gzip压缩过的包，命名为jpg.tar.gz
    tar –cjf jpg.tar.bz2 *.jpg //将目录里所有jpg文件打包成jpg.tar后，并且将其用bzip2压缩，生成一个bzip2压缩过的包，命名为jpg.tar.bz2
    tar –cZf jpg.tar.Z *.jpg //将目录里所有jpg文件打包成jpg.tar后，并且将其用compress压缩，生成一个umcompress压缩过的包，命名为jpg.tar.Z
    rar a jpg.rar *.jpg //rar格式的压缩，需要先下载rar for linux
    zip jpg.zip *.jpg //zip格式的压缩，需要先下载zip for linux

解压

    tar –xvf file.tar //解压 tar包
    tar -xzvf file.tar.gz //解压tar.gz
    tar -xjvf file.tar.bz2 //解压 tar.bz2
    tar –xZvf file.tar.Z //解压tar.Z
    unrar e file.rar //解压rar
    unzip file.zip //解压zip
     

总结

    1、*.tar 用 tar –xvf 解压
    2、*.gz 用 gzip -d或者gunzip 解压
    3、*.tar.gz和*.tgz 用 tar –xzf 解压
    4、*.bz2 用 bzip2 -d或者用bunzip2 解压
    5、*.tar.bz2用tar –xjf 解压
    6、*.Z 用 uncompress 解压
    7、*.tar.Z 用tar –xZf 解压
    8、*.rar 用 unrar e解压
    9、*.zip 用 unzip 解压

## 压缩文件校验


```bash
tar tvf abc.tar.gz
```

```bash
gzip -t abc.tar.gz
```
