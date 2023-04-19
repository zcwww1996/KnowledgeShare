[TOC]
注解在Spark的源码中周围都能看见，

例如：

**@volatile**:这是一个多线程并发控制轻量级框架所使用的一个注解，在多线程并发操作一个全局变量的时候，线程首先得copy一个副本到本地变量中，处理完成之后，然后再对全局变量进行update

不同线程去update的时候，会有很多问题：例如版本问题，业务逻辑的不一致性

当加了这个注解之后，就是说每次读取这个变量的时候，都会自动地到各机器刷新一下最新的数据，也就是正常情况下，可以获取当前情况下最新的值

**@deprecated**：是说在当前版本中可用，但是在以后的巨大的版本变化中，可能会被丢弃
 
**@DeveloperApi**:就是框架开发者使用的API

**@Experimental**:说明是实验性的功能

**@transient**:当序列化的时候，我们需要序列化所有的成员进磁盘，反序列化的时候也需要反序列化，当有这个关键字的时候，不参与序列化和反序列化
```scala
@transient var name="leo"
//瞬间字段，不会虚拟化这个字段
```

**@varargs**:参数可变

```scala
@varargs def test(args:String*){}
//标注方法中的参数是变长参数
```

关于提取器：在进行模式匹配的时候，是需要提取器的，case class 和 case object就天生实现了提取器，一般看见有case的地方，就有提取器

这可以极大地简化了机器/线程之间的通信

例子：

```scala
@Coder("Scala") case class Person1(name:String,age:Int)

class DTCoder(val name:String,val sal:Int)
object DTCoder {
  def apply(name:String,sal:Int) = new DTCoder(name,sal)
  def unapply(information:String) = {
    Some(information.substring(0,information.indexOf(" ")),information.substring(information.indexOf(" ")+1))
  }
}

class Coder(val name:String) extends annotation.Annotation


object HelloExtractor {
  def main(args:Array[String]){
    val person1=Person1("Spark",6) 
    //这里就是调用了apply工厂构造方法构造出类的实例对象
    val Person1(name,age)=person1
    //调用upapply方法把person1实例中的name和age提取出来，并赋值给Person1的类的成员
    println(name+":"+age)
    
    person1 match {
      case Person1(name,age) => println(name+":"+age)
    }
    
    //只需要在伴生对象中定义好unapply方法，就能实现提取器
    //这个例子可以理解为，可以把一个对象封装成一个String,然后传给另外一台机器，然后通过String，提取出对象,这样可以很好地节省网络带宽，磁盘，内存等等
    val DTCoder(dtName,sal)="Spark 1000"
  }
}
```

归纳总结：1.Spark源码中关键注解的说明:@volatile,@deprecated,@DeveloperApi,@Experimental,@transient，@varargs


```scala
@BeanProperty
//标记生成java风格的getter／setter方法
```

```scala
@BooleanBeanProperty
//标记生生成is风格的getter方法
```

```scala
@deprecated（message=""）
//标记过时的方法，让编译器发出警告
```

```scala
@unchecked
//让编译器发出类型转换的警告
```


