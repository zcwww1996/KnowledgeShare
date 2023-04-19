[TOC]

#### 1.合并文件命令 cat 和 getmerge
1.   hdfs文件夹里的所有文件合并 并 下载到 指定文件，getmerge方法只能将文件夹底下的全部文件合并，不支持通配符；**
2.   使用cat命令，cat支持通配符，但是它合并成的文件还是在hdfs上，只能合并完之后拉到本地。
3.   getmerge命令第二个参数是本地路径，所以不用再拉到本地；cat命令第二个参数是hdfs路径，所以需要拉下来**

使用方法:
```html
hadoop fs -getmerge hdsf目录 linux目录
```

#### 2.inux 下读取某一文件的前n行
    head -n a.txt >test.txt
  **a.txt 就是读取的文件，test.txt中存在a.txt的前n行 
如果test.txt文件不存在就会自动创建，如果存在就会覆盖以前的该文件，如果拒绝写入，那就是没写入权限，需要更改该文件的权限**
#### 3.屏幕翻页快捷键：
shift+PgUp    向前翻看,一般翻13页左右。
shift+PgDown  向后翻看,一般翻13页左右
#### 4.Spark on Yarn 查看日志

```shell
1. 查看某个job的日志：yarn logs -applicationId application_1503298640770_02302
2. 查看某个job的状态：yarn application -status application_1503298640770_02303
3. kill掉某个job：yarn application -kill application_1503298640770_0230
```
for example:
```shell
 yarn logs -applicationId application_1555408714079_787749> logs.txt
 yarn logs -applicationId application_1555408714079_787671 |more -10
```
#### 5.cat 的几种不同用法
##### 1\. 使用 stdin 输入给 cat
```
xxxx | cat - output.txt
```

本例中使用“`-`”表示stdin，用xxxx命令的输出重定向给`cat`

##### 2\. 合并文件

```
cat file1 | file2 > output.txt
```

将两个文本合并

##### 3\. 删除多余的空白行
```
cat -s input.txt
```
##### 4\. 显示行号
```
cat -n input.txt
```
#### 6.find xargs grep查找文件及文件内容
`-name`用于匹配文件名
`-path`用来匹配路径

 dj_temp目录下的所有文件：
![](index_files/01c61ca4-337b-43e2-97f8-4944a1ee60b3.png)
##### 1.在某个路径下查文件。
 在/etc下查找“*.log”的文件
```
find dj_temp -name '*.log'
```
![](index_files/ffb3bebe-09c2-483f-a0d6-59c60ecfd1cb.png)

##### 2.扩展，列出某个路径下所有文件，包括子目录。

```
find dj_temp/ -name '*'
```
![](index_files/fd0b9f29-9499-4f5d-9df2-337cec282dbc.png)

##### 3.在某个路径下查找所有包含“hello abcserver”字符串的文件。

```
find dj_temp/ -name  '*' | xargs grep 'beijing'

```

或者(将所有包含“beijing”字符串存放到beijing.csv中)

```
find dj_temp/ -name  '*' | xargs grep 'beijing'>beijing.csv
```

##### 4. 要求忽略大小写，则使用选项`-iname`。name参数后面的字符串可以使用正则表达式。
```shell
find -iname 'xxx'
```
![](index_files/0ee6b1ab-809d-4d49-a729-a465a62aab89.png)

##### 6. 匹配多个文件:
可以使用`-o`选项来表示“**OR**”，并且用“`\(`”和“`\)`”作为条件整体括起来,需要注意的是，`\(`之后和`\)`之前必须有空格
```shell
find \( -name 'a.log' -o -name 'b.log' \)
```
![](index_files/ff4665c2-66c0-4e7e-88cd-7834b82d7042.png)
##### 7. `-path`用来匹配路径
```
find ./ -path "*/doc/*"
```

如果要匹配正则表达式路径，则使用`-regex`或`-iregex`（忽略大小写）。此时已path而不是name方式搜寻。
##### 8. **指定“非”条件**
查找dj_temp目录下不包含'A.log'的所有文件及目录
```shell
find dj_temp/ ! -name 'A.log'
```
![](index_files/add8402e-62ca-43c2-aaf9-382b32f22537.png)

##### 9.过滤文件类型
find可以匹配文件类型：`-type x`，其中x可以是以下值：

*   `f`：file，文件
*   `l`：link，符号链接
*   `d`：directory，目录/文件夹
*   `c`：char，字符设备
*   `b`：block，块设备
*   `s`：socket
*   `p`：pipe

#### **过滤文件时间**

参数都是以下格式：`-选项 天数`，天数表示距离今天几天内。选项如下：

*   `-atime`：access time
*   `-mtime`：modification time
*   `-ctime`：change time（**注：**这个和mtime有啥区别？）
    需要注意的是，UNIX中并没有“create time”的概念。

比如搜索7天内访问过的所有文件：`find . -type f -atime -7`
或者是访问超过7天的文件：`find . -type f -atime +7`
也可以是恰好7天：`find . -type f -atime 7`

上述参数是以天为单位的。下面几个参数以分钟为单位：
`-amin`, `-mmin`, `-cmim`

#### **过滤文件大小**

`-size 参数`
后面的参数可以用多种单位，比如`c`（char，字节）、`k`、`M`、`G`。比如查找大于2kB的文件：
`find . -type f -size +2k`

#### **删除匹配的文件**

在find命令后面加上`-delete`参数，find会将匹配到的文件都删除掉。这个功能很有用，但是要小心用。

#### **对每一个文件执行操作**

使用`-exec`对find的每一个目标进行操作，使用“`{}`”匹配每一个文件名。比如将所有带“configure”的文件加上执行权限：
`find . -name "configure" -type f -exec chmod +x {}`


#### **查找所有空文件夹**

```
find . -type d -empty
```

#### 7.Linux 在文档中查找满足条件的行并输出到文件：

文件名称： dlog.log    输出文件： out.log

1、满足一个条件(包含  “TJ”  )的语句：

```shell
grep  “TJ”  dlog.log  > out.log
cat  dlog.log | grep "TJ" > out.log
```
2、满足两个条件中的一个条件（包含“TJ” 或者 包含“DT ”）的命令：

```shell
egrep "TJ|DT" dlog.log > out.log
grep -E "TJ|DT" dlog.log > out.log
cat  dlog.log | grep -E "TJ|DT"  > out.log
```
3、同时满足两个条件中（包含“TJ” 和 “DT ”）的命令：

```shell
grep "TJ"  dlog.log  | grep "DT"  > out.log
egrep "TJ.*DT | DT.*TJ" dlog.log > out.log
cat dlog.log | grep "TJ"  | grep "DT"  > out.log
```

### gerp -e和grep -E，以及egrep的区别
  #### 1.grep -e 只能传递一个检索内容
    grep -e pattern1 -e pattern2 filename

  #### 2.grep -E 可以传递多个内容 ，使用 | 来分割多个pattern，以此实现OR操作
    grep -E 'pattern1|pattern2' filename

  #### 3.  egrep = grep -E 可以使用基本的正则表达外, 还可以用扩展表达式


    扩展表达式:
    + 匹配一个或者多个先前的字符, 至少一个先前字符.
    ? 匹配0个或者多个先前字符.
    a|b|c 匹配a或b或c
    () 字符组, 如: love(able|ers) 匹配loveable或lovers.
    (..)(..)\1\2 模板匹配. \1代表前面第一个模板, \2代第二个括弧里面的模板.
    x{m,n} =x\{m,n\} x的字符数量在m到n个之间.
    
    egrep '^+' file   以一个或者多个空格开头的行.
    grep '^*' file   同上
    egrep '(TOM|DAN) SAVAGE' file 包含 TOM SAVAGE 和DAN SAVAGE的行.
    egrep '(ab)+' file 包含至少一个ab的行.
    egrep 'x[0-9]?' file 包含x或者x后面跟着0个或者多个数字的行.
    egrep 'fun\.$' * 所有文件里面以fun.结尾的行.
    egrep '[A-Z]+' file 至少包含一个大写字母的行.
    egrep '[0-9]' file 至少一个数字的行.
    egrep '[A-Z]...[0-9]' file 有五个字符, 第一个式大写, 最后一个是数字的行.
    egrep '[tT]est' file 包含单词test或Test的行.
    egrep 'ken sun' file 包含ken sun的行.
    egrep -v 'marry' file 不包含marry的行.
    egrep -i 'sam' file 不考虑sam的大小写,含有sam的行.
    egrep -l "dear ken" * 包含dear ken的所有文件的清单.
    egrep -n tom file 包含tom的行, 每行前面追加行号.
    egrep -s "$name" file 找到变量名$name的, 不打印而是显示退出状态. 0表示找到. 1表示表达式没找到符合要求的, 2表示文件没找到.

##### 现在需要列举该目录中所有大于200MB的子文件目录，以及该子文件目录的占用空间
```shell
du -h --max-depth=10 . | awk '{ if($1 ~ /M/){split($1, arr, "M")}; if(($1 ~ /G/) || ($1 ~ /M/ && arr[1]>20)) {printf "%-10s %s\n", $1, $2} }' | sort -n -r
```