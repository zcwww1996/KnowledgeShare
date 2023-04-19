[TOC]
# 1. 简介

awk 是一种处理文本文件的语言，是一个强大的文本分析工具。

awk 其实不仅仅是工具软件，还是一种编程语言。

awk 是以文件的一行内容为处理单位的。awk读取一行内容，然后根据指定条件判断是否处理此行内容，若此行文本符合条件，则按照动作处理文本，否则跳过此行文本，读取下一行进行判断。
回到顶部

# 2. 基本用法

condition：条件。若此行文本符合该条件，则按照 action 处理此行文本。不添加条件时则处理每一行文本；

action：动作。按照动作处理符合要求的内容。一般用于打印指定的内容信息；

　　**注意:下面的引号为英文的单引号**
## 2.1 处理指定文件的内容


```bash
awk   'condition { action }'   filename
```

## 2.2 处理某个命令的执行结果

```bash
command | awk ' condition { action }'
```

## 2.3  常用参数
### 2.3.1  F（指定字段分隔符）

默认使用空格作为分隔符。
　　
```bash
[root@localhost awk]# echo "aa bb  cc dd  ee ff" | awk  '{print $1}'
aa
[root@localhost awk]# echo "aa bb l cc dd l ee ff" | awk -F 'l' '{print $1}'
aa bb 
[root@localhost awk]# echo "aa bb  cc : dd  ee ff" | awk -F ':' '{print $1}'
aa bb  cc 
```

# 3. 变量
## 3.1  FS（字段分隔符）　

默认是空格和制表符。

$0 表示当前整行内容，$1，$2 表示第一个字段，第二个字段

```bash
[root@localhost zabbix_agentd.d]# echo "aa bb cc  dd" | awk '{ print $0}'
aa bb cc  dd
[root@localhost zabbix_agentd.d]# echo "aa bb cc  dd" | awk '{ print $1}'
aa
[root@localhost zabbix_agentd.d]# echo "aa bb cc  dd" | awk '{ print $2}'
bb
```

## 3.2 NF（当前行的字段个数）

$NF就代表最后一个字段，$(NF-1)代表倒数第二个字段

```bash
[root@localhost zabbix_agentd.d]# echo "aa bb cc  dd" | awk '{ print $NF}'
dd
[root@localhost zabbix_agentd.d]# echo "aa bb cc  dd" | awk '{ print $(NF-1)}'
cc
```


## 3.3  NR (当前处理的是第几行)

打印当前行号和当前文本内容

```bash
[root@localhost awk]# cat test.txt 
aa ss
dd ff
gg hh
[root@localhost awk]# cat test.txt | awk '{print NR")", $0}'
1) aa ss
2) dd ff
3) gg hh
```

- 逗号表示输出的变量之间用空格分隔；
- 右括号必需使用 双引号 才可以原样输出

打印指定行内容：

```bash
[root@localhost S17]# java -version 
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
[root@localhost S17]# java -version 2>&1  | awk 'NR==1 {print $0}'
java version "1.8.0_131"
```


## 3.4 FILENAME(当前文件名)

```bash
[root@localhost awk]#  awk '{print FILENAME, NR")", $0}' test.txt 
test.txt 1) aa ss
test.txt 2) dd ff
test.txt 3) gg hh

[root@localhost awk]# cat test.txt | awk '{print FILENAME, NR")", $0}'
- 1) aa ss
- 2) dd ff
- 3) gg hh
```

- **awk   '{ condition  action }'   filename 这种形式时可以打印文件名**；
- **通过 |（管道符）读取内容时打印的是`-`**

## 3.5 其他变量

- RS：行分隔符，用于分割每一行，默认是换行符。
- OFS：输出字段的分隔符，用于打印时分隔字段，默认为空格。
- ORS：输出记录的分隔符，用于打印时分隔记录，默认为换行符。
- OFMT：数字输出的格式，默认为％.6g。

# 4. 函数
## 4.1 print 和 printf

awk中同时提供了print和printf两种打印输出的函数。

**print函数**，参数可以是变量、数值或者字符串。字符串必须用双引号引用，参数用逗号分隔。如果没有逗号，参数就串联在一起而无法区分。这里，逗号的作用与输出文件的分隔符的作用是一样的，只是后者是空格而已。

**printf函数**，其用法和c语言中printf基本相似,可以格式化字符串,输出复杂时，printf更加好用，代码更易懂。

## 4.2 其他函数

```
toupper()：字符转为大写。
tolower()：字符转为小写。
length()：返回字符串长度。
substr()：返回子字符串。 
substr($1,2)：返回第一个字段，从第2个字符开始一直到结束。 
substr($1,2,3)：返回第一个字段，从第2个字符开始开始后的3个字符。 
sin()：正弦。
cos()：余弦。
sqrt()：平方根。
rand()：随机数。
```


### 4.2.1 示例

```bash
[root@localhost awk]# echo "aa bb  cc dd  ee ff" | awk  '{print toupper($1)}'
AA
[root@localhost awk]# echo "aa BB  cc dd  ee ff" | awk  '{print tolower($2)}'
bb
[root@localhost awk]# echo "aa BB  cc dd  ee ff" | awk  '{print length($2)}'
2
[root@localhost awk]# echo "asdfghj" | awk '{print substr($1,2,3)}'
sdf
```

# 5. 条件
awk 允许指定输出条件，只输出符合条件的行。

```bash
awk  ' 条件 {动作 }' 文件名
```

条件有以下几种：

## 5.1 正则表达式

**特殊字符需要转义</br>**
**/str/  两个//符号之间为包含内容**
```bash
[root@localhost awk]# cat exp.txt 
/stsvc/fms/conf/application.yml
/stsvc/sms/conf/application.yml
/stsvc/tms/conf/application.yml
/root/home/chenfan
/root/home/jhhuang
[root@localhost awk]# cat exp.txt | awk '/stsvc/ {print $0}'     包含 stsvc 的行
/stsvc/fms/conf/application.yml
/stsvc/sms/conf/application.yml
/stsvc/tms/conf/application.yml
[root@localhost awk]# cat exp.txt | awk '/stsvc\/fms/ {print $0}' 包含 stsvc/fms 的行
/stsvc/fms/conf/application.yml
```

## 5.2  布尔值判断

```bash
[root@localhost awk]# cat exp.txt | awk 'NR==2 {print $0}'　　等于第二行
/stsvc/sms/conf/application.yml
[root@localhost awk]# cat exp.txt | awk 'NR>4 {print $0}'　　大于第四行
/root/home/jhhuang
[root@localhost awk]# cat exp.txt | awk 'NR%2==1 {print $0}'　　奇数行
/stsvc/fms/conf/application.yml
/stsvc/tms/conf/application.yml
/root/home/jhhuang
```


某个字段等于具体值

```bash
[root@localhost awk]# cat test.txt 
aa ss
dd ff
gg hh
[root@localhost awk]# cat test.txt | awk ' $2=="ff" {print $0}'
dd ff
```


```bash
[root@localhost awk]# cat a.txt

-1|SCTY|TY1208-Z|20200318143500
-1|SCTY|TY1208-Z|20200318143500
0|SCTY|TY1208-Z|20200318143500
0|SCTY|TY1208-Z|20200318143500
0|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500


[root@localhost awk]# cat a.txt  | awk -F '|' '$1=="1" {print $0}'

1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
1|SCTY|TY1208-Z|20200318143500
```


## 5.3 if 语句

```bash
[root@localhost awk]# echo "aa ss dd" | awk '{ if($3 == "dd") print $0; else print "nothing"}'
aa ss dd
[root@localhost awk]# echo "aa ss dds" | awk '{ if($3 == "dd") print $0; else print "nothing"}'
nothing
```