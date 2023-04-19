[TOC]
# 路径
Linux的目录结构为树状结构，最顶级的目录为根目录 /
- 绝对路径：

路径的写法，由根目录 / 写起，例如： /usr/share/doc 这个目录。
- 相对路径：

路径的写法，不是由 / 写起，例如由 /usr/share/doc 要到 /usr/share/man 底下时，可以写成： cd ../man 这就是相对路径的写法啦！
# 处理目录的常用命令
## ls: 列出目录
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
## cd：切换目录
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
## pwd：显示目前的目录
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
## mkdir：创建一个新的目录

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

## rmdir (删除空的目录)

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
## cp (复制文件或目录)

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

## mv (移动文件与目录，或修改名称)

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

## rm: 移除文件或目录
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

**更改文件所属**

**chgrp  用户名    文件名  -R**

**chown 用户名   文件名  -R**

**-R表示递归目录下所有文件**

## chgrp:修改文件所属组群
修改文件所属组群很简单-chgrp命令，就是change group的缩写）

语法：`chgrp  组群  文件名/目录`

举例：
```shell
[root@redhat ~]# ls -l
total 8
-rw-r--r--  1 zgz groupa 0 Sep 26 05:48 filea
-rw-r--r--  1 zgz groupa 0 Sep 26 05:50 fileb

[root@redhat zgz]# chgrp groupb filea      --改变filea所属群组
[root@redhat zgz]# ls -l
total 8
-rw-r--r--  1 zgz groupb 0 Sep 26 05:48 filea
-rw-r--r--  1 zgz groupa 0 Sep 26 05:50 fileb
```
## chown:修改文件拥有者
修改文件拥有者的命令自然是chown，即change owner。chown功能很多，不仅仅能更改文件拥有者，还可以修改文件所属组群。如果需要将某一目录下的所有文件都改变其拥有者，可以使用-R参数。

语法如下：

chown [-R] 账号名称      文件/目录

chown [-R] 账号名称:组群  文件/目录

举例：
```shell
[root@redhat zgz]# ls -l
total 20
-rw-r--r--  1 zgz groupb    0 Sep 26 05:48 filea
-rw-r--r--  1 zgz groupa    3 Sep 26 05:59 fileb
drwxr-xr-x  2 zgz groupa 4096 Sep 26 06:07 zgzdir
[root@redhat zgz]# chown myy fileb --修改fileb的拥有者为myy
[root@redhat zgz]# ls -l
total 20
-rw-r--r--  1 zgz groupb    0 Sep 26 05:48 filea
-rw-r--r--  1 myy groupa    3 Sep 26 05:59 fileb
drwxr-xr-x  2 zgz groupa 4096 Sep 26 06:07 zgzdir
```
```shell
[root@redhat zgz]# chown myy:groupa filea --修改filea的拥有者为myy，并且同时修改组群为groupa
[root@redhat zgz]# ls -l
total 20
-rw-r--r--  1 myy groupa    0 Sep 26 05:48 filea
-rw-r--r--  1 myy groupa    3 Sep 26 05:59 fileb
drwxr-xr-x  2 zgz groupa 4096 Sep 26 06:07 zgzdir
```
# Linux 文件内容查看

## 管道符 |

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

## cat
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

## more
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

## less
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

## head
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

## tail
取出文件后面几行(默认10行)

tail -f      等同于--follow=descriptor，根据文件描述符进行追踪，当**文件改名或被删除，追踪停止**

tail -F    等同于--follow=name  --retry，根据文件名进行追踪，并保持重试，即该文件被删除或改名后，如果再次创建**相同的文件名，会继续追踪**

语法：
tail [-n number] 文件

选项与参数：
- -n ：后面接数字，代表显示几行的意思


# 输入输出
## read

read 内部命令被用来从标准输入读取单行数据。这个命令可以用来读取键盘输入，当使用重定向的时候，可以读取文件中的一行数据。

语法

```bash
read [-ers] [-a aname] [-d delim] [-i text] [-n nchars] [-N nchars] [-p prompt] [-t timeout] [-u fd] [name ...]
```

参数说明:

- -a 后跟一个变量，该变量会被认为是个数组，然后给其赋值，默认是以空格为分割符。
- -d 后面跟一个标志符，其实只有其后的第一个字符有用，作为结束的标志。
- -p 后面跟提示信息，即在输入前打印提示信息。
- -e 在输入的时候可以使用Tab命令补全功能。
- -n 后跟一个数字，定义输入文本的长度，很实用。
- -r 屏蔽`\`，如果没有该选项，则`\`作为一个转义字符，有的话`\`就是个正常的字符了。
- -s 安静模式，在输入字符时不再屏幕上显示，例如login时输入密码。
- -t 后面跟秒数，定义输入字符的等待时间。
- -u 后面跟fd，从文件描述符中读入，该文件描述符可以是exec新开启的。

**1. read 简单读取**

```bash
# a b c 三个变量读取123这个输入，会看做一个变量，所以只有a被赋值
[root@server ~] read a b c
123
[root@server ~] echo $a
123
[root@server ~] echo $b
 
[root@server ~] echo $c
 
# 下边的方法可以避免上边的错误
[root@server ~] read a b c
1 2 3
[root@server ~] echo $a
1
[root@server ~] echo $b
2
[root@server ~] echo $c
3
```

**2. read -a**

```bash
# 将读取的数据存到 array 数组中
[root@server ~] read -a array
aa bb cc dd ee ff gg
[root@server ~] echo ${array[@]}
aa bb cc dd ee ff gg
[root@server ~] echo ${array[0]}
aa
[root@server ~] echo ${array[5]}
ff
```

**3. read -d**

```bash
# 指定读取结束符为"/"
[root@server ~] read -d /
12345/
[root@server ~] read -d / a
12345/
[root@server ~] echo $a
12345
```

**4. read -n/N**

```bash
# 最多读取5个字符，不到5个字符时输入回车会终止读取
[root@server ~] read -n5 a
12345
[root@server ~] echo $a
12345
[root@server ~] read -n5 a
12
[root@server ~] echo $a
12
```

```bash
# 最多读取5个字符，不到5个字符时的换行符不会终止读取
[root@server ~] read -N5 a
12
34
[root@server ~]
[root@server ~] echo $a
12 34
```

模式匹配

```bash
#!/bin/bash

read -n1 -p "Do you want to continue [Y/N]?" answer
case $answer in
Y | y)
      echo "fine ,continue";;
N | n)
      echo "ok,good bye";;
*)
     echo "error choice";;

esac
exit 0
```

该例子使用了-n 选项，后接数值 1，指示 read 命令只要接受到一个字符就退出。只要按下一个字符进行回答，read 命令立即接受输入并将其传给变量，无需按回车键

**5. read -p**

```bash
# 允许在 read 命令行中直接指定一个提示
[root@server ~] read -p 请输入用户名：
请输入用户名：123
[root@server ~]
```

read -s -p / read -sp ：s选项必须在前边

-s 选项能够使 read 命令中输入的数据不显示在命令终端上

```bash
# 密码不会被显示，更安全，注意pass前面有空格
[root@server ~] read -s -p 请输入密码： pass
请输入密码
echo "\n您输入的密码是 $pass"
```

**6. read -t**

等待输入的秒数，当计时满时，read命令返回一个非零退出状态


```bash
#!/bin/bash

if read -t 5 -p "输入网站名:" website
then
    echo "你输入的网站名是 $website"
else
    echo "\n抱歉，你输入超时了。"
fi
exit 0
```

执行程序不输入，等待 5 秒后：

```
输入网站名:
抱歉，你输入超时了
```

**7. read -r**

```bash
# 禁止反斜线转义
[root@server ~] read -r a
\123
[root@server ~] echo $a
\123
[root@server ~] read a
\123
[root@server ~] echo $a
123
```

# 上传下载
## rz sz
### 安装

yum安装

`yum install lrzsz -y`
### sz下载
```shell
#下载一个文件
sz filename
#下载多个文件
sz filename1 filename2
#下载dir目录下的所有文件，不包含dir下的文件夹
sz dir/*
```
### rz上传
在命令终端输入rz回车后，就会出现文件选择对话框，选择需要上传文件，一次可以指定多个文件，上传到服务器的路径为当前执行rz命令的目录。

<font color="red">**注意**</font>：单独用rz会有两个问题：上传中断、上传文件变化（md5不同），解决办法是上传是用<font color="red">**rz -be**</font>，并且去掉弹出的对话框中“Upload files as ASCII”前的勾选。
- -b binary 用binary（二进制）的方式上传下载，不解释字符为ascii
- -e 强制escape 所有控制字符转义，比如Ctrl+x，DEL等。
- -r 使用 Crash recovery mode. 即文件传输中断会重传
- -y 表示文件已存在的时候会覆盖
## sftp

### 连接远程主机
```shell
sftp -oPort=<port> <user>@<host>
通过sftp连接<host>，端口为<port>，用户为<user>。
eg：sftp -oPort=60001 root@192.168.0.254
```
### 常用命令

1. sftp命令前面加上l表示在本地执行，比如pwd查看远程服务器当前目录，lpwd查看本地主机当前目录
2. !command是指在本机linux上执行command这个命令，比如!ls是列举linux当前目录下的东东，!rm a.txt是删除linux当前目录下的a.txt文件。在sftp> 后输入命令，默认值针对sftp服务器的， 所以执行rm a.txt删除的是sftp服务器上的a.txt文件， 而非本地的linux上的a.txt文件。

```shell
help/? 打印帮助信息。

pwd   查看远程服务器当前目录；

cd <dir>   将远程服务器的当前目录更改为<dir>；

ls -l  显示远程服务器上当前目录的文件详细列表；

ls <pattern> 显示远程服务器上符合指定模式<pattern>的文件名；

get <file> 下载指定文件<file>；

get <pattern> 下载符合指定模式<pattern>的文件。

put <file> 上传指定文件<file>；

put <pattern> 上传符合指定模式<pattern>的文件。

progress 切换是否显示文件传输进度。

mkdir <dir> 在远程服务器上创建目录；

exit/quit/bye 退出sftp。

! 启动一个本地shell。

! <commandline> 执行本地命令行

version 显示协议版本
```
# 帮助命令
## help
语法：

**命令名 --help 列举该命令的常用选项**

eg:
```shell
 cp --help
```

查看shell内置命令的帮助信息

eg:

```shell
help cd
```
内置命令，使用whereis,which,man都不能查看

## man
命令路径：/usr/bin/man

执行权限：所有用户

作用：获取命令或配置文件的帮助信息

语法：**man [命令]**

eg：
```shell
man ls    man  services
```
**（查看配置文件时，不需要配置文件的绝对路径，只需要文件名即可）**

1. 调用的是more命令来浏览帮助文档，按空格翻下一页，按回车翻下一行，按q退出。
2. 使用/加上关键的参数可直接定位搜索，  n  查找下一个，shift+n  查找上一个

eg: /-l   直接查看-l的介绍
# 网络通信命令
## ping
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
## ifconfig
命令路径：/sbin/ifconfig

**执行权限：root**

作用：查看和设置网卡网络配置

语法：

ifconfig [-a] [网卡设备标识]  
- -a：显示所有网卡信息

ifconfig [网卡设备标识] IP地址 修改ip地址

## netstat
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

![3306端口是否被占用.png](https://i0.wp.com/i.loli.net/2019/09/16/ByIDxCdL5O2K7vQ.png)

图1中主要看监控状态为LISTEN表示已经被占用，最后一列显示被服务mysqld占用，查看具体端口号，只要有如图这一行就表示被占用了。

netstat  -anp  |grep 82

查看82端口的使用情况，如图2：

[![82端口使用情况.png](https://ae01.alicdn.com/kf/Hc249dbd96efc448fadab569ba068c385t.jpg "82端口使用情况") ](https://imageproxy.pimg.tw/resize?url=https://i0.wp.com/i.loli.net/2019/09/16/WE6lc7wiT1n94RK.png)

可以看出并没有LISTEN那一行，所以就表示没有被占用。此处注意，图中显示的LISTENING并不表示端口被占用，不要和LISTEN混淆哦，查看具体端口时候，**必须要看到tcp，端口号，LISTEN那一行，才表示端口被占用了**

# 磁盘空间命令
## df命令
作用：用于查看Linux文件系统的状态信息,显示各个分区的容量、已使用量、未使用量及挂载点等信息。

语法：

df [-hkam] [挂载点]
- -h 根据磁盘空间和使用情况 以易读的方式显示 KB,MB,GB等

## du命令
作用：用于查看文件或目录的大小（磁盘使用空间）

语法：**du [-abhs] [文件名目录]**
- -a 显示目录占用的磁盘空间大小，还要显示其下目录和文件占用磁盘空间的大小
- -h 以易读的方式显示 KB,MB,GB等
- -s 统计总占有量，不要显示其下子目录和文件占用的磁盘空间大小

eg:
- `du -a /root` 　显示/root 目录下每个子文件的大小,默认单位为kb</br>
[![du -a /root](https://www.helloimg.com/images/2022/03/08/R5HipK.jpg)](https://b-ssl.duitang.com/uploads/item/201901/28/20190128174911_GePnT.jpeg)
- `du -h /root`   以K，M,G为单位显示/root 文件夹下各个子目录的大小</br>
[![du -h /root](https://b-ssl.duitang.com/uploads/item/201901/28/20190128174911_zzTVN.jpeg "du -h /root")](https://b-ssl.duitang.com/uploads/item/201901/28/20190128174911_zzTVN.jpeg "du -h /root")
- `du -sh /root`  以常用单位（K,M,G）为单位显示/root 目录的总大小</br>
[![du -sh /root](https://i0.wp.com/i.loli.net/2019/09/16/mSjsryw9kTMGxnV.jpg "du -sh /root")](https://images.weserv.nl/?url=i.loli.net/2019/09/16/mSjsryw9kTMGxnV.jpg "du -sh /root")

- `du -sh /root/*|sort -n` 统计root文件夹(目录)大小，并按文件大小排序</br>
[![du -sh /root/*|sort -n](https://s6.jpg.cm/2022/07/08/PWTVXG.jpg)](https://b-ssl.duitang.com/uploads/item/201901/28/20190128174911_jVf2Q.jpeg)

# 压缩解压缩命令
## gzip
命令路径：/bin/gzip

执行权限：所有用户

作用：压缩(解压)文件,压缩文件后缀为.gz

gzip默认只能压缩文件，<font style="background-color: yellow;">不能压缩目录；不保留原文件</font>

- -c：将压缩数据输出到标准输出中，可以用于保留源文件；
- -d：解压缩；
- -r：压缩目录(把目录下的文件单独压缩)；
- -v：显示压缩文件的信息；
- -数字：用于指定压缩等级，-1 压缩等级最低，压缩比最差；-9 压缩比最高。默认压缩比是 -6

语法：
```bash
#gzip压缩不需要指定压缩之后的压缩包名，只需指定源文件名即可
gzip install.log

#保留源文件
gzip -c ce.txt > ce.txt.gz

解压使用gzip -d或者 gunzip
```

压缩目录
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

`gzip -r`命令不会打包目录，而是把目录下所有的子文件分别压缩

在 Linux 中，打包和压缩是分开处理的。而 gzip 命令只会压缩，不能打包，所以才会出现没有打包目录，而只把目录下的文件进行压缩的情况

## zip
命令路径：/usr/bin/zip

执行权限：所有用户

作用：**压缩(解压)文件,压缩文件后缀为.zip**

语法：

zip 选项[-r]  压缩后文件名称 被文件或目录

 -r 压缩目录
 
eg：
```shell
zip -r test.zip  /test  压缩目录
如果不加-r选项，压缩后的文件没有数据。
解压使用unzip
```
## tar
命令路径：/bin/tar

执行权限：所有用户

作用：文件、目录打（解）包

语法：

tar [-zcf] 压缩后文件名  文件或目录
- -c 建立一个压缩文件的参数指令（create），后缀是.tar
- -x 解开一个压缩文件的参数指令（extract）
- -z 以gzip命令压缩/解压缩 
- -j  以bzip2命令压缩/解压缩
- -v 压缩的过程中显示文件（verbose）
- -f file 指定文件名,必选项
tar -cf   -xf     单独 压缩  解压缩
tar  -z 以gzip打包目录并压缩  文件格式.tar.gz（.tgz）
tar  -j 以bzip2打包目录并压缩  文件格式.tar.bz2

**-C 是指定你的压缩包要解压到的目录**</br>
比如：`tar -zxvf log.tar.gz -C /tmp/` 就是要解压到tmp目录下

**最常用: tar + gzip**

**<font color="red">tar -zcvf 压缩</font>**</br>
`tar -zcvf log.tar.gz /tmp/log`

**<font color="red">tar -zxvf 解压</font>**</br>
`tar -zxvf log.tar.gz -C /tmp/`

**-C 是指定你的压缩包要解压到的目录**</br>
例子要把log.tar.gz解压到tmp目录下

补充：
1. 文件路径，压缩包带文件路径
2. 源文件是保留的，不会被删除

# 文件搜索
## find     
命令路径：/bin/find

执行权限：所有用户

作用：查找文件或目录     

语法:<br>
**`find   path   -option   [ -print ]   [ -exec   -ok   command ]   {} \;`**

- **-print** 打印文件绝对路径
- **`-exec 命令2 {} \;`**  上一个命令的结果放入{}，并作为参数传递给命令2。注意：`{}`与`\;`是执行-exec的必备条件。注意：之间有空格
- **ok** 与exec作用相同，      区别在于，在执行命令之前，ok都会给出提示，让用户确认是否执行

如果没有指定搜索路径，默认将在当前目录下查找子目录与文件。并且将查找到的子目录和文件全部进行显示<br>

find 根据下列规则判断 path 和 expression，在命令列上第一个 - ( ) , ! 之前的部份为 path，之后的是 expression。如果 path 是空字串则使用目前路径，如果 expression 是空字串则使用 -print 为预设 expression

> **expression:** 
> 1. 指定的查找文件的规则(条件)，当没有指定是默认规则是‘-print’()。可指定多个规则，多个规则之间可以进行逻辑运算（or-and）.
> 2. 可在指定规则前加!(感叹号)，取规则的反义。
> 3. 规则的计算默认是从左到右，除非表达式存在圆括号。
> 4. 在规则中使用圆括号，方括号[],问号？星号*，需要转义字符(/),以阻止shell对其解释

**按名字查找**<br>
- -name 按名称查找  精准查找
- -iname 按名称查找(忽略大小写)
- -prune 忽略某个目录

eg:
```bash
# 在目录/etc中查找文件init
find  /etc  -name “init”

# 在当前目录除aa之外的子目录内搜索 txt文件
find . -path "./aa" -prune -o -name "*.txt" -print

# 在当前目录，不再子目录中，查找txt文件
方法1：find . ! -name "." -type d -prune -o -type f -name "*.txt" -print
方法2：find . -name *.txt -type f -print

# 查找 del.txt 并删除，删除前提示确认
find . -name 'del.txt' -ok rm {} \; 
# 查找 aa.txt 并备份为aa.txt.bak
find . -name 'aa.txt' -exec cp {} {}.bak \;
```

find查找中的字符匹配：
- *：匹配所有
- ？：匹配单个字符

eg:
```shell
find  /etc  -name  “init???”
```
在目录/etc中查找以init开头的，且后面有三位的文件

**按大小查找**<br>
- -size  按文件大小查找


```bash
# 查找超过1M的文件
find / -size +1M -type f -print

# 查找等于6字节的文件
find . -size 6c -print

# 查找小于32k的文件
find . -size -32k -print
```


**按文件类型查找**<br>
- -type  按文件类型查找
  - d: 目录
  - f: 一般文件
  - c: 字型装置文件
  - b: 区块装置文件
  - p: 具名贮列
  - l: 符号连结
  - s: socket
 
eg:
```shell
# 将当前目录及其子目录中的所有文件列出
find . -type f

# 显示 /var/log 和 /etc 目录中的常规文件总数
find /var/log/ -type f -print | wc -l
```

**按文件时间查找**<br>
- -amin n : 在过去 n 分钟内被读取过
- -anewer file : 比文件 file 更晚被读取过的文件
- -cmin n : 在过去 n 分钟内被修改过
- -cnewer file :比文件 file 更新的文件
- -atime n : 在过去n天内被读取过的文件
- -ctime n : 在过去n天内被创建的文件
- -mtime n ：在过去n天内被更改过的文件

**(+|-)n是表示一个十进制整数，+n表示比n大 ，-n比n小，n等于n**

```bash
# 查找2天内被更改过的文件
find . -mtime -2 -type f -print
# 查找2天前被更改过的文件
find . -mtime +2 -type f -print
　
# 查找一天内被访问的文件
find . -atime -1 -type f -print
# 查找一天前被访问的文件
find . -atime +1 -type f -print
　
# 查找一天内状态被改变的文件
find . -ctime -1 -type f -print
# 查找一天前状态被改变的文件
find . -ctime +1 -type f -print

# 查找10分钟以前状态被改变的文件
find . -cmin +10 -type f -print
```


```bash
# 将当前目录及其子目录下所有最近 20 天内更新过的文件列出
find . -ctime -20
```

```bash
# 查找比 aa.txt 新的文件
find . -newer "aa.txt" -type f -print

# 查找比 aa.txt 旧的文件
find . ! -newer "aa.txt" -type f -print

# 查找比aa.txt新，比bb.txt旧的文件
find . -newer 'aa.txt' ! -newer 'bb.txt' -type f -print
```


**按文件所属组查找**<br>
- -empty: 空的文件
- -user uname: 查找文件归属用户名为uname的文件
- -group gname: 查找文件所属组gname的文件
- -perm: 按执行权限来查找
- -nouser uname: 已经不属于uname的文件
- -nogroup 查无有效属组的文件，即文件的属组在/etc/groups中不存在
- -gid n or -group name: gid 是 n 或是 group 名称是 name


```bash
# 查找属主是www的文件
find / -user www -type f -print

# 查找属主被删除的文件
find / -nouser -type f -print

# 查找属组 mysql 的文件
find / -group mysql -type f -print

# 查找用户组被删掉的文件
find / -nogroup -type f -print
```


**find查找的基本原则：**

占用最少的系统资源，即查询范围最小，查询条件最精准

如果明确知道查找的文件在哪一个目录，就直接对指定目录查找，不查找根目录`/`


```bash
# 查找 /var/log 目录中更改时间在 7 日以前的普通文件，并在删除之前询问它们：
find /var/log -type f -mtime +7 -ok rm {} \;

# 查找 del.txt 并删除，删除前提示确认
find . -name 'del.txt' -ok rm {} \; 
# 查找 aa.txt 并备份为aa.txt.bak
find . -name 'aa.txt' -exec cp {} {}.bak \;

# 查找当前目录中文件属主具有读、写权限，并且文件所属组的用户和其他用户具有读权限的文件：
find . -type f -perm 644 -exec ls -l {} \;

# 查找系统中所有文件长度为 0 的普通文件，并列出它们的完整路径：
find / -type f -size 0 -exec ls -l {} \;
```


## which   
命令路径：/usr/bin/which

执行权限：所有用户

作用：**显示系统命令所在目录（绝对路径及别名）**

使用which命令，就可以看到某个系统命令是否存在，以及执行的到底是哪一个位置的命令
## whereis
命令路径：/usr/bin/whereis

执行权限：所有用户

作用：**搜索命令所在目录 文件所在目录  及帮助文档路径**

eg: which passwd    和   whereis  passwd  

eg:查看/etc/passwd配置文件的帮助，就用  man 5 passw

## grep
文本搜索工具，它能使用**正则表达式**搜索文本，并把匹配的行打印出来
- -i 忽略字符大小写的差别
- -l 列出文件内容符合指定的范本样式的文件名称。
- -L 列出文件内容不符合指定的范本样式的文件名称。

在文件中搜索一个单词
命令会返回一个包含“match_pattern”的文本行：
`grep match_pattern file_name`或
`grep "match_pattern" file_name`
- 在多个文件中查找：
`grep "match_pattern" file_1 file_2 file_3 ...`
- -v 输出除之外的所有行
`grep -v "match_pattern" file_name`
- -E 使用正则表达式
`grep -E "[1-9]+"`
或
`egrep "[1-9]+"`
- -c 统计文件或者文本中包含匹配字符串的行数：
`grep -c "text" file_name`
- -n 输出包含匹配字符串的行数：

**grep正则表达式元字符集（基本集）**<br>
无引号将先处理所有shell的meta。

单引号为硬转义，shell的meta在内部应无效。

双引号为软转义，大部分shellmeta无效，但$，\，`不会失效。

元字符：`?`, `+`, `{`, `|`,`(`, `)`

- ^ 锚定行的开始 如：'^grep'匹配所有以grep开头的行。
- $ 锚定行的结束 如：'grep$'匹配所有以grep结尾的行。
- . 匹配一个非换行符的字符 如：'gr.p'匹配gr后接一个任意字符，然后是p。
- `*` 匹配零个或多个先前字符 如：'*grep'匹配所有一个或多个空格后紧跟grep的行。
- `.*` 一起用代表任意字符。
- [] 匹配一个指定范围内的字符，如'[Gg]rep'匹配Grep和grep。
- [^] 匹配一个不在指定范围内的字符，如：'[^A-FH-Z]rep'匹配不包含A-R和T-Z的一个字母开头，紧跟rep的行。
- /(../) 标记匹配字符，如'/(love/)'，love被标记为1。
- /< 锚定单词的开始，
- /> 锚定单词的结束，如'grep/>'匹配包含以grep结尾的单词的行。
- x/{m/} 重复字符x，m次，如：'o/{5/}'匹配包含5个o的行。 x/{m,/} 重复字符x,至少m次，如：'o/{5,/}'匹配至少有5个o的行。
- x/{m,n/} 重复字符x，至少m次，不多于n次，如：'o/{5,10/}'匹配5--10个o的行。
- /w 匹配文字和数字字符，也就是[A-Za-z0-9_]，如：'G/w*p'匹配以G后跟零个或多个文字或数字字符，然后是p。
- /W /w的反置形式，匹配一个或多个非单词字符，如点号句号等。
- /b 单词锁定符，如: '/bgrep/b'只匹配grep。


```shell
grep "text" -n file_name
或
cat file_name | grep "text" -n

#多个文件
grep "text" -n file_1 file_2
```
- 在多级目录中对文本进行递归搜索：
```shell
grep "text" . -r -n
# .表示当前目录。
```

# 进程管理
- 进程和程序的区别：
1. 程序是静态概念，本身作为一种软件资源长期保存；而进程是程序的执行过程，它是动态概念，有一定的生命期，是动态产生和消亡的。
2. 程序和进程无一一对应关系。一个程序可以由多个进程共用；另一方面，一个进程在活动中可以有顺序地执行若干个程序。
- 进程和线程的区别：
1. 进程： 就是正在执行的程序或命令，每一个进程都是一个运行的实体，都有自己的地址空间，并占用一定的系统资源。
2. 线程：进程有独立的地址空间，线程没有；线程不能独立存在，它由进程创建；相对讲，线程耗费的cpu和内存要小于进程

## ps命令
作用：查看系统中的进程信息

语法：ps [-auxle]

常用选项
   - -a：显示所有用户的进程
   -  -u：显示用户名和启动时间
   -  -x：显示没有控制终端的进程
   -  -e：显示所有进程，包括没有控制终端的进程
   -  -f：全格式
   -  -l：长格式显示
查看系统中所有进程
```shell
# ps aux     #查看系统中所有进程，使用BSD操作系统格式，unix
# ps -le        #查看系统中所有进程，使用Linux标准命令格式
```
ps应用实例
```shell
# ps -u or ps -l  查看隶属于自己进程详细信息
# ps aux | grep sam    查看用户sam执行的进程
# ps -ef | grep init        查看指定进程信息
```

### ps aux、ps -aux、ps -ef之间的区别
*1. ps aux和ps -aux*

请注意"ps -aux"不同于"ps aux"。POSIX和UNIX的标准要求"ps -aux"打印用户名为"x"的用户的所有进程，以及打印所有将由-a选项选择的过程。如果用户名为"x"不存在，ps的将会解释为"ps aux"，而且会打印一个警告。这种行为是为了帮助转换旧脚本和习惯。它是脆弱的，即将更改，因此不应依赖。 

如果你运行ps -aux >/dev/null，那么你就会得到下面这行警告信息

Warning: bad ps syntax, perhaps a bogus '-'? See http://procps.sf.net/faq.html

**综上： 使用时两者之间直接选择ps aux**

*2. ps aux 和ps -ef*

两者的输出结果差别不大，但展示风格不同。aux是BSD风格，-ef是System V风格。这是次要的区别，一个影响使用的区别是aux会截断command列，而-ef不会。当结合grep时这种区别会影响到结果。

**综上：以上三个命令推荐使用：ps -ef**

### 查看当前占用CPU或内存最多的5个进程

1、ps命令

`ps -aux | sort -nr -k4 | head -N`

*命令详解：*

（1） head：-N可以指定显示的行数，默认显示10行。

（2） ps：参数a指代all——所有的进程，u指代userid——执行该进程的用户id，x指代显示所有程序，不以终端机来区分。ps -aux的输出格式如下：

```shell
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
root         1  0.0  0.0 191144  2560 ?        Ss   6月10 101:04 /usr/lib/systemd/systemd --switched-root --system --deserialize 21
root         2  0.0  0.0      0     0 ?        S    6月10   0:27 [kthreadd]
root         3  0.0  0.0      0     0 ?        S    6月10  27:49 [ksoftirqd/0]
root         5  0.0  0.0      0     0 ?        S<   6月10   0:00 [kworker/0:0H]
root         7  0.0  0.0      0     0 ?        S    6月10  40:21 [migration/0]
root         8  0.0  0.0      0     0 ?        S    6月10   0:00 [rcu_bh]
```

（3） sort -k4 -nr中（k代表从根据哪一个关键词排序，后面的数字4表示按照第四列排序；n指代numberic sort，根据其数值排序；r指代reverse，这里是指反向比较结果，输出时默认从小到大，反向后从大到小。）。本例中，可以看到%MEM在第4个位置，根据%MEM的数值进行由大到小的排序。-k3表示按照cpu占用率排序。

2、 top工具

命令行输入top回车，然后按下大写M按照memory排序，按下大写P按照CPU排序。

### 查看进程启动的精确时间和启动后所流逝的时间

```bash
ps -ef的输出结果，其首行如下，说明了输出的各列：

UID  PID   PPID   C STIME  TTY  TIME  CMD
```
STIME 列指的是命令启动的时间，如果在 24 小时之内启动的，则输出格式为”HH:MM”（小时：分钟），
否则就是”Mmm:SS”（月份英语单词前 3 个字母：一月的第几号），并不能直接看出 24 小时之前启动的命令的精确启动时间。

TIME 列指的命令使用的累积 CPU 时间（user+system），显示格式通常是”MMM:SS”。（分钟：秒），并不是指从命令启动开始到现在所花的时间
```bash
[hadoop@hadoop01 ~]$ ps -ef|grep SparkSubmit
hadoop    34602  34578  3 8月18 ?       06:50:50 /opt/jdk1.8.0_152/bin/java -cp /home/hadoop/installs/spark/conf/:/home/hadoop/installs/spark/jars/*:/home/hadoop/installs/hadoop/etc/hadoop/:/home/hadoop/installs/hadoop/etc/hadoop/:/home/hadoop/installs/hadoop/share/hadoop/common/lib/*:/home/hadoop/installs/hadoop/share/hadoop/common/*:/home/hadoop/installs/hadoop/share/hadoop/hdfs/:/home/hadoop/installs/hadoop/share/hadoop/hdfs/lib/*:/home/hadoop/installs/hadoop/share/hadoop/hdfs/*:/home/hadoop/installs/hadoop/share/hadoop/yarn/lib/*:/home/hadoop/installs/hadoop/share/hadoop/yarn/*:/home/hadoop/installs/hadoop/share/hadoop/mapreduce/lib/*:/home/hadoop/installs/hadoop/share/hadoop/mapreduce/*:/home/hadoop/installs/hadoop/contrib/capacity-scheduler/*.jar -Xmx6g org.apache.spark.deploy.SparkSubmit --master yarn --deploy-mode client --conf spark.driver.memory=6g --conf spark.executor.extraJavaOptions=-XX:+UseConcMarkSweepGC --class mains.StreamSocket202_jedis --num-executors 4 --executor-memory 3g --executor-cores 4 --jars /data/iptv/dependencies/kafka-clients-0.11.0.0.jar,/data/iptv/dependencies/spark-streaming-kafka-0-10_2.11-2.0.2.jar,/data/iptv/dependencies/jedis-2.9.0.jar,/data/iptv/dependencies/commons-pool2-2.4.2.jar,/data/iptv/dependencies/fastjson-1.2.47.jar /data/iptv/data_clean_new.jar WARN false Mac
```

一个进程启动的精确时间和进程启动后的运行时间：

通过`lstart`、`etime`参数
```bash
- lstart      STARTED   time the command started.  See also bsdstart, start, start_time, and stime.
- etime       ELAPSED   elapsed time since the process was started, in the form [[DD-]hh:]mm:ss.
```

==`ps -eo pid,lstart,etime,cmd | grep 进程名称`==

```bash
[hadoop@hadoop01 ~]$ ps -eo pid,lstart,etime,cmd | grep SparkSubmit

 34602 Tue Aug 18 08:54:15 2020  7-06:24:26 /opt/jdk1.8.0_152/bin/java -cp /home/hadoop/installs/spark/conf/:/home/hadoop/installs/spark/jars/*:/home/hadoop/installs/hadoop/etc/hadoop/:/home/hadoop/installs/hadoop/etc/hadoop/:/home/hadoop/installs/hadoop/share/hadoop/common/lib/*:/home/hadoop/installs/hadoop/share/hadoop/common/*:/home/hadoop/installs/hadoop/share/hadoop/hdfs/:/home/hadoop/installs/hadoop/share/hadoop/hdfs/lib/*:/home/hadoop/installs/hadoop/share/hadoop/hdfs/*:/home/hadoop/installs/hadoop/share/hadoop/yarn/lib/*:/home/hadoop/installs/hadoop/share/hadoop/yarn/*:/home/hadoop/installs/hadoop/share/hadoop/mapreduce/lib/*:/home/hadoop/installs/hadoop/share/hadoop/mapreduce/*:/home/hadoop/installs/hadoop/contrib/capacity-scheduler/*.jar -Xmx6g org.apache.spark.deploy.SparkSubmit --master yarn --deploy-mode client --conf spark.driver.memory=6g --conf spark.executor.extraJavaOptions=-XX:+UseConcMarkSweepGC --class mains.StreamSocket202_jedis --num-executors 4 --executor-memory 3g --executor-cores 4 --jars /data/iptv/dependencies/kafka-clients-0.11.0.0.jar,/data/iptv/dependencies/spark-streaming-kafka-0-10_2.11-2.0.2.jar,/data/iptv/dependencies/jedis-2.9.0.jar,/data/iptv/dependencies/commons-pool2-2.4.2.jar,/data/iptv/dependencies/fastjson-1.2.47.jar /data/iptv/data_clean_new.jar WARN false Mac
```


## lsof
链接：http://man.linuxde.net/lsof

来源：https://blog.csdn.net/xifeijian/article/details/9088137

lsof命令用于查看你进程开打的文件，打开文件的进程，进程打开的端口(TCP、UDP)。找回/恢复删除的文件。是十分方便的系统监视工具，因为lsof命令需要访问核心内存和各种文件，所以**需要root用户执行**。
- -a：列出打开文件存在的进程；
- -c<进程名>：列出指定进程所打开的文件；
- -g：列出GID号进程详情；
- -d<文件号>：列出占用该文件号的进程；
- +d<目录>：列出目录下被打开的文件；
- +D<目录>：递归列出目录下被打开的文件；
- -n<目录>：列出使用NFS的文件；
- -i<条件>：列出符合条件的进程。（4、6、协议、:端口、 @ip ）
- -p<进程号>：列出指定进程号所打开的文件；
- -u：列出UID号进程详情；
- -h：显示帮助信息；
- -v：显示版本信息。

[![lsof.jpg](https://ae01.alicdn.com/kf/Hb50e6ca980dc42da8fef3bd4637cf29e9.jpg "lsof" )](https://imageproxy.pimg.tw/resize?url=https://i0.wp.com/i.loli.net/2019/09/16/at3ymTcbVgNSxQl.jpg)

lsof输出各列信息的意义如下：
- COMMAND：进程的名称
- PID：进程标识符
- PPID：父进程标识符（需要指定-R参数）
- USER：进程所有者
- PGID：进程所属组
- FD：文件描述符，应用程序通过文件描述符识别该文件。

lsof指令的用法如下：

**lsof abc.txt 显示开启文件abc.txt的进程**

**lsof 目录名 查找谁在使用文件目录系统**

**lsof -i :22 查看22端口被哪个进程占用**

lsof -c abc 显示abc进程现在打开的文件

lsof -g gid 显示归属gid的进程情况

**lsof -p 12 看进程号为12的进程打开了哪些文件**

**lsof -u username 查看用户打开哪些文件**

**lsof -i @192.168.1.111 查看远程已打开的网络连接（连接到192.168.1.111）**

## kill
作用：关闭进程

语法：**kill [-选项] pId**
```shell
kill -9 进程号（强行关闭）  常用
kill -1 进程号（重启进程）
```
# 用户管理命令
## 用户
### useradd 添加用户

语法：**useradd [选项] 用户名**
- -m 自动建立用户家目录；
- -g 指定用户所在的组，否则会建立一个和用户名同名的组


```bash
[root@hadoop01 ~] useradd -m -g hdp test1
hdp为用户组，test1为用户名
```

### 用户密码
#### passwd 修改和创建密码

如果不加用户名则默认修改当前登录者的密码

语法：**passwd [选项] [用户名]**

```
[root@hadoop01 ~]# passwd test1
Changing password for user test1.
New password:
BAD PASSWORD: The password is shorter than 8 characters
Retype new password:
passwd: all authentication tokens updated successfully.
```

#### 设置用户不能修改密码

```bash
[root@hadoop01 ~]# passwd -l test1     //在root下，禁止test1用户修改密码的权限
Locking password for user test1.            //锁住test1不能修改密码
passwd: Success
[root@hadoop01 ~]# su test1            //切换用户
[test1@hadoop01 root]$ passwd          //修改密码
Changing password for user test1.
Changing password for test1.
(current) UNIX password:
passwd: Authentication token manipulation error  //没有权限修改密码
[test1@hadoop01 root]$
```

#### 清除密码
```bash
[root@hadoop01 ~]# passwd -d test1    //删除test1的密码
Removing password for user test1.
passwd: Success
[root@hadoop01 ~]# passwd -S test1     //查看test1的密码
test1 NP 2019-07-22 0 99999 7 -1 (Empty password.)   //密码为空
[root@hadoop01 ~]#
```

#### 设置密码失效时间
编辑`/etc/login.defs`来设定几个参数，==能对之后新建用户起作用==，而目前系统**已经存在的用户，则直接用chage来配置**

```bash
PASS_MAX_DAYS   99999
PASS_MIN_DAYS   0
PASS_MIN_LEN    5
PASS_WARN_AGE   7
```

#### chage 修改帐号和密码的有效期

- -m：密码可更改的最小天数。为零时代表任何时候都可以更改密码。
- -M：密码保持有效的最大天数。
- -w：用户密码到期前，提前收到警告信息的天数。
- -E：帐号到期的日期。过了这天，此帐号将不可用。
- -d：上一次更改的日期。
- -i：停滞时期。如果一个密码已过期这些天，那么此帐号将不可用。
- -l：例出当前的设置。由非特权用户来确定他们的密码或帐号何时过期。

查看root账号的信息

```bash
[root@hadoop01 ~] chage -l root
Last password change                    : Jul 22, 2019
Password expires                    : never
Password inactive                    : never
Account expires                        : never
Minimum number of days between password change        : 0
Maximum number of days between password change        : 99999
Number of days of warning before password expires    : 7
```



chage -M 60 test  设置密码过期时间为60天<br>
chage -I 5 test    设置密码失效时间为5天

```bash
[root@hadoop01 ~]# chage -l test1
Last password change                    : Jul 22, 2019
Password expires                    : Sep 20, 2019
Password inactive                    : Sep 25, 2019
Account expires                        : never
Minimum number of days between password change        : 0
Maximum number of days between password change        : 60
Number of days of warning before password expires    : 7
```
上述命令可以看到，在密码过期后5天，密码自动失效，这个用户将无法登陆系统了

### usermod 修改用户帐号

参数：
- -a 把用户追加到某些组中(会保留原所属组)，仅与-G选项一起使用
- -c <备注> 　修改用户帐号的备注文字。
- -d <登入目录> 　修改用户登入时的目录。
- -e <有效期限> 　修改帐号的有效期限。
- -f <缓冲天数> 　修改在密码过期后多少天即关闭该帐号。
- -g <群组> 　修改用户所属的群组。
- -G <群组> 　修改用户所属的附加群组。
- -l <帐号名称> 　修改用户帐号名称。
- -L 　锁定用户密码，使密码无效。
- -s <shell> 　修改用户登入后所使用的shell。
- -u <uid> 　修改用户ID。
- -U 　解除密码锁定。


**将newuser2添加到组staff中**
```bash
usermod -G staff newuser2
```
**将把test用户加入usertest、test等组**<br>
`-a` 会保留test用户原来的所属组
```bash
[root@hdp01 ~] usermod -aG usertest test ##多个组之间用空格隔开 

[root@hdp01 ~] id test 
uid=500(test) gid=500(test) groups=500(test),501(usertest) 
```

**修改newuser的用户名为newuser1**
```bash
usermod -l newuser1 newuser
```


### userdel 删除用户

-r 删除账号时同时删除宿主目录（remove）

```bash
userdel -r test
```

### 切换root
```shell
su - root
```
su命令和su -命令区别就是：

su只是切换了root身份，但Shell环境仍然是普通用户的Shell；而su -连用户和Shell环境一起切换成root身份了。只有切换了Shell环境才不会出现PATH环境变量错误，报command not found的错误。

su切换成root用户以后，pwd一下，发现工作目录仍然是普通用户的工作目录；而用su -命令切换以后，工作目录变成root的工作目录了。

## 用户组
### 添加组 groupadd 组名
- -g：指定新建工作组的 id；
- -r：创建系统工作组，系统工作组的组ID小于 500；
- -K：覆盖配置文件 "/ect/login.defs"；
- -o：允许添加组 ID 号不唯一的工作组。
- -f,--force: 如果指定的组已经存在，此选项将失明了仅以成功状态退出。当与 -g 一起使用，并且指定的GID_MIN已经存在时，选择另一个唯一的GID（即-g关闭）

```bash
groupadd -g 233 people 新建用户组并指定用户组people的ID为233
```

### 删除组：groupdel 组名

```bash
groupdel people
```

### 查询组：cat /etc/group

```bash
cat /etc/group | grep people
```

# Linux硬件
## CPU
总核数 = 物理CPU个数 X 每颗物理CPU的核数

总逻辑CPU数 = 物理CPU个数 X 每颗物理CPU的核数 X **超线程数**

- 查看物理CPU个数
`cat /proc/cpuinfo| grep "physical id"| sort| uniq| wc -l`

- 查看每个物理CPU中core的个数(即核数)
`cat /proc/cpuinfo| grep "cpu cores"| uniq`

- 查看逻辑CPU的个数
`cat /proc/cpuinfo| grep "processor"| wc -l`

- 查看CPU信息（型号）
`cat /proc/cpuinfo | grep name | cut -f2 -d: | uniq -c`

- 查看CPU的详细信息
`cat /proc/cpuinfo | head -20`
```shell
processor       : 0   //逻辑处理器的ID
vendor_id       : GenuineIntel
cpu family      : 6
model           : 63
model name      : Intel(R) Xeon(R) CPU E5-2630 v3 @ 2.40GHz   //CPU型号
stepping        : 2
cpu MHz         : 2394.481
cache size      : 20480 KB
physical id     : 0
siblings        : 16   //相同物理封装处理器中逻辑处理器数
core id         : 0
cpu cores       : 8    //相同物理封装处理器中物理内核数
apicid          : 0
initial apicid  : 0
fpu             : yes
fpu_exception   : yes
cpuid level     : 15
wp              : yes
flags           : fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good xtopology nonstop_tsc aperfmperf pni pclmulqdq dtes64 ds_cpl vmx smx est tm2 ssse3 fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm arat epb xsaveopt pln pts dts tpr_shadow vnmi flexpriority ept vpid fsgsbase bmi1 avx2 smep bmi2 erms invpcid
bogomips        : 4788.96
```
## 内存
- 查看内存信息
`cat /proc/meminfo`
- 内存使用情况
`free -m`

total used free shared buffers cached

Mem: 249 163 86 0 10 94

-/+ buffers/cache: 58 191

Swap: 511 0 511

参数解释：
- total 内存总数
- used 已经使用的内存数
- free 空闲的内存数
- shared 多个进程共享的内存总额
- buffers Buffer Cache和cached Page Cache 磁盘缓存的大小
- -buffers/cache (已用)的内存数:used - buffers - cached
- +buffers/cache(可用)的内存数:free + buffers + cached
- 可用的memory=free memory+buffers+cached

上面的数值是一台开发人员使用的DELL PE2850，内存为2G的服务器，其可使用内存为=217+515+826
## 硬盘
## 查看硬盘使用情况
`df -hT`
## 查看硬盘性能
`iostat -x 1 10`

[![磁盘IO测试](https://b-ssl.duitang.com/uploads/item/201901/30/20190130103023_GUsAA.jpeg "磁盘IO测试")](https://b-ssl.duitang.com/uploads/item/201901/30/20190130103023_GUsAA.jpeg "磁盘IO测试")
图解：

- Tps：该设备每秒I/O传输的次数(每秒的I/O请求)

- Blk_read/s：表求从该设备每秒读的数据块数量

- Blk_wrth/s：表示从该设备每秒写的数据块数量

