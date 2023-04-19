[TOC]
# 1. scala方法和函数
## 1.1 调用方法和函数
Scala中的<font color="red">**+ - * / %**</font>等操作符的作用与Java一样，位操作符<font color="red">** & | ^ >> <<**</font>也一样。只是有一点特别的：这些操作符实际上是方法。例如：

a + b

是如下方法调用的简写：

a<font color="red">**.+**</font>(b)

<font color="red">**a 方法 b**</font>可以写成<font color="red">**a.方法(b)**</font>

> ps:因为方法即操作符,所以在调用方法时有时可以省略括号,而函数必须带上括号


Scala 方法是类的一部分，而函数是一个对象可以赋值给一个变量

## 1.2 定义方法和函数

**Scala 中使用 val 语句可以定义函数，def 语句定义方法**
### 1.2.1 定义方法


```scala
  /**
    * 使用def定义一个方法
    * 方法的定义：def 方法名(参数列表):返回值={方法体}
    */

  def method01(): Unit = println("this is my first method")
 
 
  /**
    * 如果没有参数、可以直接不用写()
    */
  def method02 = println("this is my first method")
```


[![定义方法](https://hbimg.huabanimg.com/ad6ca4a54fa215127f6eebf8195b285c3f6893dd312e-0JZWC7 "定义方法")](https://hbimg.huabanimg.com/ad6ca4a54fa215127f6eebf8195b285c3f6893dd312e-0JZWC7)

方法的返回值类型可以不写，编译器可以自动推断出来，但是<font color="red">对于递归函数，必须指定返回类型</font>

### 1.2.2 定义函数


**函数的定义：val/var 函数名称=(函数的参数列表)=>函数体**

```scala
  val fun01 = (a: Int) => a + 10
 
  //定义函数，有两个Int类型的参数
  val fun02 = (a: Int, b: Int) => {
    if (a > 10 || b > 10) a + b else a - b
  }
```

[![定义函数](https://hbimg.huabanimg.com/d5f30995decc3be3fb3d46fc5aa0e271cef72c0227c9-7wk3dq "定义函数")](https://hbimg.huabanimg.com/d5f30995decc3be3fb3d46fc5aa0e271cef72c0227c9-7wk3dq)


### 1.2.3 方法和函数的区别
**1. 方法不能作为单独的表达式而存在（参数为空的方法除外），而函数可以。** 如：

```
scala> //定义一个方法
scala> def m(x:Int) = 2*x
m: (x: Int)Int

scala> //定义一个函数
scala> val f = (x:Int) => 2*x
f: Int => Int = <function1>

scala> //方法不能作为最终表达式出现
scala> m
<console>:9: error: missing arguments for method m;
follow this method with `_‘ if you want to treat it as a partially applied function
              m
              ^

scala> //函数可以作为最终表达式出现
scala> f
res9: Int => Int = <function1>

scala> //定义无参方法，并可以作为最终表达式出现
scala> def m1()=1+2 
m1: ()Int

scala> m1
res10: Int = 3
```
在如上的例子中，我们首先定义了一个方法m，接着有定义了一个函数f。接着我们把函数名（函数值）当作最终表达式来用，由于f本身就是

一个对象（实现了FunctionN特质的对象），所以这种使用方式是完全正确的。但是我们把方法名当成最终表达式来使用的话，就会出错。

**2. 函数必须要有参数列表（可以为空），而方法可以没有参数列表**

```
scala> //方法可以没有参数列表
scala> def m2 = 100;
m2: Int

scala> //方法可以有一个空的参数列表
scala> def m3() = 100
m3: ()Int

scala> //函数必须有参数列表，否则报错
scala> var f1 =  => 100
<console>:1: error: illegal start of simple expression
       var f1 =  => 100
                 ^

scala> //函数也可以有一个空的参数列表
scala> var f2 = () => 100
f2: () => Int = <function0>
```
在如上的例子中，m3方法接受零个参数，所以可以省略参数列表（变为m2）。而函数不能省略参数列表

**3. 方法名是方法调用，而函数名只是代表函数对象本身**

因为保存函数字面量的变量（又称为函数名或者函数值）本身就是实现了FunctionN特质的类的对象，要调用对象的apply方法，就需要使用obj()的语法。所以**函数名后面加括号才是调用函数**。如下：
 
 
```
scala> //该方法没有参数列表
scala> m2
res11: Int = 100

scala> //该方法有一个空的参数列表
scala> m3
res12: Int = 100

scala> //得到函数自身，不会发生函数调用
scala> f2
res13: () => Int = <function0>

scala> //调用函数
scala> f2()
res14: Int = 100
```
如果你写了一个方法的名字并且该方法不带参数（没有参数列表或者无参)，该表达式的意思是：调用该方法得到最终的表达式。

因为函数可以作为最终表达式出现，如果你写下函数的名字，函数调用并不会发生，该方法自身将作为最终的表达式进行返回，如果要强制调用一个函数，你必须在函数名后面写()

**4. 方法可以自动(称之ETA扩展)或手动强制转换为函数；但是函数不可以转换成方法**

在期望出现函数的地方使用方法，该方法自动转换成函数；手动强制转换可以使用 "`方法名  _`" 转换成函数

在scala中很多高级函数，如map(),filter()等，都是要求提供一个函数作为参数。但是为什么我们可以提供一个方法呢？就像下面这样：

```
scala> val myList = List(3,56,1,4,72)
myList: List[Int] = List(3, 56, 1, 4, 72)

scala> // map()参数是一个函数
scala> myList.map((x) => 2*x)
res15: List[Int] = List(6, 112, 2, 8, 144)

scala> //尝试给map()函提供一个方法作为参数
scala> def m4(x:Int) = 3*x
m4: (x: Int)Int

scala> //正常执行
scala> myList.map(m4)
res17: List[Int] = List(9, 168, 3, 12, 216)
```
这是因为，如果期望出现函数的地方我们提供了一个方法的话，该方法就会自动被转换成函数。该行为被称为**ETA expansion**。

如果我们直接把一个方法赋值给变量会报错。如果我们指定变量的类型就是函数，那么就可以通过编译，如下：
```
scala> def m(x:Int) = x+1
m4: (x: Int)Int

scala> //不期望出现函数的地方，方法并不会自动转换成函数
// 直接把一个方法赋值给变量会报错
scala> val f1 = m
<console>:8: error: missing arguments for method m;
follow this method with `_‘ if you want to treat it as a partially applied function
       val f1 = m
                ^
 
scala> //期望出现函数的地方，我们可以使用方法
// 指定变量的类型就是函数，那么就可以通过编译
scala>  val f2:(Int)=>Int = m
f2: Int => Int = <function1>

当然我们也可以强制把一个方法转换给函数，这就用到了scala中的部分应用函数：
scala>  val f2 = m _
f2: Int => Int = <function1>
```

**5. 传名参数本质上是个方法**

传名参数实质上是一个参数列表为空的方法，正是因此你才可以使用名字调用而不用添加()，如下
```scala
 //使用两次‘x’，意味着进行了两次方法调用
scala> def m1(x: => Int)=List(x,x)
m1: (x: => Int)List[Int]
```
如上代码实际上定义了一个方法m1，m1的参数是个传名参数（方法）。由于对于参数为空的方法来说，方法名就是方法调用，所以List(x,x)实际上是进行了两次方法调用。

```
scala> import util.Random
import util.Random

scala> val r = new Random()
r: scala.util.Random = scala.util.Random@d4c330b

scala> //因为方法被调用了两次，所以两个值不相等

scala> m1(r.nextInt)
res21: List[Int] = List(-159079888, -453797380)
```

如果你在方法体部分缓存了传名参数（函数），那么你就缓存了值（因为x函数被调用了一次）

```
 //把传名参数代表的函数缓存起来
scala> def m1(x: => Int) ={val y=x;List(y,y)}
m1: (x: => Int)List[Int]

scala> m1(r.nextInt)
res22: List[Int] = List(-1040711922, -1040711922)
```

能否在函数体部分引用传名参数所代表的方法呢，是可以的(缓存的是传名参数所代表的方法)。

```
scala> def m1(x: => Int)={val y=x _;List(y(),y())}
m1: (x: => Int)List[Int]

scala> m1(r.nextInt)
res23: List[Int] = List(1677134799, 180926366)
```


### 1.2.4 方法和函数的使用

在函数式编程语言中，<font color="red">**函数**</font>是“头等公民”，它<font color="red">**可以像任何其他数据类型一样被传递和操作**</font>

案例：**首先定义一个方法，再定义一个函数，然后将函数传递到方法里面**
[![将函数传递到方法里面](https://hbimg.huabanimg.com/fcacd7f850ccf6b8a868123d749908950b3fc53a4e0d-OFFYcd "将函数传递到方法里面")](https://hbimg.huabanimg.com/fcacd7f850ccf6b8a868123d749908950b3fc53a4e0d-OFFYcd)

```scala
object MethodAndFunctionTest {

  //定义一个方法

  //方法m2参数要求是一个函数，函数的参数必须是两个Int类型
  //返回值类型也是Int类型

  def m1(f: (Int, Int) => Int) : Int = {

    f(2, 6)

  }

  

  //定义一个函数f1，参数是两个Int类型，返回值是一个Int类型

  val f1 = (x: Int, y: Int) => x + y

  //再定义一个函数f2

  val f2 = (m: Int, n: Int) => m * n

  
  //main方法

  def main(args: Array[String]) {

    //调用m1方法，并传入f1函数

    val r1 = m1(f1)

    println(r1)

  

    //调用m1方法，并传入f2函数

    val r2 = m1(f2)

    println(r2)

  }

}
```

### 1.2.5 将方法转换成函数（神奇的下划线）
[![方法转函数](https://hbimg.huabanimg.com/73723c31e0c6264b365d66b9fa05c79c866e422c3799-payqTw "方法转函数")](https://hbimg.huabanimg.com/73723c31e0c6264b365d66b9fa05c79c866e422c3799-payqTw)

# 2. scala shell命令

```bash
scala> :help
All commands can be abbreviated, e.g., :he instead of :help.
:edit <id>|<line>        edit history
:help [command]          print this summary or command-specific help
:history [num]           show the history (optional num is commands to show)
:h? <string>             search the history
:imports [name name ...] show import history, identifying sources of names
:implicits [-v]          show the implicits in scope
:javap <path|class>      disassemble a file or class name
:line <id>|<line>        place line(s) at the end of history
:load <path>             interpret lines in a file
:paste [-raw] [path]     enter paste mode or paste a file
:power                   enable power user mode
:quit                    exit the interpreter
:replay [options]        reset the repl and replay all previous commands
:require <path>          add a jar to the classpath
:reset [options]         reset the repl to its initial state, forgetting all session entries
:save <path>             save replayable session to a file
:sh <command line>       run a shell command (result is implicitly => List[String])
:settings <options>      update compiler options, if possible; see reset
:silent                  disable/enable automatic printing of results
:type [-v] <expr>        display the type of an expression without evaluating it
:kind [-v] <expr>        display the kind of expression's type
:warnings                show the suppressed warnings from the most recent line which had any
```

> 经常将程序片段直接黏贴到spark-shell里，会遇到**多行输入**的异常，可按以下方法解决
> 
> 1. scla-shell里直接输入`:paste`命令，黏贴后结束按ctrl+D
> 2. scala-shell里通过`:load` 命令

# 3. 代码示例
## 3.1 WordCount
```scala
object WordCount {

  def main(args: Array[String]): Unit = {

    val lines = Array("hello tom tom hello jerry", "hello tom jerry", "hello tom hello hello", "hello jerry")

    //将每一行进行切分，然后压平
    val words: Array[String] = lines.flatMap(l => l.split(" "))

    //将单词和一组合在一起
    val wordAndOne: Array[(String, Int)] = words.map(w => (w, 1))

    //按单词进行分组
    val grouped: Map[String, Array[(String, Int)]] = wordAndOne.groupBy(t => t._1)

    //计数
    val summed: Map[String, Int] = grouped.mapValues(a => a.length)

    //排序
    val sorted: List[(String, Int)] = summed.toList.sortBy(t => -t._2)

    //打印结果
    println(sorted)
```

## 3.2 ScalaIO


```scala
import java.io.{File, PrintWriter}
import scala.io.Source

object ScalaIO {
  def main(args: Array[String]): Unit = {
    val pathStr = "d:\\citycode.txt"
    val filePath = new File(pathStr)

    val source = Source.fromFile(filePath)
    val source1 = Source.fromFile("d:\\citycode.txt")
    val source2 = Source.fromFile(filePath, "UTF-8")
    val source3 = Source.fromURL("https://www.baidu.com/", "UTF-8")
    val source4 = Source.fromString("i like strawberry")


    //读取文件中所有行,返回一个迭代器
    val lineIterator = source.getLines()
    lineIterator.foreach(println)

    //读取文件中所有行,toArray返回一个数组
    val lineArray = source.getLines().toArray
    for (ele <- lineArray) {
      println("line: " + ele)
    }

    //读取文件中所有行,mkstring返回一个字符串
    val lineString = source.getLines().mkString //文件不能过大


    //将文件内容以空格分割,返回一个数组
    val arrayTokens = source.mkString.split("\\s+")

    source.close() //关闭流


    //写文件
    val out = new PrintWriter("D:\\appleWrite.txt")
    for (i <- 1 to 10) {
      // write 不换行，println换行
      // out.write(s"line numbers: $i")
      out.println(s"line numbers: $i")
    }
    out.close() //关闭打印流
  }
}
```

## 3.2 Scala加载环境变量（Properties）

```Scala
// 单个变量
val filePath = System.getProperty("user.dir")
println("filePath: " + filePath)

// 全部变量
val properties = System.getProperties
```

**Scala遍历Properties**

必须导入<font style="color: red;">**scala.collection.JavaConversions._**</font>

```
import scala.collection.JavaConversions._

    val properties = System.getProperties

    println("----------------方法1-------------------------")

    for ((k, v) <- properties) {
      println("(k,v): " + k + "===" + v)
    }
    println("----------------方法1-------------------------")
    
    
  println("==================方法2=========================")
    for (entry <- properties.entrySet) {
      val key = entry.getKey
      val value = entry.getValue
      System.out.println(s"(k-v): $key --> $value")
    }
  println("==================方法2=========================")
```

# 4. Scala 内部执行Linux命令

必须导入<font style="color: red;">**import sys.process._**</font>

```scala
import sys.process._
//shell命令最后加上.!表示执行命令，也可是把执行结果赋值给一个不可变变量
//.!返回结果为int，0表示成功，.!!返回结果为打印的内容，为string

"ls -l".! //执行命令，并把结果打印到控制台上

val list = "ls -la".!! //执行命令，并把结果赋值给list

val sh = ("ls" #| "grep .txt").!! //不能在命令表达式中直接用管道，必须用 #| 声明

import java.io.File
sh.#>(new File("./text1.txt")).! //把命令执行结果输出到一个文件中，必须用 new java.io.File("")封装，文件是重写模式
```

## 4.1 scala.sys.process的四种形式
### 4.1.1 `run` 直接执行命令
`run`：最通用的方法，它立即返回一个scala.sys.process.Process，并且**外部命令同时执行，无返回值**。

```scala
scala> "ls -al".run
total 1905000
drwxrwx---  21 zhangchao ss_deploy       4096 Dec 20 18:12 .
drwxr-xr-x+ 19 root      root            4096 Aug  5 11:04 ..
-rw-r--r--   1 zhangchao ss_deploy  148729712 Dec 14 11:47 000034_0.tar.gz
-rw-------   1 zhangchao ss_deploy      46935 Dec 28 18:48 .bash_history
-rw-r--r--   1 zhangchao ss_deploy         18 Nov 20  2015 .bash_logout
-rw-r--r--   1 zhangchao ss_deploy        210 Jun 18  2021 .bash_profile
-rw-r--r--   1 zhangchao ss_deploy       2702 Jun  2  2021 .bashrc
-rw-r--r--   1 zhangchao ss_deploy 1305996268 Dec 20 18:11 wlw_uid_20200212.csv
-rw-r--r--   1 zhangchao ss_deploy  426020874 Dec 20 18:13 wlw_uid_20200212.csv.gz
-rw-r--r--   1 zhangchao ss_deploy        150 Aug 17 16:32 words.txt
res0: scala.sys.process.Process = scala.sys.process.ProcessImpl$SimpleProcess@34dec477
```


### 4.1.2 `.!` 执行命令，返回退出代码
**`.!`：阻塞，直到所有外部命令退出，并返回执行链中最后一个的退出代码**

```Scala
scala> import sys.process._
import sys.process._
 
scala> "ls -al".!
total 1905000
drwxrwx---  21 zhangchao ss_deploy       4096 Dec 20 18:12 .
drwxr-xr-x+ 19 root      root            4096 Aug  5 11:04 ..
-rw-r--r--   1 zhangchao ss_deploy  148729712 Dec 14 11:47 000034_0.tar.gz
-rw-------   1 zhangchao ss_deploy      46935 Dec 28 18:48 .bash_history
-rw-r--r--   1 zhangchao ss_deploy         18 Nov 20  2015 .bash_logout
-rw-r--r--   1 zhangchao ss_deploy        210 Jun 18  2021 .bash_profile
-rw-r--r--   1 zhangchao ss_deploy       2702 Jun  2  2021 .bashrc
-rw-r--r--   1 zhangchao ss_deploy 1305996268 Dec 20 18:11 wlw_uid_20200212.csv
-rw-r--r--   1 zhangchao ss_deploy  426020874 Dec 20 18:13 wlw_uid_20200212.csv.gz
-rw-r--r--   1 zhangchao ss_deploy        150 Aug 17 16:32 words.txt
res1: Int = 0
```

**命令执行后，有个返回值(int，0表示成功)，可以赋值给一个常量**

```Scala
scala> val exitCode = "ls -al".!
总用量 24
drwxrwxr-x 6 1001 1001 4096 3月   4 2016 .
drwxr-xr-x 4 root root 4096 9月  13 15:53 ..
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 bin
drwxrwxr-x 4 1001 1001 4096 3月   4 2016 doc
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 lib
drwxrwxr-x 3 1001 1001 4096 3月   4 2016 man
exitCode: Int = 0

println(exitCode)
```

判断hdfs目录是否存在
```Scala
val inputPath = args(0)

    val exitCode = s"hdfs dfs -test -e $inputPath".!
    if (0 == exitCode) {
      println(s"目录$inputPath/存在")
      val parquetFile: DataFrame = spark.read.parquet(inputPath)
      parquetFile.show()
    } else {
      println("目录" + inputPath + "不存在")
    }
```




**`Seq("ls","-al")`  会自动拼接成  `ls -al`**

```
scala> Seq("ls","-al").!
总用量 24
drwxrwxr-x 6 1001 1001 4096 3月   4 2016 .
drwxr-xr-x 4 root root 4096 9月  13 15:53 ..
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 bin
drwxrwxr-x 4 1001 1001 4096 3月   4 2016 doc
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 lib
drwxrwxr-x 3 1001 1001 4096 3月   4 2016 man
res2: Int = 0

scala> Process("ls -al").!
总用量 24
drwxrwxr-x 6 1001 1001 4096 3月   4 2016 .
drwxr-xr-x 4 root root 4096 9月  13 15:53 ..
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 bin
drwxrwxr-x 4 1001 1001 4096 3月   4 2016 doc
drwxrwxr-x 2 1001 1001 4096 3月   4 2016 lib
drwxrwxr-x 3 1001 1001 4096 3月   4 2016 man
res3: Int = 0
```

### 4.1.3 `.!!` 直接打印结果

**`!!`：阻塞直到所有外部命令都退出，并返回一个拼接命令结果的String**


```
val result = "ls -l".!!

#############
result: String =
"bin
boot
data01
data02
dev
etc
home
hs_err_pid10122.log
hs_err_pid11261.log
hs_err_pid11785.log
hs_err_pid12136.log
hs_err_pid1213.log
hs_err_pid14154.log
hs_err_pid1448.log
……
```

### 4.1.4 `lineStream` 返回流
`lineStream`：像run一样立即返回，生成的输出通过Stream [String]提供

获取该流的下一个元素可能会阻塞，直到它变为可用。 如果返回码不为零，此方法将抛出异常


```
val contents: Stream[String] = Process("ls").lineStream
```

注：**如果不需要，请使用`lineStream_!`方法**。

```
val etcFiles = "find /etc" lines_! ProcessLogger(line => ())
```

## 4.2 处理输入和输出

### 4.2.1 ProcessBuilder以三种不同的方式组合；

- `#|` 将第一个命令的输出传递给第二个命令的输入。 它等价shell管道符 **|**
- `#&&` 有条件地执行第二个命令，如果前一个命令以退出值0结束。它等价shell的 **&&**
- `#||` 如果前一个命令的退出值不为零，则有条件地执行第三个命令。 它等价shell的 **||**



```
("ls" #| "grep test").!
```

### 4.2.2 重定向 `#>`、`#<`


```
a #< url or url #> a
a可以是一个文件或者一个命令，比如:

例子1:
new java.net.URL("http://www.baidu.com") #> new java.io.File("/tmp/baidu.html").!
示例中的url和file均需进行封装

例子2:
("hdfs dfs -ls -h " #> new java.io.File("/home/zhangchao/1.txt")).!

例子3:
"ls" #> new java.io.File("123") !
```


注意：<br>
1. **重定向必须用 new java.io.File("")封装，否则会当作命令，比如"ls" #> "/tmp/a" !将会出错，必须`"ls" #> new java.io.File("/tmp/a") !`**
2. 或者导入`import java.io.File`，使用`"hdfs dfs -df -h " #> new File("66") !`


重定向结合cat
```scala
scala> import java.io.File
import java.io.File

scala> val contents = ("cat" #< new File("/etc/passwd")).!!
contents: String =
"root:x:0:0:root:/root:/bin/bash
bin:x:1:1:bin:/bin:/sbin/nologin
daemon:x:2:2:daemon:/sbin:/sbin/nologin
adm:x:3:4:adm:/var/adm:/sbin/nologin
lp:x:4:7:lp:/var/spool/lpd:/sbin/nologin
sync:x:5:0:sync:/sbin:/bin/sync
shutdown:x:6:0:shutdown:/sbin:/sbin/shutdown
halt:x:7:0:halt:/sbin:/sbin/halt
mail:x:8:12:mail:/var/spool/mail:/sbin/nologin
operator:x:11:0:operator:/root:/sbin/nologin
games:x:12:100:games:/usr/games:/sbin/nologin
ftp:x:14:50:FTP User:/var/ftp:/sbin/nologin
nobody:x:99:99:Nobody:/:/sbin/nologin
avahi-autoipd:x:170:170:Avahi IPv4LL Stack:/var/lib/avahi-autoipd:/sbin/nologin
systemd-bus-proxy:x:999:997:systemd Bus Proxy:/:/sbin/nologin
systemd-network:x:998:996:systemd Network Management:/:/sbin/nologin
dbus:x:81:81:System message...

```

### 4.2.3 追加操作 `#>>`、 `#<<`

```
a #>> file or file #<<

例子：
"cat /etc/passwd " #>> new File("66.txt") !
```
