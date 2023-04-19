[TOC]
shell脚本编程

同传统的编程语言一样，shell提供了很多特性，这些特性可以使你的shell脚本编程更为有用。

[在线shell测试网站](http://www.dooccn.com/shell/)

# 1. 创建Shell脚本
一个shell脚本通常包含如下部分：

## 首行

第一行内容在脚本的首行左侧，表示脚本将要调用的shell解释器，内容如下：

#!/bin/bash

#！符号能够被内核识别成是一个脚本的开始，这一行必须位于脚本的首行，/bin/bash是bash程序的绝对路径，在这里表示后续的内容将通过bash程序解释执行。

## 注释

注释符号# 放在需注释内容的前面，如下：

![shell_注释.png](https://i0.wp.com/i.loli.net/2019/09/19/hnJulX6SWrFgBDb.png "shell注释")
## 内容

可执行内容和shell结构

![shell_内容.png](https://i0.wp.com/i.loli.net/2019/09/19/7XsSgvhfpRa4KJc.png "[shell内容")


vim,pwd,实际上就是一个可执行的shell脚本而已，自己写的脚本，放到PATH系统环境中去了之后，就可以在任意的地方运行自定义的脚本。要求，自己写的脚本也要加.sh后缀。

免安装的方式，直接解压，然后配置环境变量。

# 2. Shell脚本的权限
一般情况下，默认创建的脚本是没有执行权限的

![shell_权限1.png](https://i0.wp.com/i.loli.net/2019/09/19/lfcaZCxmWt7hbiR.png "shell无权限")
没有权限不能执行，需要赋予可执行权限

![shell_权限2.png](https://i0.wp.com/i.loli.net/2019/09/19/7nrbZJIP2yCj9uz.png "shell有权限")

# 3 Shell脚本的执行
## 3.1 shell脚本执行方式
### 3.1.1 输入脚本的绝对路径或相对路径

```shell
/root/helloWorld.sh
./helloWorld.sh
```

### 3.1.2 bash或sh +脚本

```shell
bash /root/helloWorld.sh
sh helloWorld.sh
```

注：当脚本没有x权限时，root和文件所有者通过该方式可以正常执行。

![shell_执行1.png](https://i0.wp.com/i.loli.net/2019/09/19/PGHTaV3wyc1jErW.png "sh执行脚本")
### 3.1.3 在脚本的路径前再加". " 或source

```shell
source /root/helloWorld.sh
. ./helloWorld.sh
```

区别：第一种和第二种会新开一个bash，不同bash中的变量无法共享

但是使用. ./脚本.sh 这种方式是在同一个shell里面执行的。

![shell_执行2.png](https://i0.wp.com/i.loli.net/2019/09/19/Qm3FxsJ5YWukD1w.png ".执行脚本")

可以使用pstree查看

source eg.sh

## 3.2 shell脚本后台执行
后台运行

```shell
(1)  sh a.sh & (干扰前台)
(2)  sh a.sh 1>/root/sh.log
             2>/root/sh.err &
```

放到前台  fg 1

```shell
(3) sh a.sh 1>/dev/null 2>&1 &   (/dev/null Linux黑洞)
(4) nohup sh a.sh 1>/dev/null 2>&1 &  （nohup 变为系统级进程（忽略 HUP 信号），root用户退出，任务仍继续）
```

查看后台任务 jobs


在linux中，&和&&,|和||介绍如下：

```shell
&  表示任务在后台执行，如要在后台运行redis-server,则有  redis-server &

&& 表示前一条命令执行成功时，才执行后一条命令 ，如 echo '1‘ && echo '2'    

|  表示管道，上一条命令的输出，作为下一条命令参数，如 echo 'yes' | wc -l

|| 表示上一条命令执行失败后，才执行下一条命令，如 cat nofile || echo "fail"
```
# 4. Shell变量
变量：是shell传递数据的一种方式，用来代表每个取值的符号名。

当shell脚本需要保存一些信息时，如一个文件名或是一个数字，就把它存放在一个变量中。

## 4.1 变量设置规则：

1. 变量名称可以由字母，数字和下划线组成，但是不能以数字开头，环境变量名建议大写，便于区分。
2. 在bash中，变量的默认类型都是字符串型，如果要进行数值运算，则必须指定变量类型为数值型。 
3. 变量用等号连接值，等号左右两侧不能有空格。
4. 变量的值如果有空格，需要使用单引号或者双引号包括。

## 4.2 变量分类
Linux Shell中的变量分为用户自定义变量,环境变量，位置参数变量和预定义变量。

可以通过set命令查看系统中存在的所有变量

**系统变量**：保存和系统操作环境相关的数据。`$HOME、$PWD、$SHELL、$USER`等等

**位置参数变量**：主要用来向脚本中传递参数或数据，变量名不能自定义，变量作用固定。

**预定义变量**：是Bash中已经定义好的变量，变量名不能自定义，变量作用也是固定的。

### 4.2.1 用户自定义变量
用户自定义的变量由字母或下划线开头，由字母，数字或下划线序列组成，并且大小写字母意义不同，变量名长度没有限制。

#### 设置变量

习惯上用大写字母来命名变量。变量名以字母表示的字符开头，不能用数字。

#### 变量调用

在使用变量时，要在变量名前加上前缀“$”.

使用echo 命令查看变量值。eg:echo $A

#### 变量赋值
**1,定义时赋值：**

变量＝值

等号两侧不能有空格

eg:

STR="hello world"

A=9

 

**2, 将一个命令的执行结果赋给变量**

A='ls -la' 反引号，运行里面的命令，并把结果返回给变量A

A=$(ls -la) 等价于反引号

eg: aa=$((4+5))

bb='expr 4 + 5'

**3，将一个变量赋给另一个变量**

eg : A=$STR

#### 变量叠加

eg:#aa=123

eg:#cc="$aa"456

eg:#dd=${aa}789


<font color="red">**单引号和双引号的区别**：</font>

现象：单引号里的内容会全部输出，而双引号里的内容会有变化

原因：单引号会将所有特殊字符脱意

```bash
NUM=10    

SUM="$NUM hehe"     echo $SUM     输出10 hehe

SUM2='$NUM hehe'     echo $SUM2    输出$NUM hehe
```


#### 列出所有的变量：

```shell
set
```

#### 删除变量：

```shell
unset  NAME
```

eg :

```shell
# unset A 撤销变量 A

# readonly B=2 声明静态的变量 B=2 ，不能 unset
```
![shell_删除变量.png](https://i0.wp.com/i.loli.net/2019/09/19/RvTFObUiG7AIElx.png "删除变量")

用户自定义的变量，作用域为**当前的shell环境**。

### 4.2.2 环境变量

用户自定义变量只在当前的shell中生效，而环境变量会在当前shell和其所有子shell中生效。如果把环境变量写入相应的配置文件，那么这个环境变量就会在所有的shell中生效。

**export** 变量名=变量值   申明变量

export 变量名

作用域：当前shell以及所有的子shell

### 4.2.3 位置参数变量

|参数|备注|
|---|---|
|$n|n为数字，$0代表命令本身，`$1`-`$9`代表第一到第9个参数,十以上的参数需要用大括号包含，如${10}。|
|$*|代表命令行中所有的参数，把所有的参数看成一个整体。以`"$1 $2 … $n"`的形式输出所有参数|
|$@|代表命令行中的所有参数，把每个参数区分对待。以`"$1" "$2" … "$n"` 的形式输出所有参数|
|$#|代表命令行中所有参数的个数。添加到shell的参数个数|

- `$0`	本shell脚本文件名
- `$1`	第一个参数
- `$2`	第二个参数,依次类推

<font color="red">shift指令：</font>参数左移，每执行一次，参数序列顺次左移一个位置，$# 的值减1，用于分别处理每个参数，移出去的参数不再可用

**`$*`和 $@的区别**

$* 和 $@ 都表示传递给函数或脚本的所有参数，不被双引号" "包含时，都以`"$1" "$2" … "$n"` 的形式输出所有参数

当它们被双引号" "包含时，"$*" 会将所有的参数<font color="red">作为一个整体</font>，以`"$1 $2 … $n"`的形式输出所有参数；"$@" 会将各个参数分开，以`"$1" "$2" … "$n"` 的形式输出所有参数

shell脚本中执行测试：

![shell_参数测试.png](https://i0.wp.com/i.loli.net/2019/09/19/YgsZfRaA8m6Ec9i.png "$n、$*、$#参数测试")

输出结果：

![shell_参数结果.png](https://i0.wp.com/i.loli.net/2019/09/19/GlcUsHNoWxV2pJ9.png "参数结果")

### 4.2.4 预定义变量

|变量|备注|
|---|---|
|$?|执行上一个命令的返回值   执行成功，返回0；执行失败，返回非0（具体数字由命令决定）|
|$$|当前进程的进程号（PID），即当前脚本执行时生成的进程号|
|$!|后台运行的最后一个进程的进程号（PID），最近一个被放入后台执行的进程 &|

把程序放到后台执行，在程序后面加上 &

如果说把程序放到后台去执行，nohup pwd &


```shell
# vi pre.sh
pwd >/dev/null
echo $$

ls /etc  >/dev/null &
echo $!
```


**read命令**

read [选项] 值

read -n(字符个数) -t(等待时间，单位为秒) -s(隐藏输入)  -p(提示语句)

-s后面不用跟参数

eg:

read -t 30 -p “please input your name: ” NAME

echo $NAME

read -s -p “please input your age : ” AGE

echo $AGE

read -n 1 -p “please input your sex  [M/F]: ” GENDER

echo $GENDER

# 5. 运算符

```bash
num1=11
num2=22
sum=$num1+$num2

echo $sum
```


格式: expr m + n 或$((m+n)) 注意expr运算符间要有空格
## 5.1 expr命令：对整数型变量进行算术运算

(注意：运算符前后必须要有空格) 

expr 3 + 5   
expr 3 - 5

echo 'expr 10 / 3'           

10/3的结果为3，因为是取整
expr  3 \* 10    

\ 是转义符

计算（2 ＋3 ）×4 的值

1 .分步计算
       S='expr 2 + 3'
       expr $S \* 4

2.一步完成计算

    expr `expr 2 + 3` \* 4

    S=`expr \`expr 2 + 3\`  \* 4`

    echo $S

    或

    echo $(((2 + 3) * 4))

**$()与${}的区别**

$( )的用途和反引号``一样，用来表示优先执行的命令

eg:echo $(ls a.txt)

${ } 就是取变量了  eg：echo ${PATH}

## 5.2 $((运算内容)) 适用于数值运算(推荐)

eg: echo $((3+1*4))

# 6. 条件测试(条件判断)
### 内置test命令
内置test命令常用操作符号[]表示，将表达式写在[]中，如下：
[ expression ] 

或者：

test expression

**注意**：expression首尾都有个空格

eg: [  ] ;echo $?

测试范围：整数、字符串、文件

表达式的结果为真，则test的返回值为0，否则为非0。

当表达式的结果为真时，则变量$?的值就为0，否则为非0

 

### 字符串测试：

|命令|备注|
---|---|
|test  str1 == str2|测试字符串是否相等 =|
|test  str1 != str2|测试字符串是否不相等|
|test  str1| 测试字符串是否<font color="red">不为空</font>,不为空，true，false|
|test  -z  str1|测试字符串是否为空|

eg:

name=linzhiling

[ “$name” ] && echo ok

- `;` --- 命令连接符号

- `&&` --- 逻辑与 条件满足，才执行后面的语句

[ -z “$name” ] && echo  invalid  || echo ok

- `||` --- 逻辑或，条件不满足，才执行后面的语句

test “$name” == ”yangmi” && echo ok  || echo  invalid


**Bash Shell 判断一个变量是否为空 - 方法一**
```bash
if [ -z "$var" ]; then
      echo "\$var is empty"
else
      echo "\$var is NOT empty"
fi
```

**Bash Shell 判断一个变量是否为空 - 方法二**
```bash
## 检查 $var 是否设置使用! 即检查 expr 是否为假 ##
[ ! -z "$var" ] || echo "Empty"
[ ! -z "$var" ] && echo "Not empty" || echo "Empty"
 
[[ ! -z "$var" ]] || echo "Empty"
[[ ! -z "$var" ]] && echo "Not empty" || echo "Empty"
```



 
### 整数测试:

|命令|备注|
|---|---|
|test   int1 -eq  int2| 测试整数是否相等 equals|
|test   int1 -ge  int2| 测试int1是否 ≥ int2|
|test   int1 -gt  int2| 测试int1是否 > int2|
|test   int1 -le  int2| 测试int1是否 ≤ int2|
|test   int1 -lt  int2| 测试int1是否 < int2|
|test   int1 -ne  int2| 测试整数是否不相等|

> -eq只支持整数的比较，不支持字符串，字符串使用`=`或`==`，`if [ "$A" = "$B" ]`

eg:

test 100 -gt 100

test 100 -ge 100

如下示例两个变量值的大小比较：

![shell_变量比较大小.png](https://i0.wp.com/i.loli.net/2019/09/19/cTa5Lky9lCZ6PsK.png "变量比较大小")

-gt表示greater than大于的意思，-le表示less equal表示小于等于。

### 文件测试：

|命令|备注|
|---|---|
|test  -d  file|指定文件是否目录|
|test  -e  file|文件是否存在 exists|
|test  -f  file|指定文件是否常规文件|
|test  -L  file|文件存在并且是一个符号链接 |
|test  -r  file|指定文件是否可读|
|test  -w  file|指定文件是否可写|
|test  -x  file|指定文件是否可执行|



eg:


```bash
test -d install.log

test -r install.log

test -f xx.log ; echo $?

[ -L service.soft  ] && echo “is a link”

test -L /bin/sh ;echo $?

[ -f /root ] && echo “yes” || echo “no”
```



```bash
# 一、判断文件夹是否存在,如果文件夹不存在，创建文件夹
if [ ! -d "/myfolder" ]; then
  mkdir /myfolder
fi

# 二、判断文件,目录是否存在或者具有权限
folder="/var/www/"
# -x 参数判断 $folder 是否存在并且是否具有可执行权限
if [ ! -x "$folder"]; then
  mkdir "$folder"
fi

# 三、判断文件是否存在,不存在则创建空文件
file="/var/www/log"
# -f 参数判断 $file 是否存在
if [ ! -f "$file" ]; then
  touch "$file"
fi
```


### 多重条件测试：

**条件1** *-a* **条件2** --> 逻辑与 ==> 两个都成立，则为真

**条件1** *-o* **条件2** --> 逻辑或 ==> 只要有一个为真，则为真

**!条件** --> 逻辑非 ==> 取反

eg:

```bash
num=520

[ -n “$num” -a “$num” -ge 520 ] && echo “marry you” || echo “go on”

age=20

pathname=outlog

[ ! -d“$ pathname”] &&  echo usable || echo  used
```

# 7.流程控制语句
## if/else命令
### 1) 单分支if条件语句

```shell
if [ 条件判断式 ]
    then
程序
fi
```

或者

```shell
if [ 条件判断式 ] ; then 
    程序
fi
eg:#!/bin/sh

if  [ -x  /etc/rc.d/init.d/httpd ]

    then

    /etc/rc.d/init.d/httpd restart

fi
```


单分支条件语句需要注意几个点

- if语句使用fi结尾，和一般语言使用大括号结尾不同。
- [ 条件判断式 ]
  - 就是使用test命令判断，所以中括号和条件判断式之间必须有空格<br>
  - then后面跟符号条件之后执行的程序，可以放在[]之后，用“;”分割，也可以换行写入，就不需要"；"了。

### 2) 双分支if条件语句
如果有两个分支，就可以使用 if else 语句，它的格式为：

```bash
if  condition
then
   statement1
else
   statement2
fi
```

如果 condition 成立，那么 then 后边的 statement1 语句将会被执行；否则，执行 else 后边的 statement2 语句

例子：


```bash
#!/bin/bash

read a
read b

if (( $a == $b ))
then
    echo "a和b相等"
else
    echo "a和b不相等，输入错误"
fi
```

运行结果：<br>
10↙<br>
20↙<br>
a 和 b 不相等，输入错误

### 3) 多分支if条件语句

```shell
if [ 条件判断式1 ]
    then
        当条件判断式1成立时，执行程序1
elif  [ 条件判断式2 ]
    then      
        当条件判断式2成立时，执行程序2
...省略更多条件
else
    当所有条件都不成立时，最后执行此程序
fi
```

示例：

read -p "please input your name: " NAME

eg:

```shell
#!/bin/bash

read -p "please input your name:" NAME

#echo  $NAME

if [ “$NAME” == root ]; then

	echo "hello ${NAME},  welcome !"

elif [ $NAME == tom ]; then

	echo "hello ${NAME},  welcome !"

else

	echo "SB, get out here !"

fi
```

## case命令
case命令是一个多分支的if/else命令，case变量的值用来匹配value1,value2,value3等等。匹配到后则执行跟在后面的命令直到遇到双分号为止(;;)case命令以esac作为终止符。

格式

```bash
CMD=$1
case $CMD in
start)
       echo "starting"
       ;;
Stop)
       echo "stoping"
       ;;
*)
       echo "Usage: {start|stop}"
esac
```


## for循环

for循环命令用来在一个列表条目中执行有限次数的命令。比如，你可能会在一个姓名列表或文件列表中循环执行同个命令。for命令后紧跟一个自定义变量、一个关键字in和一个字符串列表（可以是变量）。第一次执行for循环时，字符串列表中的第一个字符串会赋值给自定义变量，然后执行循环命令，直到遇到done语句；第二次执行for循环时，会右推字符串列表中的第二个字符串给自定义变量，依次类推，直到字符串列表遍历完。

第一种：


```shell
for N in 1 2 3
do
echo $N
done
或
for N in 1 2 3; do echo $N; done
或
for N in {1..3}; do echo $N; done
```


第二种：

```shell
for ((i = 0; i <= 5; i++))
do
echo "welcome $i times"
done
或
for ((i = 0; i <= 5; i++)); do echo "welcome $i times"; done
```

练习：计算从1到100的加和。

![shell_累加.png](https://i0.wp.com/i.loli.net/2019/09/19/MqJGc6z3WIVw2yk.png "shell_1-100累加")

## while循环

while命令根据紧跟其后的命令(command)来判断是否执行while循环，当command执行后的返回值(exit status)为0时，则执行while循环语句块，直到遇到done语句，然后再返回到while命令，判断command的返回值，当得打返回值为非0时，则终止while循环。

**第一种**

```shell
while expression
do
command
…
done
```

练习：求1-10 各个数的平方和

![shell_平方和1.png](https://i0.wp.com/i.loli.net/2019/09/19/LS8TYhWDm6JgABy.png "平方和1")

**第二种**

![shell_平方和2.png](https://i0.wp.com/i.loli.net/2019/09/19/iVKnobhqYQkWlUf.png "平方和2")

# 8. nohup后台运行脚本

## 8.1 nohup和`&`后台运行

### 8.1.1 nohup
nohup命令用于不挂断地运行命令（关闭当前session不会中断改程序，只能通过kill等命令删除）。

使用nohup命令提交作业，如果使用nohup命令提交作业，那么在缺省情况下该作业的所有输出都被重定向到一个名为nohup.out的文件中，除非另外指定了输出文件。

```bash
nohup command > myout.file 2>&1 &

nohup java -jar xxx.jar > /dev/null 2>&1 &
```

### 8.1.1 `&`符号
`&`用于后台执行程序，但是关闭当前session程序也会结束

### 8.1.2 `2>&1 &`详解

bash中：

- 0 代表STDIN_FILENO 标准输入（一般是键盘）
- 1 代表STDOUT_FILENO 标准输出（一般是显示屏，准确的说是用户终端控制台）
- 2 代表STDERR_FILENO (标准错误（出错信息输出）


```bash
> 直接把内容生成到指定文件，会覆盖原来文件中的内容 [ls > test.txt],
>> 尾部追加，不会覆盖原有内容 [ls >> test.txt],
< 将指定文件的内容作为前面命令的参数 [cat < text.sh]
```

`2>&1`就是用来将标准错误2重定向到标准输出1中的。此处1前面的&就是为了让bash将1解释成标准输出而不是文件1。至于最后一个`&`，则是让bash在后台执行。

### 8.1.3 `/dev/null 2>&1`

可以把/dev/null 可以看作"黑洞". 它等价于一个只写文件. 所有写入它的内容都会永远丢失. 而尝试从它那儿读取内容则什么也读不到.

`/dev/null 2>&1`则表示吧标准输出和错误输出都放到这个“黑洞”，表示什么也不输出。


### 8.1.4 并行调用shell

默认情况下，脚本执行顺序是串行执行，可以通过 ==**&**== 符号，变成并行执行
```bash
#!/bin/bash
for N in beijing shanghai guanzhou shenzhen; do
        {
                nohup sh write.sh $N >$N.txt 2>&1
                if [ $? -eq 0 ]; then
                        runTime=$(date "+%Y-%m-%d %H:%M:%S")
                        echo "$runTime $N processing completed" >>result.txt
                fi
        } &
done

wait
echo "===测试完成==="
```

上面例子为同时执行`write.sh`脚本4次，参数分别为城市名称

## 8.2 查看后台运行的进程

### 8.2.1 jobs的使用
jobs命令用于显示Linux中的任务列表及任务状态，包括后台运行的任务。该命令可以显示任务号及其对应的进程号。其中，任务号是以普通用户的角度进行的，而进程号则是从系统管理员的角度来看的。一个任务可以对应于一个或者多个进程号。


```bash
语法： jobs(选项)(参数)

选项

 -l：显示进程号； -p：仅任务对应的显示进程号；
 -n：显示任务状态的变化； -r：仅输出运行状态（running）的任务；
 -s：仅输出停止状态（stoped）的任务。
```

常用命令： jobs -l

[![2OOcfx.png](https://z3.ax1x.com/2021/06/16/2OOcfx.png)](https://www.z4a.net/images/2021/06/16/zaqeghrjwj.png)

其中，输出信息的第一列表示任务编号，第二列表示任务所对应的进程号，第三列表示任务的运行状态，第四列表示启动任务的命令。

缺点：jobs命令只看当前终端生效的，关闭终端后，在另一个终端jobs已经无法看到后台跑得程序了，此时利用ps（进程查看命令）

### 8.2.2 ps的使用
 ps命令用于报告当前系统的进程状态。可以搭配kill指令随时中断、删除不必要的程序。ps命令是最基本同时也是非常强大的进程查看命令，使用该命令可以确定有哪些进程正在运行和运行的状态、进程是否结束、进程有没有僵死、哪些进程占用了过多的资源等等，总之大部分信息都是可以通过执行该命令得到的。
 
常用命令：ps -aux

> - a:显示所有程序
> - u:以用户为主的格式来显示
> - x:显示所有程序，不以终端机来区分


通常与nohup &配合使用，用于查看后台进程ID 配合 kill命令杀掉程序

常用命令：`ps -aux|grep test.sh| grep -v grep`

> 注：grep -v grep 用grep -v参数可以将grep命令排除掉


# 9. 字符串处理
https://www.cnblogs.com/gaochsh/p/6901809.html

字符串截取

- **按索引截取 `${varible:n1:n2}`**

截取变量varible从n1开始的n2个字符，组成一个子字符串。


只需用冒号分开来指定起始字符和子字符串长度。可以根据特定字符偏移和长度，使用另一种形式的变量扩展，来选择特定子字符串。

例子：
```bash
$ EXCLAIM=cowabunga
$ echo ${EXCLAIM:0:3}
cow
$ echo ${EXCLAIM:3:7}
abunga
```

- **按指定的字符串截取**

```bash
从左向右截取最后一个string后的字符串
${varible##*string}
从左向右截取第一个string后的字符串
${varible#*string}
从右向左截取最后一个string后的字符串
${varible%%string*}
从右向左截取第一个string后的字符串
${varible%string*}

“*”只是一个通配符可以不要
```

例子：

```bash
MYVAR=foodforthought.jpg
echo ${MYVAR##*fo}
>> rthought.jpg

MYVAR=foodforthought.jpg
echo ${MYVAR#*fo}
>> odforthought.jpg
```


## 9.1 取得字符串长度
```bash
string=abc12342341          //等号二边不要有空格
echo ${#string}             //结果11
expr length $string         //结果11
expr "$string" : ".*"       //结果11 分号二边要有空格,这里的:跟match的用法差不多
-----
echo $(expr "$string" : ".*")
```

## 9.2 字符串所在位置

index获取的下标是从1开始的
```bash
string=abc12342341
expr index $string '123'    //结果4 字符串对应的下标是从1开始的
```

```bash
str="abc"  
expr index $str "a"  # 1
expr index $str "b"  # 2
expr index $str "x"  # 0
expr index $str ""   # 0
```

## 9.3 从字符串开头到子串的最大长度

```bash
string=abc12342341
expr match $string 'abc.*3' //结果9
```
匹配的字符串为abc123423


## 9.4 字符串截取

```bash
string=abc12342341
echo ${string:4}      //2342341  从第4位开始截取后面所有字符串 
echo ${string:0:3}    //abc      从第0位开始截取后面3位
echo ${string:3:6}    //123423   从第3位开始截取后面6位
echo ${string: -4}    //2341     冒号右边有空格，截取后4位
echo ${string:(-4)}   //2341     截取后4位 
expr substr $string 3 3   //123  从第3位开始截取后面3位
```


```bash
str="abcdef"  
expr substr "$str" 1 3  # 从第一个位置开始取3个字符， abc
expr substr "$str" 2 5  # 从第二个位置开始取5个字符， bcdef
expr substr "$str" 4 5  # 从第四个位置开始取5个字符， def
  
echo ${str:2}           # 从第二个位置开始提取字符串， bcdef
echo ${str:2:3}         # 从第二个位置开始提取3个字符, bcd
echo ${str:(-6):5}        # 从倒数第二个位置向左提取字符串, abcde
echo ${str:(-4):3}      # 从倒数第二个位置向左提取6个字符, cde 
```

## 9.5 匹配显示内容


```bash
//例3中也有match和这里的match不同，上面显示的是匹配字符的长度，而下面的是匹配的内容
string=abc12342341
expr match $string '\([a-c]*[0-9]*\)'  //abc12342341
expr $string : '\([a-c]*[0-9]\)'       //abc1 
expr $string : '.*\([0-9][0-9][0-9]\)' //341 显示括号中匹配的内容
```

## 9.6 截取不匹配的内容

```bash
string=abc12342341
echo ${string#a*3}     //42341 从$string左边开始，去掉最短匹配子串
echo ${string#c*3}     //abc12342341  这样什么也没有匹配到  
echo ${string#*c1*3}   //42341 从$string左边开始，去掉最短匹配子串
echo ${string##a*3}    //41 从$string左边开始，去掉最长匹配子串    
echo ${string%3*1}     //abc12342 从$string右边开始，去掉最短匹配子串 
echo ${string%%3*1}    //abc12 从$string右边开始，去掉最长匹配子串
```


```bash
str="abbc,def,ghi,abcjkl"
echo ${str#a*c}     # 输出,def,ghi,abcjkl  一个井号(#) 表示从左边截取掉最短的匹配 (这里把abbc字串去掉）  
echo ${str##a*c}    # 输出jkl，两个井号(##) 表示从左边截取掉最长的匹配 (这里把abbc,def,ghi,abc字串去掉)
echo ${str#"a*c"}   # 输出abbc,def,ghi,abcjkl 因为str中没有"a*c"子串
echo ${str##"a*c"}  # 输出abbc,def,ghi,abcjkl 同理
echo ${str#*a*c*}   # 空
echo ${str##*a*c*}  # 空
echo ${str#d*f)     # 输出abbc,def,ghi,abcjkl,
echo ${str#*d*f}    # 输出,ghi,abcjkl 
  
echo ${str%a*l}     # abbc,def,ghi  一个百分号(%)表示从右边截取最短的匹配   
echo ${str%%b*l}    # a，两个百分号表示(%%)表示从右边截取最长的匹配  
echo ${str%a*c}     # abbc,def,ghi,abcjkl 
```

这里要注意，必须从字符串的第一个字符开始，或者从最后一个开始，可以这样记忆, 井号（#）通常用于表示一个数字，它是放在前面的；百分号（%）卸载数字的后面; 或者这样记忆，在键盘布局中，井号(#)总是位于百分号（%）的左边(即前面)  

## 9.7 匹配并且替换

${变量<font color="red">**/**</font>匹配值/替换值} 一个`/`表示替换第一个，`//`表示替换所有，当查找中出现了：`/`请加转义符`\/`表示

```bash
string=abc12342341
echo ${string/23/bb}   //abc1bb42341  替换一次
echo ${string//23/bb}  //abc1bb4bb41  双斜杠替换所有匹配
echo ${string/#abc/bb} //bb12342341 #以什么开头来匹配，根php中的^有点像
echo ${string/%41/bb}  //abc123423bb %以什么结尾来匹配，根php中的$有点像
```

```bash
str="apple, tree, apple tree"
echo ${str/apple/APPLE}   # 替换第一次出现的apple
echo ${str//apple/APPLE}  # 替换所有apple  

echo ${str/#apple/APPLE}  # 如果字符串str以apple开头，则用APPLE替换它
echo ${str/%apple/APPLE}  # 如果字符串str以apple结尾，则用APPLE替换它
```

```bash
test='c:/windows/boot.ini'
echo ${test/\//\\}
>> c:\windows/boot.ini
echo ${test//\//\\}
>> c:\windows\boot.ini

# ${变量/查找/替换值} 一个“/”表示替换第一个，“//”表示替换所有，当查找中出现了：“/”请加转义符“\/”表示
# “\”本身就是转义符号，想替换成下斜线，需要使用“\\”
```

## 9.8 比较


```bash
[[ "a.txt" == a* ]]        # 逻辑真 (pattern matching)
[[ "a.txt" =~ .*\.txt ]]   # 逻辑真 (regex matching)
[[ "abc" == "abc" ]]       # 逻辑真 (string comparision)
[[ "11" < "2" ]]           # 逻辑真 (string comparision), 按ascii值比较
```

## 9.9 连接


```bash
s1="hello"
s2="world"
echo ${s1}${s2}   # 当然这样写 $s1$s2 也行，但最好加上大括号
```
 
## 9.10 字符串删除

```bash
test='c:/windows/boot.ini'
echo ${test#/}
>> c:/windows/boot.ini
echo ${test#*/}
windows/boot.ini
echo ${test##*/}
>> boot.ini
  
echo ${test%/*}
>> c:/windows 
echo ${test%%/*}
 
#${变量名#substring正则表达式}从字符串开头开始配备substring,删除匹配上的表达式。
##${变量名%substring正则表达式}从字符串结尾开始配备substring,删除匹配上的表达式。
#注意：${test##*/},${test%/*} 分别是得到文件名，或者目录地址最简单方法。
```

**ps: shell中`#*,##*,#*,##*,% *,%% *`的含义及用法**

`file=/dir1/dir2/dir3/my.file.txt`
可以用${ }分别替换得到不同的值：
- `${file#*/}`：删掉第一个 `/` 及其左边的字符串：dir1/dir2/dir3/my.file.txt
- `${file##*/}`：删掉最后一个`/` 及其左边的字符串：my.file.txt
- `${file#*.}`：删掉第一个`.`及其左边的字符串：file.txt
- `${file##*.}`：删掉最后一个 `.` 及其左边的字符串：txt
- `${file%/*}`：删掉最后一个 `/` 及其右边的字符串：/dir1/dir2/dir3
- `${file%%/*}`：删掉第一个 `/` 及其右边的字符串：(空值)
- `${file%.*}`：删掉最后一个 `.` 及其右边的字符串：/dir1/dir2/dir3/my.file
- `${file%%.*}`：删掉第一个 `.` 及其右边的字符串：/dir1/dir2/dir3/my

记忆的方法为：
- `#`是去掉左边（键盘上`#`在`$`的左边）
- `%`是去掉右边（键盘上`%`在`$`的右边）
- 单一符号是最小匹配；两个符号是最大匹配
  - ${file:0:5}：提取最左边的 5 个字节：/dir1
  - ${file:5:5}：提取第 5 个字节右边的连续5个字节：/dir2

也可以对变量值里的字符串作替换：
- ${file/dir/path}：将第一个dir 替换为path：/path1/dir2/dir3/my.file.txt
- ${file//dir/path}：将全部dir 替换为 path：/path1/path2/path3/my.file.txt

# 10. 数组处理

## 10.1 数组定义
**数组定义法1：**

```bash
arr=(1 2 3 4 5) # 注意是用空格分开，不是逗号！！
```

**数组定义法2**：
```bash
array
array[0]="a"
array[1]="b"
array[2]="c"
```

获取数组的length（数组中有几个元素）：

`${#array[@]}`

## 10.2 数组遍历
1、遍历（For循环法）：

```
#  ${arr[@]} 不要带空格
for var in ${arr[@]};
do
    echo $var
done
```

2、遍历（带数组下标）：

```
for i in "${!arr[@]}"; do 
    printf "%s\t%s\n" "$i" "${arr[$i]}"
done
```

3、遍历（While循环法）：

```
# 中括号内与判断条件需要有空格间隔
i=0
while [ $i -lt ${#arr[@]} ]
do
    echo ${arr[$i]}
    let i++
done
```


```
i=0
while (( $i < ${#arr[@]} ))
do
    echo ${arr[$i]}
    let i++
done
```


4、遍历（for循环次数）

```
for i in {1..5}
do
   echo "Welcome $i times"
done
```

## 10.3 向函数传递数组

由于Shell对数组的支持并不号，所以这是一个比较麻烦的问题。

翻看了很多StackOverFlow的帖子，除了全局变量外，无完美解法。

这里提供一个变通的思路，我们可以在调用函数前，将数组转化为字符串。
在函数中，读取字符串，并且分为数组，达到目的。

```
fun() {
        local _arr=(`echo $1 | cut -d " "  --output-delimiter=" " -f 1-`)
        local _n_arr=${#_arr[@]}
        for((i=0;i<$_n_arr;i++));
        do  
                elem=${_arr[$i]}
                echo "$i : $elem"
        done; 
}

array=(a b c)
fun "$(echo ${array[@]})"
```



# 11. 日期date处理
## 11.1 判断周几

```bash
#主要用：
date -d YYYYMMDD +%w
#周一到周日的返回值分别是：1,2,3,4,5,6,0
```

例子：

```
YYYYMMDD=20200412
flag=`date -d ${YYYYMMDD} +%w`
if [ $flag == "0" ]; then
else
    echo "非周日，无需计算"
fi

```

```bash
WEEK_DAY=$(date +%w)
echo $WEEK_DAY
if [[ $WEEK_DAY -eq 1 || $WEEK_DAY -eq 5 ]];then
      echo "周一或者周五"
fi

```



# 12. 附录
## 定时判断某进程是否僵死

**杀死pid进程号**

```shell
#!/bin/sh
# process name
pro_name=1701
interval=1800
# while true
# do
# sleep 600
# Print process number
pid="$(ps -ef|grep $pro_name|awk '{print $2}'|head -n1)"
echo $pro_name "program process number is" $pid
# Print running time
ptime="$(ps -eo pid,etime|grep $pid|awk '{print $2}' |head -n1)"
#echo $ptime
r_time="$(echo $r_time|awk '{split($1,tab,/:/);  {print tab[1]*60 + tab[2]} }')"
echo $r_time s

# 判断程序运行的时间是否超过指定周期$interval
if test $r_time -ge $interval ;then
	echo "run time more than 30 minutes"
	kill -9 $pid
  # echo "kill success " $pid
fi
# done
```

**在yarn中杀死application**

```bash
#!/bin/bash
#spark主类名
p_name=1701
#超时时间15分钟
interval=900
#日志路径
f_log=/home/hadoop/ipsy_v2/killSpark.log
#程序当前进程号
pid=$(ps -ef | grep $p_name | grep -v grep | awk '{print$2}' | head -n1)
echo $pid "is running"
ptime=$(ps -eo pid,etime | grep $pid | awk '{print $2}' | head -n1)
echo "运行时间:" $ptime
elapsed=$(echo $ptime | awk '{split($1,tab,/:/);{print tab[1]*60+tab[2]} }')
echo "[$(date)]" $pid "运行时间:" "ptime" "," $elapsed "s" >>$f_log

#判断程序运行的时间是否超过指定周期$interval
if [ $elapsed -ge $interval ]; then
	echo "run time more than " $interval "s"
	echo "spark job:" $spark_jobs
	spark_jobs=$(yarn application -list | awk -F ' ' '{print $1}' | grep application_)
	for app_id in $spark_jobs; do
		yarn application -kill $app_id
		echo "[$(date)] kill spark job:" $app_id >>f_log
		echo "yarn application -kill:" $app_id
	done
	#kill -9 $pid
	echo "kill sucess " $pid
fi
echo " " >>$f_log
```

## 批量添加用户

**方法1**

将hosts里面第一列，ip取出
```bash
awk -F " " '{print $1}' /etc/hosts >/root/ip.txt
```

其中双引号中的表示的是两列之间的分隔符，这里是空格， $1表示第一列， >/root/ip.txt表示把得到的结果输出到文件

根据ip.txt,向里面所有ip主机添加用户（注意\`是飘号，不是单引号）
```bash
#!/bin/bash

NEW_USER='oas'

IP_PATH="/root/ip.txt"

echo "start add $NEW_USER"
for line in `cat $IP_PATH`
do
ssh root@$line "useradd $NEW_USER"
done
echo "successfully added"
```

**方法2**


```bash
#!/bin/bash

NEW_USER='oas'

HOSTS_PATH="/etc/hosts"

echo "start add $NEW_USER"
for line in `awk '{print $1}' $HOSTS_PATH`
do
ssh root@$line "useradd $NEW_USER"
done
echo "successfully added"
```



## 环境变量失效

由于编写/etc/profile文件失误，导致PATH失效，vim、mv命令均失效，可以使用一下命令强制替换环境变量，再次编辑/etc/profile
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
```

## Linux实现进程监控和守护
### supervise


1、解压安装包

下载安装包[daemontools-0.76.tar.gz](https://cr.yp.to/daemontools/install.html)
```bash
tar -xvf daemontools-0.76.tar.gz
cd admin/daemontools-0.76
chmod 1755 package
```

> 1755中的1代表stick bit (粘贴位)，就是除非目录的属主和root用户有权限删除它，除此之外其它用户不能删除和修改这个目录

2、编译安装

```bash
package/install
```

> 如果在安装过程中出现安装失败的提示：
> ```
> /usr/bin/ld: errno: TLS definition in /lib64/libc.so.6 section .tbss mismatches non-TLS reference in envdir.o
>  
> /lib64/libc.so.6: could not read symbols: Bad value
>  
> collect2: ld returned 1 exit status
>  
> make: *** [envdir] Error 1
> ```
> 是因为daemontools 需要一个补丁daemontools-0.76.errno.patch。或者修改daemontools 源代码来修补这个bug
> 
> 方法1：
> 
> vi src/daemontools-0.76.errno.patch，添加以下内容
> 
> ```bash
> diff -ur daemontools-0.76.old/src/error.h daemontools-0.76/src/error.h
> --- daemontools-0.76.old/src/error.h	2001-07-12 11:49:49.000000000 -0500
> +++ daemontools-0.76/src/error.h	2003-01-09 21:52:01.000000000 -0600
> @@ -3,7 +3,7 @@
>  #ifndef ERROR_H
>  #define ERROR_H
>  
> -extern int errno;
> +#include <errno.h>
>  
>  extern int error_intr;
>  extern int error_nomem;
> 
> ```

> ```bash
> cd src
> patch < daemontools-0.76.errno.patch
> ```
> 
> 方法2：
> 
> 在src下的conf-cc文件的第一行最后添加如下代码即可 `-include /usr/include/errno.h`

安装完成后，可以使用`ps -ef|grep svscan`确定，/etc/inittab中会自动增加如下内容

```bash
SV:123456:respawn:/command/svscanboot
```

通过strace命令你能看到系统每隔五秒会核对一下服务：

```bash
strace -p `pidof svscan`
```

安装后会自动创建/service 和/command<br>

3、创建守护任务
进入/service创建一个test目录，在目录创建一个文件，run文件中是启动要执行脚本的命令

```bash
#!/bin/bash
sh /home/student/test.sh >> /home/student/test.log 2>&1
```

> **==不要使用nohup，命令最后面不能追加`&`符号==**

在/home/student目录下创建一个脚本`test.sh`

```bash
#!/bin/bash
while true
do
    echo `date +"%Y-%m-%d %H:%M:%S"`
    sleep 3
done
```

4、启动脚本
supervise添加监控的服务非常容易，格式如下

```bash
supervise serverDir [参数]
```
serverDir启动服务shell文件所在目录，即run文件所在目录，当server挂了之后，supervise会调用serverDir目录下面的run文件来启动服务


`supervise /service/test &`

这样即使将`test.sh`这个脚本进程杀死，supervise也会再次启动这个脚本，如果想彻底kill `test.sh`，必须先kill `supervise /service/test`所在进程

### 进程检测脚本+crontab
使用如下方法进行Nginx的进程守护

1、 编写脚本

监测nginx进程，如果挂掉，则重启，否则不予干预

在/usr/local/nginx目录下创建一个 restart_nginx.sh文件， 内容如下：

```bash
#!/bin/bash
#查找nginx进程，排除掉grep命令产生的进程号，awk打印第2列值（即进程号）赋给pid变量
pid=$(ps aux | grep nginx | grep -v grep | awk '{print $2}') 
#记录当前时间
dat=$(date '+%Y-%m-%d %H:%M:%S')
#输出当前时间
echo $dat 
#输出进程号
echo $pid

#当串的长度大于0时为真(进程号不为空）
if [ -n "$pid" ]; then
	{ 
	# 说明进程还活着，不用处理
		echo ===========alive!================
	}
	else
	#否则进程挂了，需要重启
	echo ===========shutdown!start==============
	/usr/local/nginx/nginx
	sleep 2
fi
```

2、设置定时任务

(1)给予restart_nginx.sh文件可执行权限

`chmod u+x restart_nginx.sh`

(2)编辑linux定时器, 增加定时任务， 每隔2分钟执行restart_nginx.sh脚本

crontab -e

```bash
*/2 * * * * sh /usr/local/nginx/restart_nginx.sh >> /usr/local/nginx/restart_nginx.log
```


(3)重启定时器

/etc/init.d/crond restart

3、测试

手动杀死nginx进程

```bash
ps -ef | grep 'nginx' | grep -v grep| awk '{print $2}' | xargs kill -9
```

两分钟后nginx可以启动<br>
![image](https://images2018.cnblogs.com/blog/1456105/201808/1456105-20180829160529488-11443560.png)