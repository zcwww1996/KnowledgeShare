[TOC]
# 1. for
Scala 语言中for循环的语法：

```scala
for( var x <- Range ){
  statement(s);
}
```

以上语法中，**Range**可以是一个数字区间表**i to j**，或者**i until j**。左箭头 `<-` 用于为变量 x 赋值。

- to：包括上界和下界
- until：包括上界，不包括下界

## 1.1 实例
以下是一个使用了 i to j 语法(包含 j)的实例:

```scala
object Test {
   def main(args: Array[String]) {
    // Range:to:默认步进为1
    val to1 = 1 to 10
    println(to1)
    // 定义一个步进为3的Range
    val to2 = 1 to 10 by 3
    println(to2)
    println(to2.toList)
   }
}
```

执行以上代码输出结果为：

```
$ scalac Test.scala
$ scala Test
Range(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
Range(1, 4, 7, 10)
List(1, 4, 7, 10)
```

以下是一个使用了 i until j 语法(<font color="red">不包含j</font>)的实例:

```scala
object Test {
   def main(args: Array[String]) {
      var a = 0;
      // for 循环
      for( a <- 1 until 10){
         println( "Value of a: " + a );
      }
   }
}
```

执行以上代码输出结果为：

```scala
$ scalac Test.scala
$ scala Test
value of a: 1
value of a: 2
value of a: 3
value of a: 4
value of a: 5
value of a: 6
value of a: 7
value of a: 8
value of a: 9
```

## 1.2 for 循环的特殊之处
### 1.2.1 卫语句

意思是for中嵌入if 语句，比如输出1到10偶数

```scala
for (i <- 1 to 10 if i % 2 == 0) {
println(i)
}
```

### 1.2.2 for/yield

for 和 yield结合感觉和Python的迭代器类似，会产生一个可迭代的对象。

for 循环中的 yield 会把当前的元素记下来，保存在集合中，循环结束后将返回与给定集合相同类型的集合。如果被循环的是 Map，返回的就是 Map，被循环的是 List，返回的就是 List，以此类推。

```scala
val a: Array[String] = Array("wu", "zheng", "Hello")
val lengths: Array[Int] = for(e<-a) yield {
  e.length()
}
lengths.foreach(println(_))
```
lengths还是array类型，结果

```scala
2
5
5
```

# 2. switch 和模式匹配
说老实话swith我一般用得较少，但是Scala的switch 我觉得不能不说，结合Scala 的模式匹配，感觉就不得不提了。

```scala
val i = 1
val x = i match {
case 1 => "one"
case 2 => "two"
case _ => "many"
}
```

这感觉是不是和switch 和相似？都推荐使用@switch ，在不能编译成`tableswitch`或`lookswitch`的时候会有警告。

```scala
val i = 1
val x = (i: @switch) match {
case 1 => "one"
case 2 => "two"
case _ => "many"
}
```
## 2.1 case语句可匹配类型
case语句中可以使用，常量模式，变量模式，构造函数模式，序列模式，元组模式，或者类型模式。

```scala
def echoCase(x: Any): String = x match {
case 0 => "zero"
case true => "true"
case "hello " => "hello wuzheng"
case Nil => "空数组"
case List(0, _, _) => "三个元素的序列，其中第一个为0"
case List(1, _*) => "第一个元素为1序列"
case s: String => s"输入的值是 String : $s"
case i: Int => s"输入的是整数： $i"
case _ => "我不知道了"
}
```

## 2.2 try/catch也是使用case捕获异常

```scala
val s = "wu zheng"
try {
val i = s.toInt
}catch {
case e: Exception => e.printStackTrace()
}
```

# 3. break和continue

## 3.1 break和continue的区别

1. break用于跳出一个循环体或者完全结束一个循环，不仅可以结束其所在的循环，还可结束其外层循环。<br>
**注意：**
   1) 只能在循环体内和switch语句体内使用break。
   2) 不管是哪种循环，一旦在循环体中遇到break，系统将完全结束循环，开始执行循环之后的代码。
   3) 当break出现在循环体中的switch语句体内时，起作用只是跳出该switch语句体，并不能终止循环体的执行。若想强行终止循环体的执行，可以在循环体中，但并不在switch语句中设置break语句，满足某种条件则跳出本层循环体。
2. continue语句的作用是跳过本次循环体中剩下尚未执行的语句，立即进行下一次的循环条件判定，可以理解为只是中止(跳过)本次循环，接着开始下一次循环。<br>
**注意：**
   1) continue语句并没有使整个循环终止。
   2) continue 只能在循环语句中使用，即只能在 for、while 和 do…while 语句中使用。

## 3.2 使用案例

在scala中，类似Java和C++的break/continue关键字被移除了

如果一定要使用break/continue，就需要使用`scala.util.control`包的Break类的breable和break方法。


### 3.2.1 实现break
- 导入Breaks包`import scala.util.control.Breaks._`
- 使用breakable{}将整个for表达式包起来
- for表达式中需要退出循环的地方，添加break()方法调用


```scala
//使用for表达式打印1-100的数字，如果数字到达50，退出for表达式

// 导入scala.util.control包下的Break
import scala.util.control.Breaks._

breakable{
    for(i <- 1 to 100) {
        if(i >= 50) break()
        else println(i)
    }
}

```

### 3.2.2 实现continue
- 导入Breaks包`import scala.util.control.Breaks._`
- 在for{}循环内部，用breakable{}将for表达式的内部循环体包含起来
- for表达式中需要退出循环的地方，添加break()方法调用


```scala
//打印1-100的数字，使用for表达式来遍历，如果数字能整除10，不打印
// 导入scala.util.control包下的Break    
import scala.util.control.Breaks._

for(i <- 1 to 100 ) {
    breakable{
        if(i % 10 == 0) break()
        else println(i)
    }
}

```


**两个例子的区别：==在循环内，就是continue，在循环外就是break==**