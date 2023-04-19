[TOC]
# 1. Scala 继承与多态
## 1.1 继承
Scala继承一个基类跟Java很相似, 但我们需要注意以下几点：

1. 重写父类（基类、超类）的非抽象方法必须使用override修饰符。
2. 只有主构造函数才可以往基类的构造函数里写参数。
3. 在子类（派生类）中重写超类的抽象方法时，不需要使用override关键字。
4. 子类实现的第一个特质，也用extends

接下来让我们来看个实例：

```scala
trait Beatable {

 def beat(): Unit = {
    println("打架")
  }
}

trait FlyAble {

  def fly()

}


abstract class Animal {

  //抽象方法，没有实现
  def run(): Unit = {
    println("run.....")
  }

  //非抽象的方法，有实现了
  def eat(): Unit = {
    println("吃食物")
  }
}

class Monkey extends Animal with FlyAble with Beatable {
  //抽象方法，没有实现（重写父类抽象的方法，可以选择性的加override）
  override def run(): Unit = {
    println("边走边跳")
  }

  //非抽象的方法，有实现了（重写父类非抽象的方法，必须加override）
  override def eat(): Unit = {
    println("吃桃子")
  }

  override def fly(): Unit = {
    println("乘筋斗云飞")
  }

  override def beat(): Unit = {
    println("用金箍棒")
  }
}

object Monkey {
  def main(args: Array[String]): Unit = {

    val m = new Monkey
    m.run()
    m.eat()
    m.fly()
    m.beat()
  }

}
```

## 1.2 多态
多态是为了让代码更灵活，降低耦合度

多态必须满足以下三点：

1. 有继承或实现接口
2. 父类引用指向子类对象或接口指向实现类
3. 方法要重写


```scala
class Pig extends FlyAble with Beatable{

  override def fly(): Unit = {
    println("会飞的猪")
  }

  override def beat(): Unit = {
    println("用耙子打")
  }
}


object DuoTaiDemo {

    def main(args: Array[String]): Unit = {
    val a: Animal = new Monkey
    a.run()
        
    val f: FlyAble = new Pig
    f.fly()
  }
}
```

# 2. 可变集合
数组的特点：**长度不能变，内容可变**

集合分为两类：
1. 一类是**不可变集合**，即**长度和内容都不可改变**，适用的场景是<font color="red">**多线程访问**</font>
2. 另一类是**可变集合**，即**长度和内容都可改变**

对于可变集合:
   1) 追加一个元素  <font color="red">+=</font>
   2) 移除一个元素  <font color="red">-=</font>
   3) 追加一个类型一样的集合用 <font style="background-color: yellow;">++=</font>

## 2.1 导入所有的可变集合包

`_` scala中的通配符，表示“所有的”，等同于Java下的 `*`

**import scala.collection.mutable.<font color="red">_</font>**


```scala
导包
import scala.collection.mutable.ListBuffer
    var br = new ListLBuffer[Int]()
    br.append(1)
    br + =2
    br.+ =3
    br + =(4,5,6)
   var list1 = List(7,8,9)
    br ++ = list1
    
    br.remove(0)
    //去掉角标为零的
    br - =2
    //把第一个符合的元素（2）删除
    br - =(1,3,4)
    //去掉不连续的1，3，4
    var list2=(3,7,9)
    br --= list2
    //遍历list2，在br中去掉第一次出现的list中的数字（可以不连续）
```

## 2.2 map
### 2.2.1 map中添加与移除元素
必须是可变map，才能添加、更新元素，`mutable.Map`包
```scala
//val mp = Map[String, Int]("a" -> 1, "b" -> 2)

    val hm = new mutable.HashMap[String, Int]()

    // 添加或更新元素
    hm += (("a", 1))
    hm += ("b" -> 2,"bb" -> 22)
    hm("c") = 3
    hm.put("d", 4)

    println(hm)
    //重map中移除元素

    hm -= "a"

    hm.remove("b")
```

### 2.2.1 immutable.Map 和 mutable.Map 相互转化
immutable.Map -> mutable.Map

```scala
// 方法1
val m = collection.immutable.Map(1->"one",2->"Two")
val n = collection.mutable.Map(m.toSeq: _*)
```
在scala 2.13中，有两种替代方法:源Map实例的to方法，或者目标Map伴生对象的from方法。

```scala
import scala.collection.mutable

val immutable = Map(1 -> 'a', 2 -> 'b')
// 方法2
val mutableMap1: mutable.Map[Int, Char] = immutable.to(mutable.Map)
// 方法3
val mutableMap2: mutable.Map[Int, Char] = mutable.Map.from(immutable)
```


mutable.Map -> immutable.Map

```scala
val m = collection.mutable.Map(1->"one",2->"Two")
val n = m.toMap
```


## 2.3 list
scala的集合中有两种List:

```scala
scala.collection.immutable.List  //长度内容都不可变
scala.collection.mutable.ListBuffer //长度内容都可变，必须导入包
```

注意：immutable下没有ListBuffer, mutable下没有List;
### 2.3.2 list 创建

**1) 最常见的创建list的方式之一。**

```
方法1：
scala> val list = 1 :: 2 :: 3 :: Nil
list: List[Int] = List(1, 2, 3)

方法2：
scala> val list = List(1,2,3)
list: List[Int] = List(1, 2, 3)
```

**2) 集合混合类型组成。**

```
scala> val list = List(1,2.0,33D,4000L)
list: List[Double] = List(1.0, 2.0, 33.0, 4000.0)
```

**3) 集合混合类型组成，可以有自己控制。**

下面的例子的集合保持了原有集合的类型。

```
scala> val list = List[Number](1,2.0,33D,4000L)
list: List[Number] = List(1, 2.0, 33.0, 4000)
```

**4) range创建和填充集合。**

```
scala> val list = List.range(1,10)
list: List[Int] = List(1, 2, 3, 4, 5, 6, 7, 8, 9)
```

**5) fill创建和填充集合。**

集合长度为3，重复元素为“hello”
```
scala> val list = List.fill(3)("hello")
list: List[String] = List(hello, hello, hello)
```

**6) tabulate创建和填充集合。**

方法的第一个参数为元素的数量，可以是二维的，第二个参数为指定的函数，我们通过指定的函数计算结果并返回值插入到列表中，起始值为 0
```
 // 通过给定的函数创建 5 个元素
    val squares = List.tabulate(6)(n => n * n)
    println( "一维 : " + squares  )
 
 // 创建二维列表
    val mul = List.tabulate( 4,5 )( _ * _ )      
    println( "多维 : " + mul  )
```

执行以上代码，输出结果为：


```
一维 : List(0, 1, 4, 9, 16, 25)
多维 : List(List(0, 0, 0, 0, 0), List(0, 1, 2, 3, 4), List(0, 2, 4, 6, 8), List(0, 3, 6, 9, 12))
```


**7) 将字符串转化为List的形式。**

```
scala> "hello".toList
res41: List[Char] = List(h, e, l, l, o)
```

**8) 创建可变的list**

方法是使用ListBuffer

```
scala> val list = collection.mutable.ListBuffer(1,2,3)
scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3)
```


```
scala> import scala.collection.mutable.ListBuffer
import scala.collection.mutable.ListBuffer

scala> var fruits = new ListBuffer[String]()
fruits: scala.collection.mutable.ListBuffer[String] = ListBuffer()

scala> fruits += "apple"
res42: scala.collection.mutable.ListBuffer[String] = ListBuffer(apple)

scala> fruits += "orange"
res43: scala.collection.mutable.ListBuffer[String] = ListBuffer(apple, orange)

scala> fruits += ("banana","grape","pear")
res44: scala.collection.mutable.ListBuffer[String] = ListBuffer(apple, orange, banana, grape, pear)

scala> val fruitsList = fruits.toList
fruitsList: List[String] = List(apple, orange, banana, grape, pear)
```


### 2.3.2 list 转换处理

#### 2.3.2.1 添加元素

1) 使用`::`方法在列表**前**添加元素

```Scala
scala> var list = List(2)
list: List[Int] = List(2)

scala> list = 1 :: list
list: List[Int] = List(1, 2)
```

2) 使用`+:`方法，复制添加元素**前**列表

```Scala
scala> val x = List(1)
x: List[Int] = List(1)

scala> val y = 2 +: x
y: List[Int] = List(2, 1)

scala> println(x)
List(1)
```

3) 使用`:+`方法，复制添加元素**后**列表

```Scala
scala> val a = List(1)
a: List[Int] = List(1)

scala> val b = a :+ 2
b: List[Int] = List(1, 2)

scala> println(a)
List(1)
```

#### 2.3.2.2 删除元素
1) List是不可变的，不能从中删除元素，但是可以过滤掉不想要的元素，然后将结果赋给一个新的变量。

```Scala
scala> val list = List(4,5,2,1,3)
list: List[Int] = List(4, 5, 2, 1, 3)

scala> val newList = list.filter(_ > 2)
newList: List[Int] = List(4, 5, 3)
```

2) 像这样反复的操作结果赋给变量的方式是可以避免的，通过声明变量var，然后将每次操作的结果返回给该变量。

```Scala
scala> var list = List(5,2,3,4,1)
list: List[Int] = List(5, 2, 3, 4, 1)

scala> list = list.filter(_ > 2)
list: List[Int] = List(5, 3, 4)
```

3) 如果列表经常变动，使用ListBuffer是一个不错的选择。ListBuffer是可变的，因此可以使用可变序列的所有方法从中删除元素。

```Scala
import scala.collection.mutable.ListBuffer

scala> val x = ListBuffer(1,2,3,4,5,6,7,8,9)
x: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3, 4, 5, 6, 7, 8, 9)

//可以按值每次删除一个元素：
scala> x -= 5
res45: x.type = ListBuffer(1, 2, 3, 4, 6, 7, 8, 9)

//可以一次删除两个或者更多的元素:
scala> x -= (2,3,4)
res46: x.type = ListBuffer(1, 6, 7, 8, 9)

//可以按位置删除元素:
scala> x.remove(0)
res47: Int = 1
scala> x
res48: scala.collection.mutable.ListBuffer[Int] = ListBuffer(6, 7, 8, 9)

//remove可以从定始位置删除指定数量的元素：
scala> x.remove(1,3)
scala> x
res50: scala.collection.mutable.ListBuffer[Int] = ListBuffer(6)

//可以用--=的方法从指定的集合中删除元素。
scala> val x = ListBuffer(1,2,3,4,5,6,7,8,9)
x: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3, 4, 5, 6, 7, 8,9)
scala> x --= Seq(1,2,3)
res51: x.type = ListBuffer(4, 5, 6, 7, 8, 9)
```

==如果ListBuffer中有相同多个元素 只能删除一个==

**批量删除相同元素**

```scala
val buffer = ListBuffer(1, 4, 6, 7, 8, 9,1)
for (x <- buffer){
    if (x==1) buffer-=1
}
print(buffer)


buffer: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 4, 6, 7, 8, 9, 1)
ListBuffer(4, 6, 7, 8, 9)
```

#### 2.3.2.3 列表的合并或者连接


1) 使用`++`方法合并两个列表

```
scala> val a = List(1,2,3)
a: List[Int] = List(1, 2, 3)

scala> val b = List(4,5,6)
b: List[Int] = List(4, 5, 6)

scala> val c = a ++ b
c: List[Int] = List(1, 2, 3, 4, 5, 6)
```

2) 使用`:::`、`concat`合并两个列表


你可以使用 `:::` 运算符或`List.:::()` 方法或 `List.concat()` 方法来连接两个或多个列表

==**`:::` 运算符是将后面的集合追加到后面，`List.:::()` 方法是将后面的集合追加到前面**==

```Scala
object Test {
   def main(args: Array[String]) {
      val site1 = "Runoob" :: ("Google" :: ("Baidu" :: Nil))
      val site2 = "Facebook" :: ("Taobao" :: Nil)

      // 使用 ::: 运算符
      var fruit = site1 ::: site2
      println( "site1 ::: site2 : " + fruit )
     
      // 使用 List.:::() 方法
      fruit = site1.:::(site2)
      println( "site1.:::(site2) : " + fruit )

      // 使用 concat 方法
      fruit = List.concat(site1, site2)
      println( "List.concat(site1, site2) : " + fruit  )
     

   }
}
```

输出结果：

```
site1 ::: site2 : List(Runoob, Google, Baidu, Facebook, Taobao)
site1.:::(site2) : List(Facebook, Taobao, Runoob, Google, Baidu)
List.concat(site1, site2) : List(Runoob, Google, Baidu, Facebook, Taobao)
```

3) <font style="color: red;">**ListBuffer添加数据操作**</font>

```
def main(args: Array[String]): Unit = {
    //ListBuffer
    val listBuffer = ListBuffer(1,2,3)
    val listBuffer1 = ListBuffer(88,99)
 
    //添加元素，可变集合，list本身发生变化，而不是返回新的list
    listBuffer += 4
    listBuffer.append(5,6)
 
    //添加整个集合(扁平)
    listBuffer ++= listBuffer1
    //++=的展开写法，需要接收返回值，也是ListBuffer
    val listBuffer2 = listBuffer ++ listBuffer1
    println(listBuffer)
    println(listBuffer2)
 
    //ListBuffer也支持不可变List的操作。同样返回值也是ListBuffer
    val listBuffer3 = listBuffer :+ 100
     
    println(listBuffer3)
 
  }
```

#### 2.3.2.4 List转ListBuffer

用`to[ListBuffer]`方法

```scala
scala> val l = List(1,2,3)
l: List[Int] = List(1, 2, 3)
scala> l.to[ListBuffer]
res1: scala.collection.mutable.ListBuffer[Int] = ListBuffer(1, 2, 3)
```


### 2.3.3 list 交并非
```scala
object ListTest {
  def main(args: Array[String]) {
    //创建一个List
    val lst0 = List(1,7,9,8,0,3,5,4,6,2)
    //将lst0中每个元素乘以10后生成一个新的集合
    //val lst1 = lst0.map(x => x * 2)
    val lst1 = lst0.map(_ * 2)
    //将lst0中的偶数取出来生成一个新的集合
    //val lst2 = lst0.filter(x => x % 2 == 0)
    val lst2 = lst0.filter(_ % 2 == 0)
    //将lst0排序后生成一个新的集合
    val lst3 = lst0.sorted
    val lst4 = lst0.sortBy(x => x)
    //val lst5 = lst0.sortWith((x, y) => x < y)
    val lst5 = lst0.sortWith(_ < _)
    //反转顺序
    val lst6 = lst3.reverse
    //将lst0中的元素4个一组,类型为Iterator[List[Int]]
    val it = lst0.grouped(4)
    //将Iterator转换成List
    val lst7 = it.toList
    //将多个list压扁成一个List
    val lst8 = lst7.flatten

    //先按空格切分，在压平
    val a = Array("a b c", "d e f", "h i j")
    a.flatMap(_.split(" "))

    lst0.reduce(_+_)
    lst0.fold(10)(_+_)

    //并行计算求和
    lst0.par.sum
    lst0.par.map(_ % 2 == 0)
    lst0.par.reduce((x, y) => x + y)
    //化简：reduce
    //将非特定顺序的二元操作应用到所有元素
    val lst9 = lst0.par.reduce((x, y) => x + y)
    //按照特定的顺序
    val lst10 = lst0.reduceLeft(_+_)

    //折叠：有初始值（无特定顺序）
    val lst11 = lst0.par.fold(100)((x, y) => x + y)
    //折叠：有初始值（有特定顺序）
    val lst12 = lst0.foldLeft(100)((x, y) => x + y)


    //聚合
    val arr = List(List(1, 2, 3), List(3, 4, 5), List(2), List(0))


    val agg1 = arr.aggregate(0)(_+_.sum, _+_)
    val agg2 = arr.par.aggregate(0)(_+_.par.sum, _+_)

    val l1 = List(5,6,4,7)
    val l2 = List(1,2,3,4,5)
    //求并集
    val r1 = l1.union(l2)
    //求交集
    val r2 = l1.intersect(l2)
    //求差集
    val r3 = l1.diff(l2)
    println(r3)
    lst0.sorted
  }
}
```

[scala的交集、并集、差集](https://blog.csdn.net/qq_37332702/article/details/87202858)

# 3. Scala 单例对象
在 Scala 中，是没有 static 这个东西的，但是它也为我们提供了单例模式的实现方法，那就是使用关键字 object。

Scala 中使用单例模式时，除了定义的类之外，还要定义一个同名的 object 对象，它和类的区别是，<font style="background-color: LawnGreen;">object对象不能带参数</font>。

当单例对象与某个类共享同一个名称时，他被称作是这个类的<font style="background-color: Cyan;">伴生对象</font>：companion object。你必须在同一个源文件里定义类和它的伴生对象。类被称为是这个单例对象的<font style="background-color: Cyan;">伴生类</font>：companion class。类和它的伴生对象可以互相访问其私有成员。

## 3.1 单例对象实例

```scala
import java.io._

class Point(val xc: Int, val yc: Int) {
   var x: Int = xc
   var y: Int = yc
   def move(dx: Int, dy: Int) {
      x = x + dx
      y = y + dy
   }
}

object Test {
   def main(args: Array[String]) {
      val point = new Point(10, 20)
      printPoint

      def printPoint{
         println ("x 的坐标点 : " + point.x);
         println ("y 的坐标点 : " + point.y);
      }
   }
}
```

执行以上代码，输出结果为：

```scala
$ scalac Test.scala 
$ scala Test
x 的坐标点 : 10
y 的坐标点 : 20
```

## 3.2 伴生对象实例

```scala
// 私有构造方法
class Marker private(val color:String) {

  println("创建" + this)
  
  override def toString(): String = "颜色标记："+ color
  
}

// 伴生对象，与类共享名字，可以访问类的私有属性和方法
object Marker{
  
    private val markers: Map[String, Marker] = Map(
      "red" -> new Marker("red"),
      "blue" -> new Marker("blue"),
      "green" -> new Marker("green")
    )
    
    def apply(color:String) = {
      if(markers.contains(color)) markers(color) else null
    }
  
    
    def getMarker(color:String) = { 
      if(markers.contains(color)) markers(color) else null
    }
    def main(args: Array[String]) { 
        println(Marker("red"))  
        // 单例函数调用，省略了.(点)符号  
        println(Marker getMarker "blue")  
    }
}
```

执行以上代码，输出结果为：

```scala
$ scalac Marker.scala 
$ scala Marker
创建颜色标记：red
创建颜色标记：blue
创建颜色标记：green
颜色标记：red
颜色标记：blue
```

# 4. 模式匹配和样例类
Scala有一个十分强大的模式匹配机制，可以应用到很多场合：如switch语句、类型检查等。
并且Scala还提供了样例类，对模式匹配进行了优化，可以快速进行匹配

## 4.1 Scala 模式匹配
一个模式匹配包含了一系列备选项，每个都开始于关键字 **case**。每个备选项都包含了一个模式及一到多个表达式。箭头符号 **=>** 隔开了模式和表达式。

参考：https://docs.scala-lang.org/zh-cn/tour/pattern-matching.html

### 4.1.1 配置字符串，匹配内容


```scala
object CaseDemo01 extends App {

  val arr = Array("YoshizawaAkiho", "YuiHatano", "AoiSola")

  val i = Random.nextInt(arr.length)
  println(i)
  val name = arr(i)
  println(name)
  name match {
    case "YoshizawaAkiho" => println("吉泽老师...")
    case "YuiHatano" => {
      println("我的女神...")
      println("波多老师...")
    }
    case "aaa" | "bbb" =>  println("连续字符串...")
    case _ => println("真不知道你们在说什么...")
  }
}
```

match 对应 Java 里的 switch，但是写在选择器表达式之后。即： **选择器 match {备选项}**。

match 表达式通过以代码编写的先后次序尝试每个模式来完成计算，只要发现有一个匹配的case，剩下的case不会继续匹配。

### 4.1.2 匹配不同数据类型


```scala
object CaseDemo02 extends App{

  //定义一个数组
  val arr:Array[Any] = Array("hello123", 1, -2.0, CaseDemo02, 2L)

  //取出一个元素
  val elem = arr(4)

  println(elem)

  elem match {
    case x: Int => println("Int " + x)
    case y: Double if(y <= 0) => println("Double "+ y)
    case z: String => println("String " + z)
    case w: Long => println("long " + w)
    case CaseDemo02 => {
      val c = CaseDemo02
      println(c)
      println("case demo 2")
      //throw new Exception("not match exception")
    }
    case _ => {
      println("no")
      println("default")
    }
  }

}
```

### 4.1.3 匹配数组、元组

数组分为头尾两部分：arr[0]为头，其余的都是尾

- Nil 空列表
- :: Nil 头与尾（空list）合成一个新list
  - <font color="red">操作符是右结合的</font>，如9 :: 5 :: 2 :: Nil相当于 <font color="red">9 :: (5 :: (2 :: Nil))</font>

println(s"name : $f") 打印“name : XXX”     $f，获取f的值

```scala
object CaseDemo03 extends App{

  val arr = Array(1, 2, 7, 0)
  arr match {
    case Array(0, 2, x, y) => println(x + " " + y)
    case Array(1, 1, 7, y) => println("only 0 " + y)
    case Array(1, 1, 7, 0) => println("0 ...")
    case _ => println("something else")
  }

  val lst = List(0, 3, 6, 7)

  println(lst.head)
  println(lst.tail)

  println(lst)

  lst match {
    case 0 :: Nil => println("only 0")
    case x :: y :: Nil => println(s"x $x y $y")
    case 0 :: f => println(s"name : $f")
    case _ => println("something else")
  }

  val tup = (6, 3, 5)
  tup match {
    case (1, x, y) => println(s"hello 123 $x , $y")
    case (_, w, 5) => println(w)
    case  _ => println("else")
  }

  val lst1 = 9 :: (5 :: (2 :: Nil))
  val lst2 = 9 :: 5 :: 2 :: List()
  println(lst2)


  val t = Array(("a", 100, 1.0), ("b", 200, 2.0))


  val t = Array(("a", 100), ("b", 200, 2.0))

  val r = t.map(tp => (tp._1, tp._2 * 10, tp._3 * 100))

  val r = t.map(tp => (tp._1, tp._2 + tp._3))

  println(r.toBuffer)

  val r = t.map {
    case (a, b) => (a, b)
    case (x, y, z) => (x, y + z)
  }
  println(r)

}
```

### 4.1.4 模式匹配+正则表达式

```scala
def getTelephone1(str: String): Boolean = {
    val regex_str = "^(13[0-9]|14[0,1,5-9]|15[0-3,5-9]|16[2,5,6,7]|17[0-8]|18[0-9]||19[1,8,9])\\d{8}$"
//    val regex: Regex = regex_str.r
//    val flag = regex.pattern.matcher(str).matches()
    val matcher = Pattern.compile(regex_str).matcher(str)
    val flag = matcher.matches()
    flag
  }

  def getTelephone2(str: String): Boolean = {
    val Pattern = "^(13[0-9]|14[5,7]|15[0-3,5-9]|17[0,3,5-8]|18[0-9]|166|198|199|147)\\d{8}$".r
    val flag = str match {
      case Pattern(str) => true
      case _ => false
    }
    flag
  }
```

> 补充：Scala 正则过滤
> ```Scala
> 
> # 去除所有非数字字符
> str.replaceAll("\\D", "")
> 
> # spark配合正则表达式过滤
> val regex: Regex = """[a-zA-Z]*""".r
> val filtered = rdd.filter(line => regex.pattern.matcher(line).matches)
> 
> val regexStr = "^[6-9]\\d{9}$"
> val bool1 = Pattern.matches(regexStr, "612345678a")
> val bool2 = Pattern.compile(regexStr).matcher("6123456789").matches()
> ```


> **Scala 正则替换字符串**
> ```Scala
> val inputPath: String = "/user/ss_deploy/beijing_share/cip/common/ucr/0/2021/07/01/csv_3_0"
> val year = "2021"
> val month = "01"
>
> // 包含 yyyy/mm 格式的路径
> val regex = new Regex("\\d{4}\\/\\d{2}")
>
> val adjustPath = regex.replaceAllIn(inputPath, year + "/" + month)
> ```

### 4.1.5 spark submit的模式匹配

```
object parseDemo {
  var modeName: String = ""
  var handleDate: String = ""
  var workspace: String = ""
  var province: String = ""
  var logBasePath: String = ""
  var processLogPath: String = ""

  def main(args: Array[String]): Unit = {
    val strings = List("--name", "poi", "-d", "20210708", "-w", "hdfs:///user/ss_deploy/workspace/ss-ng/beijing")
    parse(strings)
    println((modeName, handleDate, workspace, province, logBasePath))
  }

  def parse(args: List[String]): Unit = {
    args match {
      case ("--name" | "-n") :: value :: tail =>
        modeName = value
        parse(tail)

      case ("--day" | "-d") :: value :: tail =>
        handleDate = value
        parse(tail)

      case ("--workspace" | "-w") :: value :: tail =>
        workspace = value
        if (value == '/') {
          workspace = value.dropRight(1)
        }
    }
  }
  
}
```

`::` 用于的是向队列的头部追加数据,产生新的列表, `x::list`,x就会添加到list的头部

`case ("--name" | "-n") :: value :: tail`,就是匹配的==list集合结构==为 **以`--name`或`-n`开头，第二位拼接了一个字符串（value），后面再拼接一个list集合（tail）**

`case ("--name" | "-n") :: value :: tail`匹配后，value=`"poi"`，tail=`List("-d", "20210708", "-w", "hdfs:///user/ss_deploy/workspace/ss-ng/beijing")`

## 4.2 样例类
样例类(case classes)，样例类是种特殊的类，经过优化以用于模式匹配。

在伴生对象中提供了apply方法，所以可以不使用new关键字就可构建对象，已经实现了序列化接口

case class是多例的，后面要跟构造参数，case object是单例的

### 4.2.1 案例一：

案例类（case classes）的匹配

案例类非常适合用于模式匹配。


```Scala
abstract class Notification

case class Email(sender: String, title: String, body: String) extends Notification

case class SMS(caller: String, message: String) extends Notification

case class VoiceRecording(contactName: String, link: String) extends Notification
```

Notification 是一个虚基类，它有三个具体的子类Email, SMS和VoiceRecording，我们可以在这些案例类(Case Class)上像这样使用模式匹配：


```Scala
object caseClassesTest {

  def main(args: Array[String]): Unit = {

    val someSms = SMS("12345", "Are you there?")
    val someVoiceRecording = VoiceRecording("Tom", "voicerecording.org/id/123")

    println(showNotification(someSms)) // prints You got an SMS from 12345! Message: Are you there?
    println(showNotification(someVoiceRecording)) // you received a Voice Recording from Tom! Click the link to hear it: voicerecording.org/id/123
  }

  def showNotification(notification: Notification): String = {
    notification match {
      case Email(sender, title, _) =>
        s"You got an email from $sender with title: $title"
      case SMS(number, message) =>
        s"You got an SMS from $number! Message: $message"
      case VoiceRecording(name, link) =>
        s"you received a Voice Recording from $name! Click the link to hear it: $link"
    }
  }
}
```


showNotification函数接受一个抽象类Notification对象作为输入参数，然后匹配其具体类型。（也就是判断它是一个Email，SMS，还是VoiceRecording）。在case Email(sender, title, _)中，对象的sender和title属性在返回值中被使用，而body属性则被忽略，故使用_代替。

#### 4.2.1.1 模式守卫（Pattern gaurds）

为了让匹配更加具体，可以使用模式守卫，也就是在模式后面加上if <boolean expression>。


```Scala
def showImportantNotification(notification: Notification, importantPeopleInfo: Seq[String]): String = {
  notification match {
    case Email(sender, _, _) if importantPeopleInfo.contains(sender) =>
      "You got an email from special someone!"
    case SMS(number, _) if importantPeopleInfo.contains(number) =>
      "You got an SMS from special someone!"
    case other =>
      showNotification(other) // nothing special, delegate to our original showNotification function
  }
}

val importantPeopleInfo = Seq("867-5309", "jenny@gmail.com")

val someSms = SMS("867-5309", "Are you there?")
val someVoiceRecording = VoiceRecording("Tom", "voicerecording.org/id/123")
val importantEmail = Email("jenny@gmail.com", "Drinks tonight?", "I'm free after 5!")
val importantSms = SMS("867-5309", "I'm here! Where are you?")

println(showImportantNotification(someSms, importantPeopleInfo))
println(showImportantNotification(someVoiceRecording, importantPeopleInfo))
println(showImportantNotification(importantEmail, importantPeopleInfo))
println(showImportantNotification(importantSms, importantPeopleInfo))
```


在case Email(sender, _, _) if importantPeopleInfo.contains(sender)中，除了要求notification是Email类型外，还需要sender在重要人物列表importantPeopleInfo中，才会匹配到该模式。

### 4.2.2 案例二：
```scala
import scala.util.Random
//样例类，模式匹配，封装数据（多例）,不用new即可创建实例
case class SubmitTask(id: String, name: String)

case class HeartBeat(time: Long)

//样例对象，模式匹配（单例）
case object CheckTimeOutTask

object CaseDemo04 extends App{

  val arr = Array(CheckTimeOutTask, new HeartBeat(123), HeartBeat(88888), new HeartBeat(666), SubmitTask("0001", "task-0001"))


//  val a = CheckTimeOutTask
//  val b = CheckTimeOutTask
  //0 1 2 3 4
  //val i = Random.nextInt(arr.length)
  //println(i)
  val a = arr(2)
  println(a)
  a match {
    case SubmitTask(id, name) => {
      println(s"$id, $name")
    }
    case HeartBeat(time) => {
      println(time)
    }
    case CheckTimeOutTask => {
      println("check")
    }
  }
}
```


**Option：Some（多例）、None**


```scala
object MapTest {

  def main(args: Array[String]): Unit = {

    //Some None
    //Some代表有（多例），样例类
    //None代表没有（单例），样例对象
    //Option是Some和None的父类


    val mp = Map("a" -> 1, "b" -> 2, "c" -> 3)

    //val r: Int = mp("d")

    //println(r)

    //val r: Option[Int] = mp.get("d")

    //println(r)

    //val r1 = r.get

    //println(r)


    val r = mp.getOrElse("d", -1)

    println(r)

  }
}
```
