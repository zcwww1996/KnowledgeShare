[TOC]

# 1. Linux 目录访问与修改

## 1.1 绝对路径、相对路径
Linux的目录结构为树状结构，最顶级的目录为根目录 /
- 绝对路径：

路径的写法，由根目录 / 写起，例如： /usr/share/doc 这个目录。
- 相对路径：

路径的写法，不是由 / 写起，例如由 /usr/share/doc 要到 /usr/share/man 底下时，可以写成： cd ../man 这就是相对路径的写法啦！
## 1.2 处理目录的常用命令
### ls: 列出目录
语法：
```shell
[root@linux01 ~]# ls [-aAdfFhilnrRSt] 目录名称
[root@linux01 ~]# ls [--color={never,auto,always}] 目录名称
[root@linux01 ~]# ls [--full-time] 目录名称
```
选项与参数：
- -a ：全部的文件，连同隐藏档( 开头为 . 的文件) 一起列出来(常用)
- -d ：仅列出目录本身，而不是列出目录内的文件数据(常用)
- -l ：长数据串列出，包含文件的属性与权限等等数据；(常用)

将root目录下的所有文件列出来(含属性与隐藏档)
``` shell
[root@linux02 zkdata]# ls -al ~

total 116
    dr-xr-x---.  7 root root  4096 Dec 20 14:10 .
    dr-xr-xr-x. 22 root root  4096 Apr 19 17:20 ..
    -rw-------.  1 root root   913 Nov 17 05:18 anaconda-ks.cfg
    -rw-------.  1 root root 11982 Jan 15 17:16 .bash_history
    -rw-r--r--.  1 root root    18 May 20  2009 .bash_logout
    -rw-r--r--.  1 root root   176 May 20  2009 .bash_profile
    -rw-r--r--.  1 root root   176 Sep 23  2004 .bashrc
    -rw-r--r--.  1 root root   100 Sep 23  2004 .cshrc
    drwxr-xr-x.  3 root root  4096 Nov 19 19:29 hdp-data
    -rw-r--r--.  1 root root  8823 Nov 17 05:17 install.log
    -rw-r--r--.  1 root root  3384 Nov 17 05:16 install.log.syslog
    drwxr-xr-x. 55 root root  4096 Jan 15 17:14 kafka-logs
    drwxr-xr-x.  2 root root  4096 Nov 17 19:29 .oracle_jre_usage
    drwx------.  2 root root  4096 Nov 19 18:57 .ssh
    -rw-r--r--.  1 root root   129 Dec  4  2004 .tcshrc
    -rw-------.  1 root root   512 Nov 17 06:33 .viminfo
    drwxr-xr-x.  3 root root  4096 Jan 15 17:12 zkdata
    -rw-r--r--.  1 root root 30371 Dec 26 15:45 zookeeper.out
```

以易读的详情列表方式显示root文件夹下的文件
```shell
[root@linux02 zkdata]# ll -h /root

    total 64K
    -rw-------.  1 root root  913 Nov 17 05:18 anaconda-ks.cfg
    drwxr-xr-x.  3 root root 4.0K Nov 19 19:29 hdp-data
    -rw-r--r--.  1 root root 8.7K Nov 17 05:17 install.log
    -rw-r--r--.  1 root root 3.4K Nov 17 05:16 install.log.syslog
    drwxr-xr-x. 55 root root 4.0K Jan 15 17:14 kafka-logs
    drwxr-xr-x.  3 root root 4.0K Jan 15 17:12 zkdata
    -rw-r--r--.  1 root root  30K Dec 26 15:45 zookeeper.out
```
### cd：切换目录
cd是Change Directory的缩写，这是用来变换工作目录的命令。
```shell
 cd [相对路径或绝对路径]

#使用 mkdir 命令创建 runoob 目录
[root@linux01 ~]# mkdir runoob

#使用绝对路径切换到 runoob 目录
[root@linux01 ~]# cd /root/runoob/

#使用相对路径切换到 runoob 目录
[root@linux01 ~]# cd ./runoob/

# 表示回到自己的home目录，亦即是 /root 这个目录
[root@linux01 runoob]# cd ~

# 表示去到目前的上一级目录，亦即是 /root 的上一级目录的意思；
[root@linux01 ~]# cd ..
```
### pwd：显示目前的目录
pwd是Print Working Directory的缩写，也就是显示目前所在目录的命令。
```shell
[root@linux01 ~]# pwd [-P]
选项与参数：
-P  ：显示出确实的路径，而非使用连结 (link) 路径。

范例：单纯显示出目前的工作目录：
[root@linux01 ~]# pwd
/root   <== 显示出目录啦～

范例：显示出实际的工作目录，而非连结档本身的目录名而已
[root@linux01 ~]# cd /var/mail   <==注意，/var/mail是一个连结档
[root@linux01 mail]# pwd
/var/mail         <==列出目前的工作目录
[root@linux01 mail]# pwd -P
/var/spool/mail   <==怎么回事？有没有加 -P 差很多～
[root@linux01 mail]# ls -ld /var/mail
lrwxrwxrwx 1 root root 10 Sep  4 17:54 /var/mail -> spool/mail
# 看到这里应该知道为啥了吧？因为 /var/mail 是连结档，连结到 /var/spool/mail
# 所以，加上 pwd -P 的选项后，会不以连结档的数据显示，而是显示正确的完整路径啊！
```
### mkdir：创建一个新的目录

语法：
mkdir [-mp] 目录名称
```shell
选项与参数：
    -m ：配置文件的权限喔！直接配置，不需要看默认权限 (umask) 的脸色～
    -p ：帮助你直接将所需要的目录(包含上一级目录)递归创建起来！
```
范例：请到/tmp底下尝试创建数个新目录看看：
```shell
[root@linux01 ~]# cd /tmp
[root@linux01 tmp]# mkdir test    <==创建一名为 test 的新目录

加了这个 -p 的选项，可以自行帮你创建多层目录！
[root@linux01 tmp]# mkdir -p test1/test2/test3/test4
```

范例：创建权限为rwx--x--x的目录

```shell
[root@linux01 tmp]# mkdir -m 711 test2
[root@linux01 tmp]# ls -l
drwxr-xr-x  3 root  root 4096 Jul 18 12:50 test
drwxr-xr-x  3 root  root 4096 Jul 18 12:53 test1
drwx--x--x  2 root  root 4096 Jul 18 12:54 test2

上面的权限部分，如果没有加上 -m 来强制配置属性，系统会使用默认属性。

如果我们使用 -m ，如上例我们给予 -m 711 来给予新的目录 drwx--x--x 的权限。
```

### rmdir (删除空的目录)

语法:
rmdir [-p] 目录名称

```shell
选项与参数：
    -p ：连同上一级『空的』目录也一起删除

删除 runoob 目录
[root@linux01 tmp]# rmdir runoob/
```

范例：将於mkdir范例中创建的目录(/tmp底下)删除掉！
```shell
[root@linux01 tmp]# ls -l   <==看看有多少目录存在？
drwxr-xr-x  3 root  root 4096 Jul 18 12:50 test
drwxr-xr-x  3 root  root 4096 Jul 18 12:53 test1
drwx--x--x  2 root  root 4096 Jul 18 12:54 test2
[root@linux01 tmp]# rmdir test   <==可直接删除掉，没问题
[root@linux01 tmp]# rmdir test1  <==因为尚有内容，所以无法删除！
rmdir: `test1': Directory not empty
[root@linux01 tmp]# rmdir -p test1/test2/test3/test4
[root@linux01 tmp]# ls -l        <==您看看，底下的输出中test与test1不见了！
drwx--x--x  2 root  root 4096 Jul 18 12:54 test2

利用 -p 这个选项，立刻就可以将 test1/test2/test3/test4 一次删除。

不过要注意的是，这个 rmdir 仅能删除空的目录，你可以使用 rm 命令来删除非空目录。
```
### cp (复制文件或目录)

语法:
```shell
[root@linux01 ~]# cp [-fipr] 来源档(source) 目标档(destination)
```
选项与参数：
```shell
    -f：为强制(force)的意思，若目标文件已经存在且无法开启，则移除后再尝试一次；
    -i：若目标档(destination)已经存在时，在覆盖时会先询问动作的进行(常用)
    -p：连同文件的属性（原文件的时间不变）一起复制过去，而非使用默认属性(备份常用)；
    -r：递归持续复制，用於目录的复制行为；(最常用)

1.相对路径  cp –r /etc/* .     cp –R ../aaa  ../../test/

2.绝对路径  cp –r /ect/service  /root/test/aa/bb

```
用 root 身份，将 root 目录下的 .bashrc 复制到 /tmp 下，并命名为 bashrc
```shell
[root@linux01 ~]# cp ~/.bashrc /tmp/bashrc
[root@linux01 ~]# cp -i ~/.bashrc /tmp/bashrc
cp: overwrite `/tmp/bashrc'? n  <==n不覆盖，y为覆盖
```

### mv (移动文件与目录，或修改名称)

语法：
```shell
[root@linux01 ~]# mv [-fiu] source destination
[root@linux01 ~]# mv [options] source1 source2 source3 .... directory
```

选项与参数：
```shell
    -f ：force 强制的意思，如果目标文件已经存在，不会询问而直接覆盖；
    -i ：若目标文件 (destination) 已经存在时，就会询问是否覆盖！
    -u ：若目标文件已经存在，且 source 比较新，才会升级 (update)
```
复制一文件，创建一目录，将文件移动到目录中
```shell
[root@linux01 ~]# cd /tmp
[root@linux01 tmp]# cp ~/.bashrc bashrc
[root@linux01 tmp]# mkdir mvtest
[root@linux01 tmp]# mv bashrc mvtest
```

将某个文件移动到某个目录去，就是这样做！

将刚刚的目录名称更名为 mvtest2
```shell
[root@linux01 tmp]# mv mvtest mvtest2
```

### rm: 移除文件或目录
命令路径：/bin/rm

执行权限：所有用户

作用：删除文件

语法： **rm [-rf] 文件或目录**
```shell
-r  递归删除！最常用在目录的删除了！非常危险！！！
-f  （force） 强制删除文件或目录,无需逐一确认
-i   互动模式，在删除前会询问使用者是否动作
```

将刚刚在 cp 的范例中创建的 bashrc 删除掉！
```shell
[root@linux01 tmp]# rm -i bashrc
rm: remove regular file `bashrc'? y
```
如果加上 -i 的选项就会主动询问喔，避免你删除到错误的档名

<font color="red">**注意：工作中，谨慎使用rm -rf 命令。**</font>
- 扩展点：删除乱码文件

一些文件乱码后使用rm -rf 依然无法删除，此时，使用`ll -i`查找到文件的inode节点

然后使用`find . -inum`查找到的inode数`-exec rm {} -rf \`，就能顺利删除了



# 2. Linux 文件基本属性

在Linux中第一个字符代表这个文件是目录、文件或链接文件等等。
-  当为[ d ]则是目录
-  当为[ - ]则是文件；
-  若是[ l ]则表示为链接文档(link file)；
-  若是[ b]则表示为装置文件里面的可供储存的接口设备(可随机存取装置)；
-  若是[ c]则表示为装置文件里面的串行端口设备,例如键盘、鼠标(一次性读取装置)。

接下来的字符中，以三个为一组，且均为『rwx』 的三个参数的组合。其中，[ r ]代表可读(read)、[ w ]代表可写(write)、[ x ]代表可执行(execute)。 要注意的是，这三个**权限的位置不会改变**，如果没有权限，就会出现**减号[ - ]** 而已。

## 2.1 文件的属性
每个文件的属性由左边第一部分的10个字符来确定（如下图）。

[![文件读写权限.png](https://ae01.alicdn.com/kf/Hda6607bcb1df432eac34eab90c6cc525S.jpg
"文件读写权限")](https://shop.io.mi-img.com/app/shop/img?id=shop_d0de4cd5fada0f039131f871e317b9bd.png)

## 2.2 更改文件属性
### 2.2.1 chgrp：更改文件属组

语法：

```shell
chgrp [-R] 属组名文件名
```

参数选项

-  -R：递归更改文件属组，就是在更改某个目录文件的属组时，如果加上-R的参数，那么该目录下的所有文件的属组都会更改。

### 2.2.2 chown：更改文件属主，也可以同时更改文件属组

语法：

```shell
chown [–R] 属主名 文件名
chown [-R] 属主名：属组名 文件名
```

进入 /root 目录（~）将install.log的拥有者改为bin这个账号：

```shell
[root@Linux01 ~] cd ~
[root@Linux01 ~]# chown bin install.log
[root@Linux01 ~]# ls -l
-rw-r--r-- 1 bin  users 68495 Jun 25 08:53 install.log
```

将install.log的拥有者与群组改回为root：

```shell
[root@Linux01 ~]# chown root:root install.log
[root@Linux01 ~]# ls -l
-rw-r--r-- 1 root root 68495 Jun 25 08:53 install.log
```

### 2.2.3 chmod：更改文件9个属性

Linux文件属性有两种设置方法，一种是数字，一种是符号。

Linux文件的基本权限就有九个，分别是owner/group/others三种身份各有自己的read/write/execute权限。
#### 2.2.3.1 数字改变文件权限

先复习一下刚刚上面提到的数据：文件的权限字符为：『-rwxrwxrwx』， 这九个权限是三个三个一组的！其中，我们可以使用数字来代表各个权限，各权限的分数对照表如下：
-  r : 4
-  w : 2
-  x : 1

每种身份(owner/group/others)各自的三个权限(r/w/x)分数是需要累加的，例如当权限为： [-rwxrwx---] 分数则是：
- owner = rwx = 4+2+1 = 7
- group = rwx = 4+2+1 = 7
- others= --- = 0+0+0 = 0

所以等一下我们设定权限的变更时，该文件的权限数字就是770啦！

变更权限的指令chmod的语法是这样的：

```shell
chmod [-R] xyz 文件或目录
```

选项与参数：
-  xyz : 就是刚刚提到的数字类型的权限属性，为 rwx 属性数值的相加。
-  **-R : 进行递归(recursive)的持续变更，亦即连同次目录下的所有文件都会变更**

举例来说，如果要将.bashrc这个文件所有的权限都设定启用，那么命令如下：

```shell
[root@Linux01 ~]# ls -al .bashrc
-rw-r--r-- 1 root root 395 Jul 4 11:45 .bashrc
[root@Linux01 ~]# chmod 777 .bashrc
[root@Linux01 ~]# ls -al .bashrc
-rwxrwxrwx 1 root root 395 Jul 4 11:45 .bashrc
```

那如果要将权限变成 -rwxr-xr-- 呢？那么权限的分数就成为 [4+2+1][4+0+1][4+0+0]=754。

#### 2.2.3.2 符号类型改变文件权限

九个权限分别是(1)user (2)group (3)others三种身份！ 那么我们就可以藉由u, g, o来代表三种身份的权限！

此外， **a 则代表 all 亦即全部的身份**！那么读写的权限就可以写成r, w, x！也就是可以使用底下的方式来看：

|chmod|用户|变更|权限|文件/目录|
|:---:|:---:|:---:|:---:|:---:|
chmod | u |+(加入)|r|文件或目录|
chmod | g |-(除去)|w|文件或目录|
chmod | o |=(设定)|x|文件或目录|

如果我们需要将文件权限设置为 -rwxr-xr-- ，可以使用 chmod u=rwx,g=rx,o=r 文件名 来设定:
```shell
[root@Linux01 ~]# ls -al .bashrc
-rwxr-xr-x 1 root root 395 Jul 4 11:45 .bashrc
[root@Linux01 ~]# chmod  a+w  .bashrc
[root@Linux01 ~]# ls -al .bashrc
-rwxrwxrwx 1 root root 395 Jul 4 11:45 .bashrc
```

而如果是要将权限去掉而不改变其他已存在的权限呢？例如要拿掉全部人的可执行权限，则：
```shell
[root@Linux01 ~]# chmod  a-x  .bashrc
[root@Linux01 ~]# ls -al .bashrc
-rw-rw-rw- 1 root root 395 Jul 4 11:45 .bashrc
```

加入user的execute权限，去掉other的write权限
```shell
[root@Linux01 ~]# ls -al .bashrc
-rw-rw-rw- 1 root root 395 Jul 4 11:45 .bashrc
[root@Linux01 ~]# chmod u+x,o-w .bashrc
[root@Linux01 ~]# ls -al .bashrc
-rwxrw-r-- 1 root root 395 Jul 4 11:45 .bashrc
```
#### 2.2.3.3 Linux中的文件特殊权限

linux中除了常见的读（r）、写（w）、执行（x）权限以外，还有3个特殊的权限，分别是setuid、setgid和stick bit

**1、setuid、setgid**

先看个实例，查看你的/usr/bin/passwd 与/etc/passwd文件的权限

```bash
[root@MyLinux ~] ls -l /usr/bin/passwd /etc/passwd

-rw-r--r-- 1 root root  1549 08-19 13:54 /etc/passwd
-rwsr-xr-x 1 root root 22984 2007-01-07 /usr/bin/passwd
```

众所周知，/etc/passwd文件存放的各个用户的账号与密码信息，/usr/bin/passwd是执行修改和查看此文件的程序，但从权限上看，/etc/passwd仅有root权限的写（w）权，可实际上每个用户都可以通过/usr/bin/passwd命令去修改这个文件，于是这里就涉及了linux里的特殊权限setuid，正如 -rwsr-xr-x 中的s

**setuid就是：让普通用户拥有可以执行“只有root权限才能执行”的特殊权限，setgid同理指”组“**

作为普通用户是没有权限修改/etc/passwd文件的，但给/usr/bin/passwd以setuid权限后，普通用户就可以通过执行passwd命令，临时的拥有root权限，去修改/etc/passwd文件了

**2、stick bit （粘贴位）**

再看个实例，查看你的/tmp目录的权限


```bash
[root@MyLinux ~] ls -dl /tmp

drwxrwxrwt 6 root root 4096 08-22 11:37 /tmp
```


tmp目录是所有用户共有的临时文件夹，所有用户都拥有读写权限，这就必然出现一个问题，A用户在/tmp里创建了文件a.file，此时B用户看了不爽，在/tmp里把它给删了（因为拥有读写权限），那肯定是不行的。实际上是不会发生这种情况，因为有特殊权限stick bit（粘贴位）权限，正如drwxrwxrwt中的最后一个t

**stick bit (粘贴位)就是：除非目录的属主和root用户有权限删除它，除此之外其它用户不能删除和修改这个目录**。

也就是说，在/tmp目录中，只有文件的拥有者和root才能对其进行修改和删除，其他用户则不行，避免了上面所说的问题产生。用途一般是把一个文件夹的的权限都打开，然后来共享文件，象/tmp目录一样。

**3、如何设置以上特殊权限**

```bash
setuid：chmod u+s xxx

setgid：chmod g+s xxx

stick bit：chmod o+t xxx
```

或者使用八进制方式，在原先的数字前加一个数字，三个权限所代表的进制数与一般权限的方式类似，如下:

```bash
suid   guid    stick bit
 1        1          1
```


所以：
- suid的二进制串为：100，换算十进制为：4
- guid的二进制串为：010，换算十进制为：2
- stick bit二进制串为：001，换算十进制为：1

于是也可以这样设:

```bash
setuid：chmod 4755 xxx

setgid：chmod 2755 xxx

stick bit：chmod 1755 xxx
```


最后，在一些文件设置了特殊权限后，==字母不是小写的s或者t，而是大写的S和T，那代表此文件的特殊权限没有生效，是因为你尚未给它对应用户的x权限== 

## 2.3 Linux 文件内容查看


### 管道符 |

“|”是管道命令操作符，简称管道符。利用Linux所提供的管道符“|”将两个命令隔开，管道符左边命令的输出就会作为管道符右边命令的输入。连续使用管道意味着第一个命令的输出会作为 第二个命令的输入，第二个命令的输出又会作为第三个命令的输入，依此类推。

Linux系统中使用以下命令来查看文件的内容：
```shell
    cat  由第一行开始显示文件内容
    tac  从最后一行开始显示，可以看出 tac 是 cat 的倒著写！
    nl   显示的时候，顺道输出行号！
    more 一页一页的显示文件内容
    less 与 more 类似，但是比 more 更好的是，他可以往前翻页！
    head 只看头几行
    tail 只看尾巴几行
```
你可以使用 man [命令]来查看各个命令的使用文档，如 ：man cp。

### cat
命令路径：/bin/cat 执行权限：所有用户

作用：显示文件内容

语法：

```shell
cat [-n] [文件名]
     -A  显示所有内容，包括隐藏的字符  
     -n  显示行号    

```
eg：cat  1.txt

命令路径：/bin/cat

执行权限：所有用户

作用：显示文件内容

语法：cat [-n] [文件名]

      -A  显示所有内容，包括隐藏的字符 
      -n   显示行号    

eg：检看 /etc/issue 这个文件的内容：
```shell
[root@linux01 ~]# cat /etc/issue
```

### more
命令路径：/bin/more

执行权限：所有用户

作用：分页显示文件内容

语法：
```shell
more [文件名]
space或f   显示下一页
Enter键    显示下一行
q或Q       退出
/字串      代表在这个显示的内容当中，向下搜寻『字串』这个关键字；
```

### less
一页一页翻动，==可以使用光标移动==

less运行时可以输入的命令有：

    空白键    ：向下翻动一页；
    [pagedown]：向下翻动一页；
    [pageup]  ：向上翻动一页；
    /字串     ：向下搜寻『字串』的功能；
    ?字串     ：向上搜寻『字串』的功能；
    n         ：重复前一个搜寻 (与 / 或 ? 有关！)
    N         ：反向的重复前一个搜寻 (与 / 或 ? 有关！)
    q         ：离开 less 这个程序；

### head
查看文件前面几行（默认10行）

语法：
**head [-n number] 文件**
- -n ：后面接数字，代表显示几行的意思

```shell
[root@linux01 ~]# head /etc/man.config
```

默认的情况中，显示前面 10 行！若要显示前 20 行，就得要这样：
```shell
[root@linux01 ~]# head -n 20 /etc/man.config
```

### tail
取出文件后面几行(默认10行)

tail -f      等同于--follow=descriptor，根据文件描述符进行追踪，当**文件改名或被删除，追踪停止**

tail -F    等同于--follow=name  --retry，根据文件名进行追踪，并保持重试，即该文件被删除或改名后，如果再次创建**相同的文件名，会继续追踪**

语法：
tail [-n number] 文件

选项与参数：
- -n ：后面接数字，代表显示几行的意思

eg：<br>

```bash
tail -200f nohup.out
```

## 2.4 linux查找大文件、删除目录特定文件

### 2.4.1 统计指定目录下文件大小


```bash
du -h --max-depth=1 /home/work/    仅列出/home/work目录下一级目录文件大小；
du -h --max-depth=1 /home/work/* 列出/home/work下一级目录大小及/home/work下面的所有文件大小。
```

- --max-depth=深入目录的层数

==注意：使用“**`*`**”，可以得到文件的使用空间大小==

[![统计下一级目录文件及文件夹大小](https://www.helloimg.com/images/2022/03/08/R5HldD.png)](https://pic1.xuehuaimg.com/proxy/https://files.catbox.moe/pm7h86.png)


### 2.4.2 查找指定目录下超过指定大小的文件

搜索根目录下，超过800M大小的文件，并显示查找出来文件的具体大小

```bash
find / -type f -size +800M  -print0 | xargs -0 du -h 


801M    /data/mysql/mysqldata/ecs-hdp-1-relay-bin.000004
1.1G    /data/mysql/mysqldata/mysql-bin.000004
840M    /data/software/hdp/hdp-utils/HDP-UTILS-1.1.0.21-centos7.tar.gz
6.8G    /data/software/hdp/hdp/HDP-2.6.5.0-centos7-rpm.tar.gz
1.7G    /data/software/hdp/ambari/ambari-2.6.2.2-centos7.tar.gz
```

**查找根目录下超过800M大小的文件的文件名称，包括对文件的信息（例如，文件大小、文件属性） ★★★**


```bash
find / -type f -size +800M  -print0 | xargs -0 ls -lh


-rw-r----- 1 mysql mysql 801M Aug 31 17:35 /data/mysql/mysqldata/ecs-hdp-1-relay-bin.000004
-rw-r----- 1 mysql mysql 1.1G Aug 24 03:17 /data/mysql/mysqldata/mysql-bin.000004
-rw-r--r-- 1 root  root  1.7G Apr 27 13:36 /data/software/hdp/ambari/ambari-2.6.2.2-centos7.tar.gz
-rw-r--r-- 1 root  root  6.8G Apr 27 13:37 /data/software/hdp/hdp/HDP-2.6.5.0-centos7-rpm.tar.gz
-rw-r--r-- 1 root  root  840M Apr 27 13:37 /data/software/hdp/hdp-utils/HDP-UTILS-1.1.0.21-centos7.tar.gz
-r-------- 1 root  root  128T Aug 31 17:35 /proc/kcore
```

### 2.4.3 linux下删除目录及其子目录下某类文件
Linux下，如果想要删除目录及其子目录下某类文件，比如说所有的txt文件，则可以使用下面的命令：

```
find . -name "*.txt" -type f -print -exec rm -rf {} \;

. 表示在当前目录下
```

```
-name "*.txt"
表示查找所有后缀为txt的文件
```


```
-type f
表示文件类型为一般正规文件
```


```
-print
表示将查询结果打印到屏幕上
```


```
-exec command
command为其他命令，-exec后可再接其他的命令来处理查找到的结果
```

上式中，{}表示”由find命令查找到的结果“，如上所示，find所查找到的结果放置到{}位置，-exec一直到”\;“是关键字，表示find额外命令的开始（-exec）到结束（\;），这中间的就是find命令的额外命令，上式中就是 rm -rf

## 2.5 ln软连接与硬连接

软链接，全称是软链接文件，英文叫作 symbolic link。这类文件其实非常**类似于 Windows 里的快捷方式**，这个软链接文件（假设叫 VA）的内容，其实是另外一个文件（假设叫 B）的路径和名称，当打开 A 文件时，实际上系统会根据其内容找到并打开 B 文件。

硬链接，全称叫作硬链接文件，英文名称是 hard link。这类文件比较特殊，这类文件（假设叫 A）会拥有自己的 inode 节点和名称，其 inode 会指向文件内容所在的数据块。与此同时，该文件内容所在的数据块的引用计数会加 1。当此数据块的引用计数大于等于 2 时，则表示有多个文件同时指向了这一数据块。**一个文件修改，多个文件都会生效**。当**删除其中某个文件时，对另一个文件不会有影响**，仅仅是数据块的引用计数减 1。当**引用计数为 0 时，则系统才会清除此数据块**。

## 2.5.1 建立一个硬链接

建立硬链接的命令格式是：<br>
**ln 源文件名称 硬链接文件名称**


```bash
#先通过ls看看文件信息, 注意开头的"-", 表示这是一个普通文件
[roc@roclinux ~]$ ls -l source.txt
-rw-rw-r-- 1 roc roc 14 3月   1 00:19 source.txt
 
#用ln命令建立硬链接
[roc@roclinux ~]$ ln source.txt hardsource.txt
 
#我们通过ls -i查看两个文件的inode, 发现是完全相同的, 表示它们指向的是同一数据块
[roc@roclinux ~]$ ls -il source.txt hardsource.txt
2235010 -rw-rw-r-- 2 roc roc 14 3月   1 00:19 hardsource.txt
2235010 -rw-rw-r-- 2 roc roc 14 3月   1 00:19 source.txt
```

`-i`选项表示列出每个文件的 inode 节点 ID

注意：**硬链接不允许跨分区来建立，也不允许跨文件系统来建立**，即使是同一类型的文件系统也不行，这主要是受限于 inode 指向数据块的名字空间。==**硬链接只能在同一个分区内建立**==

## 2.5.2 建立一个软链接
建立软链接的命令格式为:<br>
**ln -s 源文件名称 软链接文件名称**

```bash
#用ln -s来建立软链接
[roc@roclinux ~]$ ln -s source.txt softsource.txt
 
#查看文件i节点信息
[roc@roclinux ~]$ ls -il source.txt softsource.txt
2235009 lrwxrwxrwx 1 roc roc 10 3月   1 00:24 softsource.txt -> source.txt
2235010 -rw-rw-r-- 2 roc roc 14 3月   1 00:19 source.txt
```

我们依然使用 `ls -il` 命令查看，发现软链接文件 softsource.txt 和源文件 source.txt 的 inode 号是不一样的，这说明它们完全指向两个不同的数据块。

软链接文件的权限栏首字符为 l（L的小写字母），这也是软链接文件区别于普通文件的地方之一。

如果这个时候，我们删除了 source.txt 文件，则软链接 softsource.txt 就会变成红色字体。这表示警告，说明这是一个有问题的文件，无法找到它所标识的目标文件 source.txt。

## 2.5.3 修改软链接

**==ln -snf 【新目标目录】 【原软链接地址】==**

[![变更软连接.png](https://s6.jpg.cm/2022/07/08/PWaGn6.png)](https://www.helloimg.com/images/2020/12/23/4f7790121bf1859d6dd7e2bb17a8fd6cebdc4bc7f7a039ef.png)

## 2.5.4 建立一个目录软链接
建立软链接的命令格式为:<br>
**ln -s 源目录路径 软链接目录路径**

```bash
#尝试建立针对目录的软链接
[roc@roclinux ~]$ ln -s tempdir/ linkdir
[roc@roclinux ~]$ ls -li
总用量 4
2235009 lrwxrwxrwx 1 roc roc    8 3月   1 00:32 linkdir -> tempdir/
2235011 drwxrwxr-x 2 roc roc 4096 3月   1 00:30 tempdir
```


```bash
# tempdir里的东西
[roc@roclinux ~]$ ls -F tempdir/
linksource.txt
 
#我们通过刚才创建的软链接, 进入linkdir
[roc@roclinux ~]$ cd linkdir/
 
#如同进入tempdir一样
[roc@roclinux linkdir]$ ls -F
linksource.txt
```

> 补充：
> 
> `ls -F`  列出目录中的文件
> 
> -F参数使得ls命令显示的目录文件名之后加一个斜线（“/”）字符
> 
> 文件后面的星号（"`*`"）表示这是一个可执行程序

**==为什么 ln 不允许硬链接到目录==**

Linux 系统中的硬链接有两个限制：
1. 不能跨越文件系统。
2. 不允许普通用户对目录作硬链接。

至于第一个限制，很好理解，而第二个就不那么好理解了。

我们对任何一个目录用 ls-l 命令都可以看到其链接数至少是 2，这也说明了系统中是存在基于目录的硬链接的，而且命令 ln-d（-d选项表示针对目录建立硬链接）也允许 root 用户尝试对目录作硬链接。这些都说明了系统限制对目录进行硬链接只是一个硬性规定，并不是逻辑上不允许或技术上不可行。那么操作系统为什么要进行这个限制呢？

这是因为，如果引入了对目录的硬连接就有可能在目录中引入循环链接，那么在目录遍历的时候系统就会陷入无限循环当中。也许有人会说，符号连接不也可以引入循环链接吗，那么为什么不限制目录的符号连接呢？

原因就在于，在 Linux 系统中，每个文件（目录也是文件）都对应着一个 inode 结构，其中 inode 数据结构中包含了文件类型（目录、普通文件、符号连接文件等）的信息，也就是说，操作系统在遍历目录时可以判断出其是否是符号连接。既然可以判断出它是否是符号连接，当然就可以采取一些措施来防范进入过大过深的循环层次，于是大部分系统会规定在连续遇到 8 个符号连接后就停止遍历。但是对于硬链接，由于操作系统中采用的数据结构和算法限制，目前是不能防范这种死循环的。

基于这样的考虑，系统不允许普通用户建立目录硬链接。


# 3. 磁盘分区与挂载
## 3.1 显示当前主机目录

命令df -h

```shell
[root@localhost ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                       26G  2.9G   22G  13% /
tmpfs                 1.9G     0  1.9G   0% /dev/shm
/dev/xvda1            485M   32M  428M   7% /boot
```
## 3.2 磁盘分区
### 3.2.1 显示机器当前的磁盘:
命令fdisk -l

```shell
[root@localhost ~]# fdisk -l /dev/xvdb 

Disk /dev/xvdb: 53.7 GB, 53687091200 bytes
255 heads, 63 sectors/track, 6527 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x00000000
```
注:这里知道新增磁盘为/dev/xvdb,就直接指定了,缩减显示篇幅。

### 3.2.2 fdisk分区/dev/xvdb:
fdisk/dev/xvdb 根据帮助提示分区,这里是把/dev/xvdb分成一个区.

```bash
[root@localhost ~] fdisk /dev/xvdb
Device contains neither a valid DOS partition table, nor Sun, SGI or OSF disklabel
Building a new DOS disklabel with disk identifier 0x0adfd119.
Changes will remain in memory only, until you decide to write them.
After that, of course, the previous content won`t be recoverable.

Warning: invalid flag 0x0000 of partition table 4 will be corrected by w(rite)

WARNING: DOS-compatible mode is deprecated. It`s strongly recommended to
         switch off the mode (command 'c') and change display units to
         sectors (command 'u').

Command (m for help): m
Command action
  a  toggle a bootable flag
  b  edit bsd disklabel
  c  toggle the dos compatibility flag
  d  delete a partition
  l  list known partition types
  m  print this menu
  n  add a new partition
  o  create a new empty DOS partition table
  p  print the partition table #查看已分区数量
  q  quit without saving changes
  s  create a new empty Sun disklabel
  t  change a partition`s system id
  u  change display/entry units
  v  verify the partition table
  w  write table to disk and exit
  x  extra functionality (experts only)

Command (m for help): n #新增加一个分区
Command action
  e  extended
  p  primary partition (1-4)
p #分区类型，选择主分区
Partition number (1-4): 1 #分区号选1
First cylinder (1-6527, default 1):  #起始扇区(一般默认值)
Using default value 1
Last cylinder, +cylinders or +size{K,M,G} (1-6527, default 6527): #结束扇区(一般默认值)
Using default value 6527

Command (m for help): p #查看已分区数量

Disk /dev/xvdb: 53.7 GB, 53687091200 bytes
255 heads, 63 sectors/track, 6527 cylinders
Units = cylinders of 16065 * 512 = 8225280 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk identifier: 0x0adfd119

   Device Boot    Start   End   Blocks  Id  System
   /dev/xvdb1       1    6527  52428096 83  Linux

Command (m for help): w #写分区表，完成后退出fdisk命令
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

### 3.2.3 磁盘格式化
将/dev/xvdb1分区格式化为ext4的文件系统格式。

> centos6文件系统是ext4，因为设计较早，对于现今动辄上T的海量数据处理，性能较低。centos7文件系统是xfs，适用于海量数据。`mkfs.xfs /dev/xvdb1`

```bash
[root@localhost ~] mkfs.ext4 /dev/xvdb1 -f # -f强制格式化
mke2fs 1.41.12 (17-May-2010)
Filesystem label=
OS type: Linux
Block size=4096 (log=2)
Fragment size=4096 (log=2)
Stride=0 blocks, Stripe width=0 blocks
3276800 inodes, 13107024 blocks
655351 blocks (5.00%) reserved for the super user
First data block=0
Maximum filesystem blocks=4294967296
400 block groups
32768 blocks per group, 32768 fragments per group
8192 inodes per group
Superblock backups stored on blocks: 
     32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208, 
     4096000, 7962624, 11239424

Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done

This filesystem will be automatically checked every 28 mounts or
180 days, whichever comes first.  Use tune2fs -c or -i to override.
```

## 3.4 挂载目录
### 3.4.1 手工挂载
新建目录/data1<br>
挂载设备到目录

```shell
[root@localhost ~]du -h
[root@localhost ~]mkdir -p /data1
[root@localhost ~]mount /dev/xvdb1 /data1
[root@localhost ~]du -h
```
**umount 挂载目录(或磁盘路径) 解除挂载**<br>
-l 强制解除挂载

```bash
[root@localhost ~]du -h
[root@localhost ~]umount /data1
[root@localhost ~]du -h
```

### 3.4.2 开机自动挂载
修改/etc/fstab配置文件,末尾添加一行:

```shell
/dev/xvdb1  /data1 ext4  defaults    0 0
```

显示当前目录,已成功挂载/u01目录:

```shell
[root@localhost ~]# df -h
Filesystem       Size  Used Avail Use% Mounted on
/dev/mapper/VolGroup-lv_root
                  26G  2.9G   22G  13% /
tmpfs            1.9G     0  1.9G   0% /dev/shm
/dev/xvda1       485M   32M  428M   7% /boot
/dev/xvdb1        50G  180M   47G   1% /data1
```

# 4. Linux系统打开文件最大数量限制

进程打开的最大文件句柄数设置

## 4.1 查看进程打开文件最大限制
- cat /proc/sys/fs/file-max　　查看系统级的最大限制
- ulimit -n　　查看用户级的限制（一般是1024，向阿里云华为云这种云主机一般是65535）

![image](https://images2018.cnblogs.com/blog/874963/201807/874963-20180725164711723-604558366.png)

## 4.2 查看某个进程已经打开的文件数
![image](https://images2018.cnblogs.com/blog/874963/201807/874963-20180725164905398-1757632819.png)

## 4.3 修改限制
**临时修改**


```bash
ulimit -HSn 2048
```


**永久修改**

```bash
vi /etc/security/limits.conf
```
加入以下两行内容

```bash
kudu       soft    nproc     unlimited
impala     soft    nproc     unlimited
```

