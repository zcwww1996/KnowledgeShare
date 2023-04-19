[TOC]

Spark 2.0.2,  使用Kyro序列化加速，和手动注册类名

Kyro通常比原生的Java默认实现快10倍，所以建议使用Kyro来加速。

# 如何开启Kyro

开启的方法很简单，就是设参数spark.serializer。有三种方式：

*   程序内：

```scala
val conf = new SparkConf()
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
 
val spark = SparkSession
  .builder()
  .config(conf)
  .getOrCreate()
```

*   spark-submit参数：  
    `spark-submit --conf spark.serializer=org.apache.spark.serializer.KryoSerializer`
*   全局默认配置：conf/spark-defaults.conf:  
    `spark.serializer org.apache.spark.serializer.KryoSerializer`

# 为什么要注册类名


Kyro默认序列化实例时在前面会写上类名，比如java.lang.Double，类名越长，额外的存储开销越大。为了解决这个问题，Kyro允许将类名注册进映射表里，通过分配数字ID来替换冗长的类名，比如java.lang.Double使用数字0来代替。这种方式节省了储存空间，但代价是我们必须手动将所有性能相关的类名注册。

spark使用Twitter chill注册了常用的Scala类，也对自己的常用类都进行了注册，具体见KryoSerializer.scala。但很遗憾，在实际使用中，仍然有大量的类名未包含其中，必须手动注册。

# 怎么注册类名


1. 找到没有注册的类名

```scala
conf.set("spark.kryo.registrationRequired", "true")
```

开启`spark.kryo.registrationRequired=true`，于是Kyro遇到没有注册的类名时就会抛异常告警。于是，一遍遍反复排查直到完全跑通，纯体力活。

2. 如何注册私有类？  

程序内部时，spark指南中使用classOf方法来找类名，

```scala
val conf = new SparkConf().setMaster(...).setAppName(...)
conf.registerKryoClasses(Array(classOf[MyClass1], classOf[MyClass2]))
```

它的问题是，你没法导入其他包的私有类。解决方法是使用`Class.forName`，如:

```scala
conf.registerKryoClasses(Array(Class.forName("org.apache.spark.SomePrivateClass")))
```

因为KyroSerializer也是使用Class.forName来解析spark.kryo.classesToRegister字段，所以直接指定类名即可。

3.  如何注册Java原生类型和数组？  
    使用Class.forName来获取，具体的书写规则见Class.getName()。

举点例子：

| Element Type | Encoding |
| --- | --- |
| double\[\] | "\[B" |
| double\[\]\[\] | "\[\[B" |
| Object() | "java.lang.Object" |
| new Object\[3\] | "\[Ljava.lang.Object;" |

2.  范例  
    最后，贴份对spark-ml中逻辑回归和XGBoost的手动注册类名，己经囊括了所有用法，

```bash
spark.kryo.classesToRegister      org.apache.spark.mllib.stat.MultivariateOnlineSummarizer,[D,[I,[F,org.apache.spark.ml.classification.MultiClassSummarizer,org.apache.spark.ml.classification.LogisticAggregator,ml.dmlc.xgboost4j.scala.Booster,ml.dmlc.xgboost4j.java.Booster,[Lml.dmlc.xgboost4j.scala.Booster;,org.apache.spark.ml.feature.LabeledPoint,org.apache.spark.ml.linalg.SparseVector,org.apache.spark.mllib.evaluation.binary.BinaryLabelCounter,scala.reflect.ClassTag$$anon$1,java.lang.Class,[Lorg.apache.spark.mllib.evaluation.binary.BinaryLabelCounter;,scala.collection.mutable.WrappedArray$ofRef,[Ljava.lang.String;,[Lorg.apache.spark.sql.execution.datasources.HadoopFsRelation$FakeFileStatus;,org.apache.spark.sql.execution.datasources.HadoopFsRelation$FakeFileStatus,[Lorg.apache.spark.sql.execution.datasources.HadoopFsRelation$FakeBlockLocation;,org.apache.spark.sql.execution.datasources.HadoopFsRelation$FakeBlockLocation,org.apache.spark.sql.execution.columnar.CachedBatch,org.apache.spark.ml.feature.Instance,[[B,org.apache.spark.sql.catalyst.expressions.GenericInternalRow,[Ljava.lang.Object;
```