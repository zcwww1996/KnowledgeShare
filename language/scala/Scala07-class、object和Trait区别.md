[TOC]
class，Object，Trait有什么区别嘛？

来源： https://blog.csdn.net/fjse51/article/details/52152362

参考：[scala中 object 和 class的区别](https://blog.csdn.net/wangxiaotongfan/article/details/48242029)

# 1. class

在scala中，类名可以和对象名为同一个名字，该对象称为<font color="red">该类的伴生对象</font>，类和伴生对象可以<font color="red">相互访问他们的私有属性</font>，但是他们必须在<font color="red">同一个源文件内</font>。类只会被编译，<font color="red">不能直接被执行</font>，类的申明和主构造器在一起被申明，在一个类中，主构造器只有一个所有必须在内部申明主构造器或者是其他申明主构造器的辅构造器，主构造器会执行类定义中的所有语句。scala对每个字段都会提供getter和setter方法，同时也可以显示的申明，但是针对val类型，只提供getter方法，默认情况下，字段为公有类型，可以在setter方法中增加限制条件来限定变量的变化范围，在scala中方法可以访问改类所有对象的私有字段
# 2. object

在scala中<font color="red">没有静态方法和静态字段</font>，所以在scala中可以用object来实现这些功能，直接用对象名调用的方法都是采用这种实现方式，例如Array.toString。对象的构造器在第一次使用的时候会被调用，如果一个对象从未被使用，那么他的构造器也不会被执行；Scala中的变量有两种var和val（val类似于Java中final，值不可改变）。对象本质上拥有类（scala中）的所有特性，除此之外，object还可以一扩展类以及一个或者多个特质：例如，
abstract class ClassName（val parameter）{}
object Test extends ClassName(val parameter){}

注意：object不能提供构造器参数，也就是说<font color="red">object必须是无参的</font>


```scala
package com.donews.objectBean
/**
  * Created by yuhui on 2016/6/15.
  *
  * 注意要点：
  * 1、scala没有静态方法或者静态字段
  * 2、伴生对象充当于静态方法的类，所以伴生对象中全是静态的
  * 3、var 是可变参数 ， val是不可变对象
  */

//伴生类
class Person() {

  {
    println("我是伴生类中代码块")
  }

  def toadd(x :Int , y :Int): Unit ={
    Person.add(x , y)
  }

}

//伴生对象
object Person{

  {
    println("我是伴生对象中静态代码块")
  }

  def add(x :Int , y :Int): Unit ={
    println(x + y )
  }

  def main (args: Array[String] ) {
    var x = 3 ;
    var y = 5 ;
    val p = new Person()
    p.toadd(2,3)
    add(x , y)
  }
}
```

# 3. trait

在java中可以通过interface<font color="red">实现多重继承</font>，在Scala中可以通过特征（trait）实现多重继承，不过与java不同的是，它可以定义自己的属性和实现方法体，在没有自己的实现方法体时可以认为它时java interface是等价的，在Scala中也是一般只能继承一个父类，可以<font color="red">通过多个with进行多重继承</font>。

```scala
trait TraitA{}
trait TraitB{}
trait TraitC{}

object Test1 extends TraitA with TraitB with TraitC{}
```
