[TOC]


[Scala 可变和不可变 集合继承层级](https://www.cnblogs.com/sowhat1412/p/12734169.html)

不可变集合继承图
```
graph LR
Traversable((Traversable))
Traversable-->Iterable
Iterable-->Set
Iterable-->Seq
Iterable-->Map
Set-->HashSet
Set-->SortedSet
Set-->BitSet
Set-->ListSet
SortedSet-->TreeSet
Seq-->IndexedSeq
Seq-->LinearSeq
IndexedSeq-->Vector
IndexedSeq-->NumericRange
IndexedSeq-.->Array
IndexedSeq-.->String
IndexedSeq-->Range
LinearSeq-->List
LinearSeq-->Stream
LinearSeq-->Queue
LinearSeq-->Stack
Map-->HashMap
Map-->SortedMap
Map-->ListMap
SortedMap-->TreeMap
style HashSet fill:SkyBlue
style TreeSet fill:SkyBlue
style BitSet fill:SkyBlue
style ListSet fill:SkyBlue
style HashMap fill:SkyBlue
style ListMap fill:SkyBlue
style TreeMap fill:SkyBlue
style Vector fill:SkyBlue
style NumericRange fill:SkyBlue
style Array fill:SkyBlue
style String fill:SkyBlue
style Range fill:SkyBlue
style List fill:SkyBlue
style Stream fill:SkyBlue
style Queue fill:SkyBlue
style Stack fill:SkyBlue
```

可变集合继承图
```
graph LR
Traversable((Traversable))
Traversable-->Iterable
Iterable-->Map
Iterable-->Seq
Iterable-->Set
Map-->HashMap
Map-->WeakHashMap
Map-->OpenHashMap
Map-->LinkedHashMap
Map-->ObservableMap
Map-->SynchronizedMap
Map-->ImmutableMapAdaptor
Map-->ListMap
Map-->MultiMap
Seq-->IndexedSeq
Seq-->Buffer
Seq-->Stack
Seq-->ArrayStack
Seq-->PriorityQueue
Seq-->LinearSeq
IndexedSeq-->ArraySeq
IndexedSeq-->StringBuilder
IndexedSeq-->ArrayBuffer
Buffer-->ArrayBuffer
Buffer-->ObservableBuffer
Buffer-->SynchronizedBuffer
Buffer-->ListBuffer
Stack-->SynchronizedStack
PriorityQueue-->SynchronizedPriorityQueue
LinearSeq-->MutableList
LinearSeq-->LinkedList
LinearSeq-->DoubleLinkedList
MutableList-->Queue
Queue-->SynchronizedQueue
Set-->HashSet
Set-->BitSet
Set-->ObservableSet
Set-->SynchronizedSet
Set-->ImmutableSetAdaptor
Set-->LinkedHashSet
style HashMap fill:SkyBlue
style WeakHashMap  fill:SkyBlue
style OpenHashMap fill:SkyBlue
style LinkedHashMap fill:SkyBlue
style ImmutableMapAdaptor fill:SkyBlue
style HashSet fill:SkyBlue
style BitSet fill:SkyBlue
style ImmutableSetAdaptor fill:SkyBlue
style LinkedHashSet fill:SkyBlue
style ArraySeq fill:SkyBlue
style StringBuilder fill:SkyBlue
style ArrayBuffer fill:SkyBlue
style ListBuffer fill:SkyBlue
style Stack fill:SkyBlue
style ArrayStack fill:SkyBlue
style PriorityQueue fill:SkyBlue
style SynchronizedStack fill:SkyBlue
style SynchronizedPriorityQueue fill:SkyBlue
style MutableList fill:SkyBlue
style LinkedList fill:SkyBlue
style DoubleLinkedList fill:SkyBlue
style SynchronizedQueue fill:SkyBlue
```



# 数组
## 声明数组

以下是 Scala 数组声明的语法格式：

```Scala
var z:Array[String] = new Array[String](3)
或
var z = new Array[String](3)
```


以上语法中，z 声明一个字符串类型的数组，数组长度为 3 ，可存储 3 个元素。我们可以为每个元素设置值，并通过索引来访问每个元素，如下所示：

```scala
z(0) = "Runoob"; z(1) = "Baidu"; z(4/2) = "Google"
```


**最后一个元素的索引使用了表达式 4/2 作为索引，类似于 z(2) = "Google"。**

我们也可以使用以下方式来定义一个数组：

```scala
var z = Array("Runoob", "Baidu", "Google")
```

下图展示了一个长度为 10 的数组 myList，索引值为 0 到 9：
![长度为10的数组](https://www.runoob.com/wp-content/uploads/2013/12/java_array.jpg)

## 处理数组

数组的元素类型和数组的大小都是确定的，所以当处理数组元素时候，我们通常使用基本的 for 循环。

以下实例演示了数组的创建，初始化等处理过程：

```Scala
object Test {
   def main(args: Array[String]) {
      var myList = Array(1.9, 2.9, 3.4, 3.5)
      
      // 输出所有数组元素
      for ( x <- myList ) {
         println( x )
      }

      // 计算数组所有元素的总和
      var total = 0.0;
      for ( i <- 0 to (myList.length - 1)) {
         total += myList(i);
      }
      println("总和为 " + total);

      // 查找数组中的最大元素
      var max = myList(0);
      for ( i <- 1 to (myList.length - 1) ) {
         if (myList(i) > max) max = myList(i);
      }
      println("最大值为 " + max);
      
      
      // 数组添加元素
      val newArr = myList :+ (1.1)
      newArr.foreach(println)
    
   }
}
```

执行以上代码，输出结果为：

```scala
1.9
2.9
3.4
3.5
总和为 11.7
最大值为 3.5

1.9
2.9
3.4
3.5
1.1
```
## 数组转字符串

**字符串添加元素**
```scala

  def arrayTest(): Unit = {
    val str = "a|b|c|d|e|2020-02-10 12:00:00|2020-02-10 12:10:00|10"
    val arr: Array[String] = str.split("\\|", -1)
    arr(4) = arr(4) + "|add"
    val value: String = arr.mkString("|")

    println(value)
 }   
    
输出结果：    
a|b|c|d|e|add|2020-02-10 12:00:00|2020-02-10 12:10:00|10

```

**字符串反向截取**
```Scala
def arrayTest2(): Unit = {
    val str = "a|b|c|d|e|2020-02-10 12:00:00|2020-02-10 12:10:00|10"
    val arr: Array[String] = str.split("\\|", -1)
    val string = arr.dropRight(3).mkString("|")

    println(string)
  }
  
输出结果：    
a|b|c|d|e
```

## 多维数组

多维数组一个数组中的值可以是另一个数组，另一个数组的值也可以是一个数组。矩阵与表格是我们常见的二维数组。

实例中数组中包含三个数组元素，每个数组元素又含有三个值。

```scala
import Array._

object Test {
   def main(args: Array[String]) {
      var myMatrix = ofDim[Int](3,3)
      
      // 创建矩阵
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            myMatrix(i)(j) = j;
         }
      }
      
      // 打印二维阵列
      for (i <- 0 to 2) {
         for ( j <- 0 to 2) {
            print(" " + myMatrix(i)(j));
         }
         println();
      }
    
   }
}
```

输出结果为：

```Scala
0 1 2
0 1 2
0 1 2
```

## 合并数组

以下实例中，我们使用 concat() 方法来合并两个数组，concat() 方法中接受多个数组参数：

```scala
import Array._

object Test {
   def main(args: Array[String]) {
      var myList1 = Array(1.9, 2.9, 3.4, 3.5)
      var myList2 = Array(8.9, 7.9, 0.4, 1.5)

      var myList3 =  concat( myList1, myList2)
      
      // 输出所有数组元素
      for ( x <- myList3 ) {
         println( x )
      }
   }
}
```

执行以上代码，输出结果为：

```
1.9
2.9
3.4
3.5
8.9
7.9
0.4
1.5
```


## 创建区间数组
### 1.range
以下实例中，我们使用了`range(start: Int, end: Int, step: Int)`方法来生成一个区间范围内的数组。range() 方法最后一个参数为步长，默认为 1：

range生成的数组，包含start，但是不含end值，包头不包尾

```scala

object Test {
   def main(args: Array[String]) {
      val myList1 = Array.range(10, 20, 2)
      val myList2 = Array.range(10,20)

      // 输出所有数组元素
      for ( x <- myList1 ) {
         print( " " + x )
      }
      println()
      for ( x <- myList2 ) {
         print( " " + x )
      }
   }
}
```


执行以上代码，输出结果为：

```
10 12 14 16 18
10 11 12 13 14 15 16 17 18 19
```

### 2.fill

如果要创建具有相同数据的多个元素的列表，则将使用fill()方法，填充数组

语法：`fill(count)(element)`
- count是数组中元素的数量。
- element是列表中重复的元素

示例
```scala
val myList3 = Array.fill(5)("hello")

// -------------------
//  hello hello hello hello hello
```

### 3.tabulate

如果要使用含递增序号的元素创建数组，则将一组值传递给函数

语法：`tabulate(count)(function)`
- count是数组中元素的数量
- function传递计数元素的函数，取值范围为从0到n-1

示例
```scala
val myList4 = Array.tabulate(5)(n=>"id_"+n)


// -------------------
//  id_0 id_1 id_2 id_3 id_4
```

## Scala 数组方法
下表中为 Scala 语言中处理数组的重要方法，使用它前我们需要使用 **import Array._** 引入包。


<html>
<table class="reference">
<tbody><tr>
<th style="width:5%">序号</th>
<th style="width:95%">方法和描述</th>
</tr>
<tr>
<td>1</td>
<td>
<p><b>def apply( x: T, xs: T* ): Array[T]</b></p>
<p>创建指定对象 T 的数组,  T 的值可以是 Unit, Double, Float, Long, Int, Char, Short, Byte, Boolean。</p>
</td>
</tr>
<tr>
<td>2</td>
<td>
<p><b>def concat[T]( xss: Array[T]* ): Array[T]</b></p>
<p>合并数组</p>
</td>
</tr>
<tr>
<td>3</td>
<td>
<p><b>def copy( src: AnyRef, srcPos: Int, dest: AnyRef, destPos: Int, length: Int ): Unit</b></p>
<p>复制一个数组到另一个数组上。相等于 Java's System.arraycopy(src, srcPos, dest, destPos, length)。</p>
</td>
</tr>
<tr>
<td>4</td>
<td>
<p><b>def empty[T]: Array[T]</b></p>
<p>返回长度为 0 的数组</p>
</td>
</tr>
<tr>
<td>5</td>
<td>
<p><b>def iterate[T]( start: T, len: Int )( f: (T) =&gt; T ): Array[T]</b></p>
<p>返回指定长度数组，每个数组元素为指定函数的返回值。</p>
<p>以上实例数组初始值为 0，长度为 3，计算函数为<b>a=&gt;a+1</b>：</p>
<pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">.</span><span class="pln">iterate</span><span class="pun">(</span><span class="lit">0</span><span class="pun">,</span><span class="lit">3</span><span class="pun">)(</span><span class="pln">a</span><span class="pun">=&gt;</span><span class="pln">a</span><span class="pun">+</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
res1</span><span class="pun">:</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">(</span><span class="lit">0</span><span class="pun">,</span><span class="pln"> </span><span class="lit">1</span><span class="pun">,</span><span class="pln"> </span><span class="lit">2</span><span class="pun">)</span></pre>
</td>
</tr>
<tr>
<td>6</td>
<td>
<p><b>def fill[T]( n: Int )(elem: =&gt;  T): Array[T]</b></p>
<p>返回数组，长度为第一个参数指定，同时每个元素使用第二个参数进行填充。</p>
</td>
</tr>
<tr>
<td>7</td>
<td>
<p><b>def fill[T]( n1: Int, n2: Int )( elem: =&gt; T ): Array[Array[T]]</b></p>
<p>返回二数组，长度为第一个参数指定，同时每个元素使用第二个参数进行填充。</p>
</td>
</tr>

<tr>
<td>8</td>
<td>
<p><b>def ofDim[T]( n1: Int ): Array[T]</b></p>
<p>创建指定长度的数组</p>
</td>
</tr>
<tr>
<td>9</td>
<td>
<p><b>def ofDim[T]( n1: Int, n2: Int ): Array[Array[T]]</b></p>
<p>创建二维数组</p>
</td>
</tr>
<tr>
<td>10</td>
<td>
<p><b>def ofDim[T]( n1: Int, n2: Int, n3: Int ): Array[Array[Array[T]]]</b></p>
<p>创建三维数组</p>
</td>
</tr>
<tr>
<td>11</td>
<td>
<p><b>def range( start: Int, end: Int, step: Int ): Array[Int]</b></p>
<p>创建指定区间内的数组，step 为每个元素间的步长</p>
</td>
</tr>
<tr>
<td>12</td>
<td>
<p><b>def range( start: Int, end: Int ): Array[Int]</b></p>
<p>创建指定区间内的数组</p>
</td>
</tr>
<tr>
<td>13</td>
<td>
<p><b>def tabulate[T]( n: Int )(f: (Int)=&gt; T): Array[T]</b></p>
<p>返回指定长度数组，每个数组元素为指定函数的返回值，默认从 0 开始。</p>
<p>以上实例返回 3 个元素：</p>
<pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">.</span><span class="pln">tabulate</span><span class="pun">(</span><span class="lit">3</span><span class="pun">)(</span><span class="pln">a </span><span class="pun">=&gt;</span><span class="pln"> a </span><span class="pun">+</span><span class="pln"> </span><span class="lit">5</span><span class="pun">)</span><span class="pln">
res0</span><span class="pun">:</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">Array</span><span class="pun">(</span><span class="lit">5</span><span class="pun">,</span><span class="pln"> </span><span class="lit">6</span><span class="pun">,</span><span class="pln"> </span><span class="lit">7</span><span class="pun">)</span></pre>
</td>
</tr>
<tr>
<td>14</td>
<td>
<p><b>def tabulate[T]( n1: Int, n2: Int )( f: (Int, Int ) =&gt; T): Array[Array[T]]</b></p>
<p>返回指定长度的二维数组，每个数组元素为指定函数的返回值，默认从 0 开始。</p>
</td>
</tr>
</tbody></table>
</html>


# 集合

集合操作的通用方法:
- 带+与带-的区别:
  - 带+是添加元素
  - 带-是删除元素
- 一个+/-与两个+/-的区别:
  - 一个+/-是添加/删除单个元素
  - 两个+/-是添加/删除一个集合所有元素
- 冒号在前、冒号在后、不带冒号的区别:
  - 冒号在前是将元素添加到集合末尾
  - 冒号在后是将元素添加到集合最前面
  - 不带冒号是将元素添加到集合末尾
- 带=与不带=的区别:
  - 带=是修改集合本身
  - 不带=是生成一个新集合,原集合没有改变
- update与updated的区别:
  - update是修改集合本身
  - updated是生成一个新集合,原集合没有改变

## List

scala的集合中有两种List:

```scala
scala.collection.immutable.List  //长度内容都不可变
scala.collection.mutable.ListBuffer //长度内容都可变，必须导入包
```

注意：immutable下没有ListBuffer;mutable下没有List;

```scala
scala> val numbers = List("1", "2","3", "4")
numbers: List[Int] = List("1", "2","3", "4")
```


<html>
<p>下表列出了 Scala List 常用的方法：</p>
<table class="reference">
<tbody><tr>
<th style="width:5%">序号</th>
<th style="width:95%">方法及描述</th>
</tr>
<tr>
<td>1</td>
<td>
<p><b>def +:(elem: A): List[A]</b></p>

<p>为列表预添加元素</p>
<pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> val x </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
x</span><span class="pun">:</span><span class="pln"> </span><span class="typ">List</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span><span class="pln">

scala</span><span class="pun">&gt;</span><span class="pln"> val y </span><span class="pun">=</span><span class="pln"> </span><span class="lit">2</span><span class="pln"> </span><span class="pun">+:</span><span class="pln"> x
y</span><span class="pun">:</span><span class="pln"> </span><span class="typ">List</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">2</span><span class="pun">,</span><span class="pln"> </span><span class="lit">1</span><span class="pun">)</span><span class="pln">

scala</span><span class="pun">&gt;</span><span class="pln"> println</span><span class="pun">(</span><span class="pln">x</span><span class="pun">)</span><span class="pln">
</span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span></pre>
</td>
</tr>
<tr>
<td>2</td>
<td>
<p><b>def ::(x: A): List[A]</b></p>
<p>在列表开头添加元素</p>
</td>
</tr>
<tr>
<td>3</td>
<td>
<p><b>def :::(prefix: List[A]): List[A]</b></p>
<p>在列表开头添加指定列表的元素</p>
</td>
</tr>
<tr>
<td>4</td>
<td>
<p><b>def :+(elem: A): List[A]</b></p>
<p>复制添加元素后列表。 </p>
<pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> val a </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span><span class="pln">
a</span><span class="pun">:</span><span class="pln"> </span><span class="typ">List</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span><span class="pln">

scala</span><span class="pun">&gt;</span><span class="pln"> val b </span><span class="pun">=</span><span class="pln"> a </span><span class="pun">:+</span><span class="pln"> </span><span class="lit">2</span><span class="pln">
b</span><span class="pun">:</span><span class="pln"> </span><span class="typ">List</span><span class="pun">[</span><span class="typ">Int</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">,</span><span class="pln"> </span><span class="lit">2</span><span class="pun">)</span><span class="pln">

scala</span><span class="pun">&gt;</span><span class="pln"> println</span><span class="pun">(</span><span class="pln">a</span><span class="pun">)</span><span class="pln">
</span><span class="typ">List</span><span class="pun">(</span><span class="lit">1</span><span class="pun">)</span></pre>
</td>
</tr>
<tr>
<td>5</td>
<td>
<p><b>def addString(b: StringBuilder): StringBuilder</b></p>
<p>将列表的所有元素添加到 StringBuilder</p>
</td>
</tr>
<tr>
<td>6</td>
<td>
<p><b>def addString(b: StringBuilder, sep: String): StringBuilder</b></p>
<p>将列表的所有元素添加到 StringBuilder，并指定分隔符</p>
</td>
</tr>
<tr>
<td>7</td>
<td>
<p><b>def apply(n: Int): A</b></p>
<p>通过列表索引获取元素</p>
</td>
</tr>
<tr>
<td>8</td>
<td>
<p><b>def contains(elem: Any): Boolean</b></p>
<p>检测列表中是否包含指定的元素</p>
</td>
</tr>
<tr>
<td>9</td>
<td>
<p><b>def copyToArray(xs: Array[A], start: Int, len: Int): Unit</b></p>
<p>将列表的元素复制到数组中。</p>
</td>
</tr>
<tr>
<td>10</td>
<td>
<p><b>def distinct: List[A]</b></p>
<p>去除列表的重复元素，并返回新列表</p>
</td>
</tr>
<tr>
<td>11</td>
<td>
<p><b>def drop(n: Int): List[A]</b></p>
<p>丢弃前n个元素，并返回新列表</p>
</td>
</tr>
<tr>
<td>12</td>
<td>
<p><b>def dropRight(n: Int): List[A]</b></p>
<p>丢弃最后n个元素，并返回新列表</p>
</td>
</tr>
<tr>
<td>13</td>
<td>
<p><b>def dropWhile(p: (A) =&gt; Boolean): List[A]</b></p>
<p>从左向右丢弃元素，直到条件p不成立</p>
</td>
</tr>
<tr>
<td>14</td>
<td>
<p><b>def endsWith[B](that: Seq[B]): Boolean</b></p>
<p>检测列表是否以指定序列结尾</p>
</td>
</tr>
<tr>
<td>15</td>
<td>
<p><b>def equals(that: Any): Boolean</b></p>
<p>判断是否相等</p>
</td>
</tr>
<tr>
<td>16</td>
<td>
<p><b>def exists(p: (A) =&gt; Boolean): Boolean</b></p>
<p>判断列表中指定条件的元素是否存在。</p>
<p>判断l是否存在某个元素:
</p><pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> l</span><span class="pun">.</span><span class="pln">exists</span><span class="pun">(</span><span class="pln">s </span><span class="pun">=&gt;</span><span class="pln"> s </span><span class="pun">==</span><span class="pln"> </span><span class="str">"Hah"</span><span class="pun">)</span><span class="pln">
res7</span><span class="pun">:</span><span class="pln"> </span><span class="typ">Boolean</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="kwd">true</span></pre>
</td>
</tr>
<tr>
<td>17</td>
<td>
<p><b>def filter(p: (A) =&gt; Boolean): List[A]</b></p>
<p>输出符号指定条件的所有元素。</p>
<p>过滤出长度为3的元素:</p>
<pre class="prettyprint prettyprinted" style=""><span class="pln">scala</span><span class="pun">&gt;</span><span class="pln"> l</span><span class="pun">.</span><span class="pln">filter</span><span class="pun">(</span><span class="pln">s </span><span class="pun">=&gt;</span><span class="pln"> s</span><span class="pun">.</span><span class="pln">length </span><span class="pun">==</span><span class="pln"> </span><span class="lit">3</span><span class="pun">)</span><span class="pln">
res8</span><span class="pun">:</span><span class="pln"> </span><span class="typ">List</span><span class="pun">[</span><span class="typ">String</span><span class="pun">]</span><span class="pln"> </span><span class="pun">=</span><span class="pln"> </span><span class="typ">List</span><span class="pun">(</span><span class="typ">Hah</span><span class="pun">,</span><span class="pln"> WOW</span><span class="pun">)</span></pre>
</td>
</tr>
<tr>
<td>18</td>
<td>
<p><b>def forall(p: (A) =&gt; Boolean): Boolean</b></p>
<p>检测所有元素。</p>
<p>
例如：判断所有元素是否以"H"开头：</p>

scala&gt; l.forall(s =&gt; s.startsWith("H"))
res10: Boolean = false

</td>
</tr>
<tr>
<td>19</td>
<td>
<p><b>def foreach(f: (A) =&gt; Unit): Unit</b></p>
<p>将函数应用到列表的所有元素</p>
</td>
</tr>
<tr>
<td>20</td>
<td>
<p><b>def head: A</b></p>
<p>获取列表的第一个元素</p>
</td>
</tr>
<tr>
<td>21</td>
<td>
<p><b>def indexOf(elem: A, from: Int): Int</b></p>
<p>从指定位置 from
开始查找元素第一次出现的位置</p>
</td>
</tr>
<tr>
<td>22</td>
<td>
<p><b>def init: List[A]</b></p>
<p>返回所有元素，除了最后一个</p>
</td>
</tr>
<tr>
<td>23</td>
<td>
<p><b>def intersect(that: Seq[A]): List[A]</b></p>
<p>计算多个集合的交集</p>
</td>
</tr>
<tr>
<td>24</td>
<td>
<p><b>def isEmpty: Boolean</b></p>
<p>检测列表是否为空</p>
</td>
</tr>
<tr>
<td>25</td>
<td>
<p><b>def iterator: Iterator[A]</b></p>
<p>创建一个新的迭代器来迭代元素</p>
</td>
</tr>
<tr>
<td>26</td>
<td>
<p><b>def last: A</b></p>
<p>返回最后一个元素</p>
</td>
</tr>
<tr>
<td>27</td>
<td>
<p><b>def lastIndexOf(elem: A, end: Int): Int</b></p>
<p>在指定的位置 end 开始查找元素最后出现的位置</p>
</td>
</tr>
<tr>
<td>28</td>
<td>
<p><b>def length: Int</b></p>
<p>返回列表长度</p>
</td>
</tr>
<tr>
<td>29</td>
<td>
<p><b>def map[B](f: (A) =&gt; B): List[B]</b></p>
<p>通过给定的方法将所有元素重新计算</p>
</td>
</tr>
<tr>
<td>30</td>
<td>
<p><b>def max: A</b></p>
<p>查找最大元素</p>
</td>
</tr>
<tr>
<td>31</td>
<td>
<p><b>def min: A</b></p>
<p>查找最小元素</p>
</td>
</tr>
<tr>
<td>32</td>
<td>
<p><b>def mkString: String</b></p>
<p>列表所有元素作为字符串显示</p>
</td>
</tr>
<tr>
<td>33</td>
<td>
<p><b>def mkString(sep: String): String</b></p>
<p>使用分隔符将列表所有元素作为字符串显示</p>
</td>
</tr>
<tr>
<td>34</td>
<td>
<p><b>def reverse: List[A]</b></p>
<p>列表反转</p>
</td>
</tr>
<tr>
<td>35</td>
<td>
<p><b>def sorted[B &gt;: A]: List[A]</b></p>
<p>列表排序</p>
</td>
</tr>
<tr>
<td>36</td>
<td>
<p><b>def startsWith[B](that: Seq[B], offset: Int): Boolean</b></p>
<p>检测列表在指定位置是否包含指定序列</p>
</td>
</tr>
<tr>
<td>37</td>
<td>
<p><b>def sum: A</b></p>
<p>计算集合元素之和</p>
</td>
</tr>
<tr>
<td>38</td>
<td>
<p><b>def tail: List[A]</b></p>
<p>返回所有元素，除了第一个</p>
</td>
</tr>
<tr>
<td>39</td>
<td>
<p><b>def take(n: Int): List[A]</b></p>
<p>提取列表的前n个元素</p>
</td>
</tr>
<tr>
<td>40</td>
<td>
<p><b>def takeRight(n: Int): List[A]</b></p>
<p>提取列表的后n个元素</p>
</td>
</tr>
<tr>
<td>41</td>
<td>
<p><b>def toArray: Array[A]</b></p>
<p>列表转换为数组</p>
</td>
</tr>
<tr>
<td>42</td>
<td>
<p><b>def toBuffer[B &gt;: A]: Buffer[B]</b></p>
<p>返回缓冲区，包含了列表的所有元素</p>
</td>
</tr>
<tr>
<td>43</td>
<td>
<p><b>def toMap[T, U]: Map[T, U]</b></p>
<p>List 转换为 Map</p>
</td>
</tr>
<tr>
<td>44</td>
<td>
<p><b>def toSeq: Seq[A]</b></p>
<p>List 转换为 Seq</p>
</td>
</tr>
<tr>
<td>45</td>
<td>
<p><b>def toSet[B &gt;: A]: Set[B]</b></p>
<p>List 转换为 Set</p>
</td>
</tr>
<tr>
<td>46</td>
<td>
<p><b>def toString(): String</b></p>
<p>列表转换为字符串</p>
</td>
</tr>
</tbody></table>
</html>

**1. 拉链(zip)与反拉链（unzip)**

```scala
// 拉链(zip)，两个集合之间元素交叉
val list1=List("张三","李四","王五")
val list2=List("阿娇","糖心")

val newList: List[(String, String)] = list1.zip(list2)

println("反拉链："+newList.unzip)
println("拉链："+newList)
```

```scala
// 结果：
反拉链：(List(张三, 李四),List(阿娇, 糖心))
拉链：List((张三,阿娇), (李四,糖心))
```

**2. sortWith**
- sortWith(func: (集合元素类型,集合元素类型) => Boolean )
- sortWith 中的函数如果第一个参数>第二个参数,降序
- sortWith 中的函数如果第一个参数<第二个参数,升序

根据指定规则排序【指定升序或者降序】


```scala
val list2=List[(String,Int)](
      ("张三",19),
      ("春娇",18),
      ("牛二娃",8),
      ("刘大叔",39),
      ("李二婶",42),
      ("李四",19),
    )
```

按年龄降序

```scala
println(list2.sortWith(_._2 > _._2))

List((李二婶,42), (刘大叔,39), (张三,19), (李四,19), (春娇,18), (牛二娃,8))
```

按年龄升序

```scala
println(list2.sortWith(_._2 < _._2))

List((牛二娃,8), (春娇,18), (张三,19), (李四,19), (刘大叔,39), (李二婶,42))
```

**3. updated**

不可变集合没有update方法，只有updated，返回新的List对象
```scala
val list = List("scala","spark","hadoop")
println(list.updated(2,"hadoop1"))
```

结果：<br>
List(scala, spark, hadoop1) 

## Set

Set集合:数学意义上的集合：无序的，不可重复的（归根于底层实现，使用Map）

```scala
scala> Set("1", "1", "2")
`res0: scala.collection.immutable.Set[Int] = Set("1", "2")
```

## 元组(Tuple)

元组可以直接把一些具有简单逻辑关系的一组数据组合在一起，并且不需要额外的类。

```scala
scala> val hostPort = ("localhost", 80)
hostPort: (String, Int) = ("localhost", 80)
```

和case class不同，元组的元素不能通过名称进行访问，不过它们可以通过基于它们位置的名称进行访问，这个位置是从1开始而非从0开始。

```scala
scala> hostPort._1
res0: String = localhost
scala> hostPort._2
res1: Int = ``80
```

元组可以很好地和模式匹配配合使用。
```scala
hostPort match {
  case ("localhost", port) => ...
  case (host, port) => ...
}

```
创建一个包含2个值的元组有一个很简单的方式：`->`
```scala
scala> 1 -> 2
res0: (Int, Int) = (1,2)
```

### 迭代元组
`Tuple.productIterator()` 方法来迭代输出元组的所有元素：

```scala
val t = (4,3,2,1)
  
t.productIterator.foreach{ i =>println("Value = " + i )}
 
输出结果为：
Value = 4
Value = 3
Value = 2
Value = 1
```


### 元素交换
`Tuple.swap`方法来交换元组的元素。如下实例：

```scala
val t = new Tuple2("www.google.com", "www.runoob.com")
      
println("交换后的元组: " + t.swap )

输出结果为：
交换后的元组: (www.runoob.com,www.google.com)
```


## Map

Map里可以存放基本的数据类型。
```scala
    //构建可变map
    val map = mutable.Map[String, Int]("abc" -> 123, ("xyz", 789))
     
    //取值
    if (map.contains("abc")) {
      val v1 = map("abc")
      val v2 = map.get("abc").get
    }
    val v3 = map.getOrElse("abc", 999)
 
    //添加或更新元素
    map("def") = 456
    map += ("java" -> 20, "scala" -> 30)
 
    //删除元素
    map -= ("abc", "ooo")
 
    //四种遍历
    for ((k, v) <- map) println(s"k=${k},v=${v}")
    for (k <- map.keys) println(s"k=${k}")
    for (v <- map.values) println(s"v=${v}")
    for (t <- map) println(s"k=${t._1},v=${t._2}")
```

`("abc" -> 123)`这个看起来是一个特殊的语法，不过回想一下前面我们讨论元组的时候，`->`符号是可以用来创建元组的。

Map()可以使用我们在第一节里讲到的可变参数的语法：`Map( 1 -> "one", 2 -> "two")`，它会被扩展为`Map((1,"one"),(2,"two"))`，其中第一个元素参数是key，第二个元素是value。

Map里也可以包含Map，甚至也可以把函数当作值存在Map里。

```scala
Map(1 -> Map("foo" -> "bar"))

Map("timesTwo" -> { timesTwo(_) })
```

**getOrElseUpdate**

getOrElseUpdate 其实是 getOrElse的升级版，当获取不存在的key时，不再返回默认值 default，而是根据 op 这个方法根据当前获取的不存在的key返回一个value，并将这个 key-value 存入 HashMap，这也就是 update 的由来


```scala
val testHashMap = mutable.HashMap[String,Int]()

// 定义 update 规则
def op(num: String) :Int = Try(num.toInt).getOrElse(0) * 2
def update(num: String): Int = testHashMap.getOrElseUpdate(num,op(num))

// key "1" 存在 key "2" 不存在 
testHashMap("1") = 1
val value_1 = update("1")
val value_2 = update("2")
    
// 检查执行结果
println(value_1,value_2)
println(testHashMap)

```

结果：

```
(1,4)
Map(2 -> 4, 1 -> 1)
```

> update和updated方法区别：<br>
> 在scala的mutable.Map中，存在update和updated两个方法，这两个方法的很容易打错
> 
> **update**<br>
> 其**中update方法**的作用是为map更新或添加一对新的键值对，这个添加是在原map上进行的，**原map内容会改变**
> 
> **updated**<br>
> **updated方法**也是更新或添加一对新的键值对，但是**不改变原map**，而是返回一个包含更新的新map


**==两个Map进行合并相同Key值相加6种方法==**

```Scala
object MapMergeFourMethods {
  def main(args: Array[String]): Unit = {
    val map1 = Map("a" -> 1L, "b" -> 2L)
    val map2 = Map("a" -> 1L, "c" -> 3L)
    println(method1(map1, map2))
    println(method2(map1, map2))
    println(method3(map1, map2))
    println(method4(map1, map2))
    println(method5(map1, map2))
    println(method6(map1, map2))
  }

  /**
    * 使用折叠
    */
  def method1(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    val result = map1.foldLeft(map2) {
      case (m: Map[String, Long], (k: String, v: Long)) =>
        m + (k -> (m.getOrElse(k, 0L) + v))
    }
    result
  }

  /**
    * 折叠的简写形式
    */
  def method2(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    val result = (map2 /: map1) {
      case (m: Map[String, Long], (k: String, v: Long)) =>
        m + (k -> (m.getOrElse(k, 0L) + v))
    }
    result
  }

  /**
    * 传递函数 不使用模式匹配
    */
  def method3(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    val result = (map2 /: map1) (
      (m: Map[String, Long], s: (String, Long)) =>
        m + (s._1 -> (m.getOrElse(s._1, 0L) + s._2))
    )
    result
  }

  /**
    * foreach 转为可变集合 更新后 再转回不可变
    */
  def method4(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    val immMap1 = scala.collection.mutable.Map(map1.toSeq: _*)
    val immMap2 = scala.collection.mutable.Map(map2.toSeq: _*)
    immMap1.foreach(s => {
      immMap2(s._1) = (immMap2.getOrElse(s._1, 0L) + s._2)
    })
    immMap2.toMap
  }

  /**
    * foreach
    */
  def method5(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    var a: Map[String, Long] = new HashMap[String, Long]
    a = map2
    map1.foreach(s => {
      a = a + ((s._1) -> (map2.getOrElse(s._1, 0L) + s._2))
    })
    a
  }


  /**
    * aggregate方法,底层调用的还是fold方法
    */
  def method6(map1: Map[String, Long], map2: Map[String, Long]): Map[String, Long] = {
    val a = map1.aggregate[Map[String, Long]](map2)((m: Map[String, Long], s: (String, Long)) =>
      m + (s._1 -> (m.getOrElse(s._1, 0L) + s._2)), (s1: Map[String, Long], s2: Map[String, Long]) => s1)
    a
  }
}

```


## Option

`Option`是一个包含或者不包含某些事物的容器。

Option的基本接口类似于：
```scala
trait Option[T] {
  def isDefined: Boolean
  def get: T
  def getOrElse(t: T): T
}
```


Option本身是泛型的，它有两个子类：`Some[T]`和`None`

我们来看一个Option的示例： `Map.get`使用`Option`来作为它的返回类型。Option的作用是告诉你这个方法可能不会返回你请求的值。
```scala
scala> val numbers = Map(1 -> "one", 2 -> "two")
numbers: scala.collection.immutable.Map[Int,String] = Map((1,one), (2,two))

scala> numbers.get(2)
res0: Option[java.lang.String] = Some(two)

scala> numbers.get(3)
res1: Option[java.lang.String] = None
```


现在，我们要的数据存在于这个`Option`里。那么我们该怎么处理它呢？

一个比较直观的方法就是根据`isDefined`方法的返回结果作出不同的处理。
```scala
//如果这个值存在的话，那么我们把它乘以2，否则返回0。

val result = if (res1.isDefined) {
  res1.get * 2
} else {
  0
}
```


不过，我们更加建议你使用`getOrElse`或者模式匹配来处理这个结构。

getOrElse让你可以很方便地定义一个默认值。
```scala
val result = res1.getOrElse(0) * 2
```
模式匹配可以很好地和`Option`进行配合使用。
```scala
val result = res1 match { case Some(n) => n * 2 case None => 0 }
```

# scala中集合的交集、并集、差集

scala中的集合Set，==用于存放无序非重复数据==

<font color="red">对于非Set集合（**Array/ArrayBuffer/List/ListBuffer**），在做交集、并集、差集时必须转换为Set，否则<font style="background-color: yellow;">**元素不去重没有意义**</font>。而对于非Set类型集合元素去重，也有个很好的方法：**distinct**</font>

> `scala> List(1,2,2,3).distinct 去重方法`
> 
> 这个方法的好处是集合在去重后类型不变，比用Set去重更简洁

## 1. 求交集：使用`&`或者`interset`方法求交集

```scala
scala> Set(1,2,3) & Set(2,4) // &方法等同于interset方法
res1: scala.collection.immutable.Set[Int] = Set(2)
scala> res1.foreach(println)
2
 
scala> Set(1,2,3) intersect Set(2,4)
scala> Array(1,2,3) intersect Array(3,4)     // Array求交集的方法，不能用 &
res9: Array[Int] = Array(3)
scala> List(1,2,3,2) intersect List(2,4)
res12: List[Int] = List(2)
```

## 2. 求并集：可用`++`方法和`union`求并集

**Set/Array/ArrayBuffer/List/ListBuffer均适用**

```scala
scala> Set(1,2,3) ++ Set(2,4)
scala> Set(1,2,3) | Set(2,4) // |方法等同于union方法
scala> Set(1,2,3) union Set(2,4)
 
scala> Set(1,2,3) ++ Set(3,4)
res19: scala.collection.immutable.Set[Int] = Set(1, 2, 3, 4)
scala> Array(1,2,3) ++ Array(3,4)
res20: Array[Int] = Array(1, 2, 3, 3, 4)
scala> List(1,2,3) ++ List(3,4)
res21: List[Int] = List(1, 2, 3, 3, 4)
```

## 3. 求差集：--方法和diff方法求差集

```scala
scala> Set(1,2,3) -- Set(2,4) //得到 Set(1,3)
scala> Set(1,2,3) &~ Set(2,4)
scala> Set(1,2,3) diff Set(2,4)
```

## 4. 添加或者删除元素

<font color="red">
添加或删除元素，可以直接用+,-方法来操作，添加删除多个元素可以用元组来封装 ：</font>

```scala
scala> Set(1,2,3) + (2,4)
scala> Set(1,2,3) - (2,4)
```

# 函数组合器

`List(1,2,3) map squared`会在列表的每个元素上分别应用`squared`函数，并且返回一个新的列表，可能是`List(1,4,9)`。我们把类似于`map`这样的操作称为_组合器_。（如果你需要一个更好的定义，你或许会喜欢Stackoverflow上的[关于组合器的解释](http://stackoverflow.com/questions/7533837/explanation-of-combinators-for-the-working-man)。

## map

在列表中的每个元素上计算一个函数，并且返回一个包含相同数目元素的列表。
```scala
scala> numbers.map((i: Int) => i * 2)
res0: List[Int] = List(2, 4, 6, 8)
```

或者传入一个部分计算的函数
```scala
scala> def timesTwo(i: Int): Int = i * 2
timesTwo: (i: Int)Int

scala> numbers.map(timesTwo _)
res0: List[Int] = List(2, 4, 6, 8)
```

## foreach

foreach和map相似，只不过它没有返回值，foreach只要是为了对参数进行作用。

```scala
scala> numbers.foreach((i: Int) => i * 2)
```

没有返回值。

你可以尝试把返回值放在一个变量里，不过它的类型应该是Unit（或者是void）
```scala
scala> val doubled = numbers.foreach((i: Int) => i * 2)
doubled: Unit = ()
```

## filter

移除任何使得传入的函数返回false的元素。返回Boolean类型的函数一般都称为断言函数。
```scala
scala> numbers.filter((i: Int) => i % 2 == 0)
res0: List[Int] = List(2, 4)
```
```scala
scala> def isEven(i: Int): Boolean = i % 2 == 0
isEven: (i: Int)Boolean

scala> numbers.filter(isEven _)
res2: List[Int] = List(2, 4)
```

## zip

`zip`把两个列表的元素合成一个由元素对组成的列表里。
```scala
scala> List(1, 2, 3).zip(List("a", "b", "c"))
res0: List[(Int, String)] = List((1,a), (2,b), (3,c))
```

## partition

`partition`根据断言函数的返回值对列表进行拆分。
```scala
scala> val numbers = List(1, 2, 3, 4, 5, 6, 7, 8, 9, 10)
scala> numbers.partition(_ %2 == 0)
res0: (List[Int], List[Int]) = (List(2, 4, 6, 8, 10),List(1, 3, 5, 7, 9))
```

## find

find返回集合里第一个匹配断言函数的元素
```scala
scala> numbers.find((i: Int) => i > 5)
res0: Option[Int] = Some(6)
```

## drop & dropWhile

`drop`丢弃前i个元素
```scala
scala> numbers.drop(5)
res0: List[Int] = List(6, 7, 8, 9, 10)
```


`dropWhile`移除前几个匹配断言函数的元素。例如，如果我们从numbers列表里`dropWhile`奇数的话，`1,3`会被移除（`5`则不会，因为它被`4`所“保护”）。
```scala
val numbers = List(1,3,4,5,6,7,8,9,10)
scala> numbers.dropWhile(_ % 2 != 0)
res0: List[Int] = List( 4,5,6,7,8,9,10)
```


## foldLeft

```scala
scala> numbers.foldLeft(0)((m: Int, n: Int) => m + n)
res0: Int = 55
```

0是起始值（注意numbers是一个List[Int])，m是累加值。

更加直观的来看：
```scala
scala> numbers.foldLeft(0) { (m: Int, n: Int) => println("m: " + m + " n: " + n); m + n }
m: 0 n: 1
m: 1 n: 2
m: 3 n: 3
m: 6 n: 4
m: 10 n: 5
m: 15 n: 6
m: 21 n: 7
m: 28 n: 8
m: 36 n: 9
m: 45 n: 10
res0: Int = 55
```


## foldRight

这个和foldLeft相似，只不过是方向相反。

```scala
scala> numbers.foldRight(0) { (m: Int, n: Int) => println("m: " + m + " n: " + n); m + n }
m: 10 n: 0
m: 9 n: 10
m: 8 n: 19
m: 7 n: 27
m: 6 n: 34
m: 5 n: 40
m: 4 n: 45
m: 3 n: 49
m: 2 n: 52
m: 1 n: 54
res0: Int = 55
```

## flatten

flatten可以把嵌套的结构展开。
```scala
scala> List(List(1, 2), List(3, 4)).flatten
res0: List[Int] = List(1, 2, 3, 4)
```


## flatMap

flatMap是一个常用的combinator，它结合了map和flatten的功能。flatMap接收一个可以处理嵌套列表的函数，然后把返回结果连接起来。
```scala
scala> val nestedNumbers = List(List(1, 2), List(3, 4))
nestedNumbers: List[List[Int]] = List(List(1, 2), List(3, 4))

scala> nestedNumbers.flatMap(x => x.map(_ * 2))
res0: List[Int] = List(2, 4, 6, 8)
```


可以把它当作map和flatten两者的缩写：
```scala
scala> nestedNumbers.map((x: List[Int]) => x.map(_ * 2)).flatten
res1: List[Int] = List(2, 4, 6, 8)
```


这个调用map和flatten的示例是这些函数的类“组合器”特点的展示。

_See Also_ Effective Scala has opinions about [flatMap]



# 广义的函数组合器

现在，我们学习了一大堆处理集合的函数。

不过，我们更加感兴趣的是怎么写我们自己的函数组合器。

有趣的是，上面展示的每个函数组合器都是可以通过fold来实现的。我们来看一些示例。
```scala
def ourMap(numbers: List[Int], fn: Int => Int): List[Int] = {
  numbers.foldRight(List[Int]()) { (x: Int, xs: List[Int]) =>
    fn(x) :: xs
  }
}

scala> ourMap(numbers, timesTwo(_))
res0: List[Int] = List(2, 4, 6, 8, 10, 12, 14, 16, 18, 20)
```


为什么要List[Int]?因为Scala还不能聪明到知道你需要在一个空的Int列表上来进行累加。

## 如何处理好Map?

我们上面所展示的所有函数组合器都能都Map进行处理。Map可以当作是由键值对组成的列表，这样你写的函数就可以对Map里的key和value进行处理。
```scala
scala> val extensions = Map("steve" -> 100, "bob" -> 101, "joe" -> 201)
extensions: scala.collection.immutable.Map[String,Int] = Map((steve,100), (bob,101), (joe,201))
```

现在过滤出所有分机号码小于200的元素。
```scala
scala> extensions.filter((namePhone: (String, Int)) => namePhone._2 < 200)
res0: scala.collection.immutable.Map[String,Int] = Map((steve,100), (bob,101))
```

因为你拿到的是一个元组，所以你不得不通过它们的位置来取得对应的key和value，太恶心了！

幸运的是，我们实际上可以用一个模式匹配来优雅地获取key和value。
```scala
scala> extensions.filter({case (name, extension) => extension < 200})
res0: scala.collection.immutable.Map[String,Int] = Map((steve,100), (bob,101))
```
## Scala集合和Java集合进行转换
### JavaConverters ★
你可以通过 JavaConverters 包在Java和Scala的集合类之间进行转换. 它给常用的Java集合提供了asScala方法，同时给常用的Scala集合提供了asJava方法。

```scala
import scala.collection.JavaConverters._
val sl = new scala.collection.mutable.ListBuffer[Int]
val jl : java.util.List[Int] = sl.asJava
val sl2 : scala.collection.mutable.Buffer[Int] = jl.asScala
assert(sl eq sl2)
```

```
Pimped Type                            | Conversion Method   | Returned Type
=================================================================================================
scala.collection.Iterator              | asJava              | java.util.Iterator
scala.collection.Iterator              | asJavaEnumeration   | java.util.Enumeration
scala.collection.Iterable              | asJava              | java.lang.Iterable
scala.collection.Iterable              | asJavaCollection    | java.util.Collection
scala.collection.mutable.Buffer        | asJava              | java.util.List
scala.collection.mutable.Seq           | asJava              | java.util.List
scala.collection.Seq                   | asJava              | java.util.List
scala.collection.mutable.Set           | asJava              | java.util.Set
scala.collection.Set                   | asJava              | java.util.Set
scala.collection.mutable.Map           | asJava              | java.util.Map
scala.collection.Map                   | asJava              | java.util.Map
scala.collection.mutable.Map           | asJavaDictionary    | java.util.Dictionary
scala.collection.mutable.ConcurrentMap | asJavaConcurrentMap | java.util.concurrent.ConcurrentMap
—————————————————————————————————————————————————————————————————————————————————————————————————
java.util.Iterator                     | asScala             | scala.collection.Iterator
java.util.Enumeration                  | asScala             | scala.collection.Iterator
java.lang.Iterable                     | asScala             | scala.collection.Iterable
java.util.Collection                   | asScala             | scala.collection.Iterable
java.util.List                         | asScala             | scala.collection.mutable.Buffer
java.util.Set                          | asScala             | scala.collection.mutable.Set
java.util.Map                          | asScala             | scala.collection.mutable.Map
java.util.concurrent.ConcurrentMap     | asScala             | scala.collection.mutable.ConcurrentMap
java.util.Dictionary                   | asScala             | scala.collection.mutable.Map
java.util.Properties                   | asScala             | scala.collection.mutable.Map[String, String]
```

### JavaConversions(@deprecated)

Java Map与Scala Map隐式转换

- Java Map转换Scala Map：在调用的函数前边导入隐式转换函数

```scala
import scala.collection.JavaConversions.mapAsJavaMap
```

- Scala Map转换Java Map：在调用的函数前边导入隐式转换函数

```scala
import scala.collection.JavaConversions.mapAsScalaMap
```


用户无需使用asScala或asJava，必要时，编译器将自动将java集合转换为正确的scala类型。
```scala
 import scala.collection.JavaConversions._

    val sl = new scala.collection.mutable.ListBuffer[Int]
    val jl : java.util.List[Int] = sl
    val sl2 : scala.collection.mutable.Buffer[Int] = jl
```


双向转换：

```scala
scala.collection.Iterable <=> java.lang.Iterable
scala.collection.Iterable <=> java.util.Collection
scala.collection.Iterator <=> java.util.{ Iterator, Enumeration }
scala.collection.mutable.Buffer <=> java.util.List
scala.collection.mutable.Set <=> java.util.Set
scala.collection.mutable.Map <=> java.util.{ Map, Dictionary }
scala.collection.mutable.ConcurrentMap <=> java.util.concurrent.ConcurrentMap
```


另外，下面提供了一些单向的转换方法：

```scala
scala.collection.Seq => java.util.List
scala.collection.mutable.Seq => java.util.List
scala.collection.Set => java.util.Set
scala.collection.Map => java.util.Map
```

### JavaConversion与JavaConverters区别

**JavaConversion在Scala2.13.0中被标注为弃用，请改用`scala.jdk.CollectionConverters`。**

JavaConversions提供了一系列隐式方法，可在Java集合和最接近的对应Scala集合之间进行转换，反之亦然。这是通过创建实现Scala接口并将调用转发到基础Java集合或Java接口并将调用转发到基础Scala集合的包装器来完成的。用户无需使用asScala或asJava，必要时，编译器将自动将java集合转换为正确的scala类型。

JavaConverters使用pimp-my-library模式将asScala方法“添加” 到Java集合和asJava方法到Scala集合，这将返回上面讨论的适当包装器。它（从2.8.1版开始）比JavaConversions（从2.8 版开始）更新，并使Scala和Java集合之间的转换变得明确。用户可以控制将发生隐式转换的唯一位置：在哪里写`.asScala`或`.asJava`


<small>来源： [Scala中的JavaConverters和JavaConversions之间有什么区别？](https://stackoverflow.com/questions/8301947/what-is-the-difference-between-javaconverters-and-javaconversions-in-scala)</small>

##  contains与exists在集合中的选择
**`set` 和`Map`（默认由Hash表实现）用`contains()`查找是最快的**，因为它们计算hash值然后立即跳到正确的位置。
例如，如果你想从一个1000项的list中找一个任意字符串，用`contains`在`set`上查找比`List`，`Vector`或`Array`快大约100倍。

**当使用`exists()`时**，你只要关心的是：how fast the collection is to traverse——集合遍历的速度，
因为你必须traverse everything anyway.这时候，**List 通常最好，除非你手动遍历`Array`**。但set是特别差的，
如：在`List`上使用`exists`比`Set`上快大约8倍，如果都是1000个元素的话。其他的结构花的时间大概是List的2.5x（通第是1.5x，但是`Vector`有一个基本的树形结构，快速遍历起来不是那么快？
？which is not all that fast to traverse.）