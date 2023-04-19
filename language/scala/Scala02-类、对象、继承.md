[TOC]
# 1. 类、对象、继承、特质
Scala的类与Java、C++的类比起来更简洁，学完之后你会更爱Scala！！！

## 1.1 类
### 1.1.1 类的定义
```scala
//在Scala中，类并不用声明为public。
//Scala源文件中可以包含多个类，所有这些类都具有公有可见性。

  class Student {

  //用val修饰的变量是只读属性，有getter但没有setter
  //（相当与Java中用final修饰的变量）

  val id = 666

  //用var修饰的变量既有getter又有setter

  var age: Int = 20
 
  //类私有字段,只能在类的内部使用

  private var name: String = "tom"

  //对象私有字段,访问权限更加严格的，Person类的方法只能访问到当前对象的字段

  private[this] val pet = "小强"
}
```

### 1.1.2 构造器

注意：<font color="red">主构造器会执行类定义中的所有语句</font>

```scala
/**

  *每个类都有主构造器，主构造器的参数直接放置类名后面，与类交织在一起

  */

  class Person(val name: String, val age: Int){

  //主构造器会执行类定义中的所有语句

  println("执行主构造器")

  

  try {

    println("读取文件")

    throw new IOException("io exception")

  } catch {

    case e: NullPointerException => println("打印异常Exception : " + e)

    case e: IOException => println("打印异常Exception : " + e)

  } finally {

    println("执行finally部分")

  }

 
  private var gender = "male"
  

  //用this关键字定义辅助构造器

  def this(name: String, age: Int, gender: String){

    //每个辅助构造器必须以主构造器或其他的辅助构造器的调用开始
    this(name, age)

    println("执行辅助构造器")

    this.gender = gender

  }

}
```


```scala
//在类名后面加private就变成了私有的

  class People private(val name: String, private var age: Int = 18){

  

  object People{

  def main(args: Array[String]) {

    //私有的构造器，只有在其伴生对象中使用

    val q = new People("hatano", 20)

  }

}
```

## 1.2 对象
### 1.2.1 单例对象

在Scala中没有静态方法和静态字段，但是<font color="red">可以使用object这个语法结构来达到同样的目的</font>

1.存放工具方法和常量

2.高效共享单个不可变的实例

3.单例模式

```scala
  import scala.collection.mutable.ArrayBuffer

  object SingletonDemo {

  def main(args: Array[String]) {

    //单例对象，不需要new，用【类名.方法】调用对象中的方法

    val session = SessionFactory.getSession()

    println(session)

  }

}


  object SessionFactory{

  //该部分相当于java中的静态块

  var counts = 5

  val sessions = new ArrayBuffer[Session]()

  while(counts > 0){

    sessions += new Session

    counts -= 1

  }


  //在object中的方法相当于java中的静态方法

  def getSession(): Session ={

    sessions.remove(0)

  }

}

  class Session{

}
```

### 1.2.2 伴生对象

在Scala的类中，与类名相同的对象叫做<font color="red">伴生对象</font>，类和伴生对象之间可以相互访问私有的方法和属性


```scala
 class Dog {

  val id = 1

  private var name = "xiaoqing"

  

  def printName(): Unit ={

    //在Dog类中可以访问伴生对象Dog的私有属性

    println(Dog.CONSTANT + name )

  }

}


  /**

  * 伴生对象

  */

  object Dog {

  

  //伴生对象中的私有属性

  private val CONSTANT = "汪汪汪 : "

  

  def main(args: Array[String]) {

    val p = new Dog

    //访问私有的字段name

    p.name = "123"

    p.printName()

  }

}
```

### 1.2.3 apply方法

通常我们会在类的伴生对象中定义apply方法，当遇到<font color="red">类名(参数1,...参数n)</font>时apply方法会被调用


```scala
object ApplyDemo {

  def main(args: Array[String]) {

    //调用了Array伴生对象的apply方法

    //def apply(x: Int, xs: Int*): Array[Int]

    //arr1中只有一个元素5

    val arr1 = Array(5)

    println(arr1.toBuffer)

  

    //new了一个长度为5的array，数组里面包含5个null

    var arr2 = new Array(5)

  }

}
```

Scala程序特别要指出的是，**<font color="red">单例对象</font>会在第一次被访问的时候初始化**。

Scala 的apply 有2 张形式，一种是 伴生对象的apply ，一种是 伴生类中的apply，下面展示这2中的apply的使用。

```scala
class ApplyOperation {
}
class ApplyTest{
    def apply() = println("I am into spark so much!!!")
    def haveATry: Unit ={
        println("have a try on apply")
    }
}
object ApplyTest{
     def apply() = {
          println("I  am into Scala so much")
        new ApplyTest
     }
}
object ApplyOperation{
     def main (args: Array[String]) {
        val array= Array(1,2,3,4)
        val a = ApplyTest() //这里就是object中的apply使用

         a.haveATry
         a() // 这里就是 class 中 apply使用
    }
}
```

运行结果

```scala
I am into Scala so much 
have a try on apply 
I am into spark so much!!!
```

1.2.4 应用程序对象

Scala程序都必须从一个对象的main方法开始，可以通过扩展App特质，不写main方法。

```scala
object AppObjectDemo extends App{

  //不用写main方法

  println("I love you Scala")

}
```

## 1.3 继承
### 1.3.1 扩展类

在Scala中扩展类的方式和Java一样都是使用extends关键字

### 1.3.2 重写方法

在Scala中重写一个非抽象的方法必须使用override修饰符

### 1.3.3 类型检查和转换
|Scala|Java|
|---|---|
|obj.isInstanceOf[C]|obj instanceof C|
|obj.asInstanceOf[C]|(C)obj|
|classOf[C]|C.class|

### 1.3.4 超类的构造

```scala
object ClazzDemo {

  def main(args: Array[String]) {

  }

}

  

  trait Flyable{

  def fly(): Unit ={

    println("I can fly")

  }

  

  def fight(): String

  }

  

  abstract class Animal {

  def run(): Int

  val name: String

  }

  

  class Human extends Animal with Flyable{

  

  val name = "abc"

  

  //打印几次"ABC"?

  val t1,t2,(a, b, c) = {

    println("ABC")

    (1,2,3)

  }

  

  println(a)

  println(t1._1)

  

  //在Scala中重写一个非抽象方法必须用override修饰

  override def fight(): String = {

    "fight"

  }

  //在子类中重写超类的抽象方法时，不需要使用override关键字，写了也可以

  def run(): Int = {
    1
  }

}
```

# 2. Any、Nothing、Null、Nil
参考：

https://blog.csdn.net/u013007900/article/details/79178749



[![](https://pic.imgdb.cn/item/6128b66c44eaada73949171a.png)](https://z3.ax1x.com/2021/08/27/hQwSiD.png)

## 2.1 Any

在scala中，Any类是所有类的超类。Any有两个子类：AnyVal和AnyRef。

对于直接类型的scala封装类，如Int、Double等，AnyVal是它们的基类；对于引用类型，AnyRef是它们的基类。

Any是一个抽象类，它有如下方法：!=()、==()、asInstanceOf()、equals()、hashCode()、isInstanceOf()和toString()。

AnyVal没有更多的方法了。AnyRef则包含了Java的Object类的一些方法，比如notify()、wait()和finalize()。

AnyRef是可以直接当做java的Object来用的。对于Any和AnyVal，只有在编译的时候，scala编译器才会将它们视为Object。换句话说，在编译阶段Any和AnyVal会被类型擦除为Object。

## 2.2 Nothing

如上图所示，Nothing是所有类的子类，是一个类。Nothing没有对象，但是可以用来定义类型。

在Java当中你很难找到类似的东西对应Scala中的Nothing。Nothing到底是什么呢？或者换个方向考虑：Nothing的用处是什么呢？

1.  用于标记不正常的停止或者中断
2.  一个空的collection

```Scala
scala> def foo = throw new RuntimeException
foo: Nothing
scala> val l:List[Int] = List[Nothing]()
l: List[Int] = List()
```

因为`List[+A]`定义是协变的，所以`List[Nothing]`是`List[Int]`的子类，但`List[Null]`不是`List[Int]`的子类

## 2.3 Null

Null是所有AnyRef的子类，是所有继承了Object的类的子类，所以Null可以赋值给所有的引用类型(AnyRef)，不能赋值给值类型(AnyVal)，这个java的语义是相同的。null是Null的唯一对象（实例）。

```Scala
val x = null         
val y: String = null 
val z: Int = null    
val u: Unit = null   
```

注意，不要被Unit值类型在赋值时的障眼法给骗了，以为null可以被赋给Unit。实际上把任何表达式结果不是Unit类型的表达式赋给Unit类型变量，都被转为`{ expr; () }`，所以上面的等同于`{null; ()}`把最终得到的`()`赋给了`u`变量。

Null在类型推导时只能被造型为AnyRef分支下的类型，不能造型为AnyVal分支下的类型，不过我们显式的通过asInstanceOf操作却又是可以把null造型为AnyVal类型的:

```Scala
val i = null.asInstanceOf[Int]
```

先装箱`Int`为引用类型，`null`被造型成了引用类型的`Int(java.lang.Integer)`，然后又做了一次拆箱，把一个为`null`的`Integer`类型变量造型成`Int`值类型，但在拆箱这一点处理上，体现了scala与java的不同：

```Scala

int i = (int)((Integer)null);

val i = null.asInstanceOf[java.lang.Integer].asInstanceOf[Int]
```

在java里基本类型(primitive type) 与引用类型是有明确差异的，虽然提供了自动装箱拆箱的便捷，但在类型上两者是不统一的；而scala里修正这一点，`Int`类型不再区分`int/Integer`，类型一致，所以值为`null`的`Integer`在通过`asInstanceOf[Int]`时被当作一个未初始化的`Int`对待，返回了一个默认值`Int`(注:`asInstanceOf`不改变原值，返回一个新值)。

## 2.4 Nil

Nil是一个空的List，定义为`List[Nothing]`，根据List的定义`List[+A]`，所有Nil是所有`List[T]`的子类。

## 2.5 None

Scala中的Option与Java 8引入的java.util.Optional语义相同，表示“**一个实例有可能不为空，也有可能为空**”，旨在通过让用户提前判空来避免讨厌的NullPointerException，降低直接返回null的风险。

Some和None都是Option的子类，None的本质就是Option[Nothing]


```Scala
scala> def showPositive(num: Int) = {
     |   getPositive(num) match {
     |     case Some(x) => println("Got a positive number")
     |     case None => println("Got a non-positive number")
     |   }
     | }
showPositive: (num: Int)Unit
```

Scala可以优雅地使用模式匹配来处理Some和None的情况，所以Option在主要以Scala写成的开源框架（如Spark）中应用甚为广泛

## 2.6 Unit
Unit完全等同于Java里的void，表示函数或方法没有返回值

Unit是一种特殊的值类型（AnyVal）的子类，并且它不代表任何实际的值类型，故其getClass()方法返回null。

# 3. scala中常用的几种比较方法的用法

## 3.1 “`==`”方法的使用及含义

首先看一下官方文档给的解释：

```Scala
final def ==(arg0: Any): Boolean
   
    Test two objects for equality. The expression x == that is equivalent to if (x eq null) that eq null else x.equals(that).
    returns true if the receiver object is equivalent to the argument; false otherwise.

```

可以看到 **\==** 含义中切分为两部分：eq与equals，具体要判断x是否为null,要更进一步理解 **\==** 需要接着往下看 **equals与eq** 的含义

## 3.2 equals方法的使用及含义

官方文档解释：

```Scala
***def equals(arg0: Any): Boolean***
  Compares the receiver object (this) with the argument object (that) for equivalence.
Any implementation of this method should be an equivalence relation:
  It is reflexive: for any instance x of type Any, x.equals(x) should return true.
  It is symmetric: for any instances x and y of type Any, x.equals(y) should return true if and only if y.equals(x) returns true.
  It is transitive: for any instances x, y, and z of type Any if x.equals(y) returns true and y.equals(z) returns true, then x.equals(z) should return true.
  If you override this method, you should verify that your implementation remains an equivalence relation. Additionally, when overriding this method it is usually necessary to override hashCode to ensure that objects which are "equal" (o1.equals(o2) returns true) hash to the same scala.Int. (o1.hashCode.equals(o2.hashCode)).
 returns true if the receiver object is equivalent to the argument; false otherwise.
 ```

上面一大堆英文总的来说有几方面意思：

1.  equals方法具有几个性质：自反性，对称性，传递性
2.  如果你需要重写equals方法，那么也必须要重写对象对应的hashCode方法，且如果对象1equals对象2的话，那么这两个对象的hashCode也应该相等
3.  equals方法比较的是值是否相等

## 3.3 eq方法的使用及含义


官方文档解释：

```Scala
***final def eq(arg0: AnyRef): Boolean***
  Tests whether the argument (that) is a reference to the receiver object (this).

  The eq method implements an equivalence relation on non-null instances of AnyRef, and has three additional properties:

  It is consistent: for any non-null instances x and y of type AnyRef, multiple invocations of x.eq(y) consistently returns true or consistently returns false.
For any non-null instance x of type AnyRef, x.eq(null) and null.eq(x) returns false.
null.eq(null) returns true.
When overriding the equals or hashCode methods, it is important to ensure that their behavior is consistent with reference equality. Therefore, if two objects are references to each other (o1 eq o2), they should be equal to each other (o1 == o2) and they should hash to the same value (o1.hashCode == o2.hashCode).
returns
true if the argument is a reference to the receiver object; false otherwise.
```

简而言之，这里eq方法进行的是引用比较，比较两个对象的引用是否相同

## 3.4 ne方法的使用及其含义

官方解释：

```Scala
***final def ne(arg0: AnyRef): Boolean***
  Equivalent to !(this eq that).
  returns true if the argument is not a reference to the receiver object; false otherwise.
  ```

可以看出来，ne方法是eq方法的反义

## 3.5 总结

综合上面几种方法的阐述，可以看出来

1.  如果要比较对象的引用是否相同或者不同，请用eq或ne方法
2.  如果要比较值是否相等，请用equals方法或者`==`方法，这里推荐使用`==`方法，因为如果比较值为null的情况下，调用equals方法是会报错的，而`==`方法则避免了这个问题，它事先为我们检查了是否为null，然后在进行相应比较