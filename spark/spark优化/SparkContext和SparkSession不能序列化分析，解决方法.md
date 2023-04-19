[TOC]

来源：https://blog.csdn.net/cyz52/article/details/103899969

# 报错信息

```java
Caused by: java.io.NotSerializableException: org.apache.spark.SparkContext
Serialization stack:
	- object not serializable (class: org.apache.spark.SparkContext, value: org.apache.spark.SparkContext@7364f68)
	- field (class: com.bqs.bigdata.analyse.spark.load.LoadGraphxData, name: sc, type: class org.apache.spark.SparkContext)
	- object (class com.bqs.bigdata.analyse.spark.load.LoadGraphxData, com.bqs.bigdata.analyse.spark.load.LoadGraphxData@454bcbbf)
	- field (class: com.bqs.bigdata.analyse.spark.rule.RuleC, name: loadGraphxData, type: class com.bqs.bigdata.analyse.spark.load.LoadGraphxData)
	- object (class com.bqs.bigdata.analyse.spark.rule.RuleC, com.bqs.bigdata.analyse.spark.rule.RuleC@111cba40)

```

# 错误原因

前置知识：SparkContext和SparkSession 都是不可被序列化的  
原因：  
一般序列化的作用是为了传输，而SparkContext和SparkSession只在driver端运行，避免被传输到executor中执行导致出现bug，所以被设计为不可序列化。  
也就是知乎大佬所说 **防止用户写bug**

# 解决方法

```scala
class A (loadGraphxData: LoadGraphxData) extends Serializable {}
class LoadGraphxData(sc: SparkContext) extends Serializable {}

```

假设有类A和类LoadGraphxData，两个类都是可以被序列化的，  
且A参数中有LoadGraphxData类，LoadGraphxData类参数中有SparkContext。  
这两个类任意进行序列化，都会产生Task not serializable异常

## 解决方法1：

LoadGraphxData类 中的去掉参数SparkContext，这是最明显的方法，采用别的方法去调用引入。

## 解决方法2：

**使用@transient注解**  
可以在A类或者LoadGraphxData类 参数加上这个注解即可

```scala
class A (@transient loadGraphxData: LoadGraphxData) extends Serializable {}
class LoadGraphxData(@transient sc: SparkContext) extends Serializable {}

```

# 需要注意

避免executor执行的方法和对象中 含有SparkContext或者SparkSession 。  
因为关键字transient，序列化对象的时候，这个属性就不会序列化到指定的目的地中。