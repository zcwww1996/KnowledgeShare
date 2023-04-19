[TOC]

来源： https://my.oschina.net/leejun2005/blog/405305

Scala作为一门函数式编程语言，对习惯了指令式编程语言的同学来说，会不大习惯，这里除了思维方式之外，还有语法层面的，比如 underscore（下划线）就会出现在多种场合，令初学者相当疑惑，今天就来总结下 Scala 中下划线的用法。

## 1、存在性类型：Existential types

```scala
def foo(l: List[Option[_]]) = ...
```


## 2、高阶类型参数：Higher kinded type parameters

```scala
case class A[K[_],T](a: K[T])
```


## 3、临时变量：Ignored variables

```scala
val _ = 5
```


## 4、临时参数：Ignored parameters

```scala
List(1, 2, 3) foreach { _ => println("Hi") }
```


## 5、通配模式：Wildcard patterns

```scala
Some(5) match { case Some(_) => println("Yes") }
match {
     case List(1,_,_) => " a list with three element and the first element is 1"
     case List(_*)  => " a list with zero or more elements "
     case Map[_,_] => " matches a map with any key type and any value type "
     case _ =>
 }
val (a, _) = (1, 2)
for (_ <- 1 to 10)
```


## 6、通配导入：Wildcard imports

```scala
import java.util._
```


## 7、隐藏导入：Hiding imports

```scala
// Imports all the members of the object Fun but renames Foo to Bar
import com.test.Fun.{ Foo => Bar , _ }


// Imports all the members except Foo. To exclude a member rename it to _
import com.test.Fun.{ Foo => _ , _ }
```

## 8、连接字母和标点符号：Joining letters to punctuation

```scala
def bang_!(x: Int) = 5
```


## 9、占位符语法：Placeholder syntax

```scala
List(1, 2, 3) map (_ + 2)
_ + _   
( (_: Int) + (_: Int) )(2,3)


val nums = List(1,2,3,4,5,6,7,8,9,10)

nums map (_ + 2)
nums sortWith(_>_)
nums filter (_ % 2 == 0)
nums reduceLeft(_+_)
nums reduce (_ + _)
nums reduceLeft(_ max _)
nums.exists(_ > 5)
nums.takeWhile(_ < 8)
```

1) **占位符用来表示的参数必须满足只在函数字面量中出现一次**
2) **在使用占位符的时候，编译器可能并没有足够的信息区推断你缺失的类型**

```scala
scala> val f = _ + _
<console>:11: error: missing parameter type for expanded function ((x$1: <error>, x$2) => x$1.$plus(x$2))
       val f = _ + _
               ^
<console>:11: error: missing parameter type for expanded function ((x$1: <error>, x$2: <error>) => x$1.$plus(x$2))
       val f = _ + _
                   ^
```
上面的例子就是，我们在定义 f 方法的时候只用了占位符来表示两个参数，但是编译器并不能推断出你的参数类型，报错。

这个时候我们需要注明参数类型

```scala
scala> val f = (_:Int) + (_:Int)
f: (Int, Int) => Int = $$Lambda$1108/2058316797@4a8bf1dc
 
scala> f(1,2)
res5: Int = 3
```
3) **当使用多个占位符的时候，代表的是不同的参数，不能是相同的参数。**

4) **占位符也可以代替一个参数列表**

```Scala
scala> def sum(a:Int,b:Int,c:Int) = a+b+c
sum: (a: Int, b: Int, c: Int)Int
 
scala> val a = sum _
a: (Int, Int, Int) => Int = $$Lambda$1137/810864083@755a4ef5
```

## 10、偏应用函数：Partially applied functions

```scala
def fun = {
    // Some code
}
val funLike = fun _

List(1, 2, 3) foreach println _

1 to 5 map (10 * _)

//List("foo", "bar", "baz").map(_.toUpperCase())
List("foo", "bar", "baz").map(n => n.toUpperCase())
```


## 11、初始化默认值：default value

```scala
var i: Int = _
```


## 12、作为参数名：

```scala
//访问map
var m3 = Map((1,100), (2,200))
for(e<-m3) println(e._1 + ": " + e._2)
m3 filter (e=>e._1>1)
m3 filterKeys (_>1)
m3.map(e=>(e._1*10, e._2))
m3 map (e=>e._2)

//访问元组：tuple getters
(1,2)._2
```


## 13、参数序列：parameters Sequence 
`_*`作为一个整体，告诉编译器你希望将某个参数当作参数序列处理。例如val s = sum(1 to 5:`_*`)就是将1 to 5当作参数序列处理。

```scala
//Range转换为List
List(1 to 5:_*)

//Range转换为Vector
Vector(1 to 5: _*)

//可变参数中
def capitalizeAll(args: String*) = {
  args.map { arg =>
    arg.capitalize
  }
}

val arr = Array("what's", "up", "doc?")
capitalizeAll(arr: _*)
```

这里需要注意的是，以下两种写法实现的是完全不一样的功能：

```scala
foo _               // Eta expansion of method into method value

foo(_)              // Partial function application
```

Example showing why foo(_) and foo _ are different:

```scala
trait PlaceholderExample {
  def process[A](f: A => Unit)

  val set: Set[_ => Unit]

  set.foreach(process _) // Error 
  set.foreach(process(_)) // No Error
}
```

In the first case, process _ represents a method; Scala takes the polymorphic method and attempts to make it monomorphic by filling in the type parameter, but realizes that there is no type that can be filled in for A that will give the type (_ => Unit) => ? (Existential _ is not a type).

In the second case, process(_) is a lambda; when writing a lambda with no explicit argument type, Scala infers the type from the argument that foreach expects, and _ => Unit is a type (whereas just plain _ isn't), so it can be substituted and inferred.

This may well be the trickiest gotcha in Scala I have ever encountered.

<font style="background-color:yellow;">Refer：</font>

[1] What are all the uses of an underscore in Scala?

http://stackoverflow.com/questions/8000903/what-are-all-the-uses-of-an-underscore-in-scala

[2] Scala punctuation (AKA symbols and operators)

http://stackoverflow.com/questions/7888944/scala-punctuation-aka-symbols-and-operators/7890032#7890032

[3] Scala中的下划线到底有多少种应用场景？

http://www.zhihu.com/question/21622725

[4] Strange type mismatch when using member access instead of extractor

http://stackoverflow.com/questions/9610736/strange-type-mismatch-when-using-member-access-instead-of-extractor/9610961

[5] Scala简明教程

http://colobu.com/2015/01/14/Scala-Quick-Start-for-Java-Programmers/

[6] Scala入门到精通——第二十三节 高级类型 （二）

https://yq.aliyun.com/articles/60371#
