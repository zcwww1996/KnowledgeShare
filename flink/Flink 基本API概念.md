[TOC]

# 1.DataSet与DataStream
> In the case of `DataSet` the data is finite while for a `DataStream` the number of elements can be unbounded

Flink 可以处理有界数据(bounded)与无界数据(unbounded)，对应的具体数据类为DataSet和DataStream，DataSet用来进行批数据处理，DataStream则进行流式数据处理。

# 2.Flink程序结构

[![Flink Program.png](https://gd-hbimg.huaban.com/be67ee11c9f1db2df255733228b57c616926e2db13c05-LApAUV)](https://images.weserv.nl/?url=https://www.z4a.net/images/2020/08/26/Flink-Program.png)


## 2.1 设置运行环境
设置运行环境是Flink程序的基础，Flink提供了三种方式在程序的开始获取运行环境：
```java
//一般情况下会通过这种方式获取运行环境（本地/集群）
getExecutionEnvironment()
//创建本地运行环境
createLocalEnvironment()
//指定IP与端口及程序所在jar包和依赖
createRemoteEnvironment(String host, int port, String... jarFiles)
```
## 2.2 配置数据源获取数据
对于具体的数据源，有多种方法读取指定的数据：
> you can just read them line by line, as CSV files, or using completely custom data input formats. 

例如读取文本文件：
```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
DataStream<String> text = env.readTextFile("file:///path/to/file");
```
这样文本数据就从本地的文件转换成DataStream类型的数据集（DataSet类型操作类似），接下来可以对数据集进行我们需要的各种转换。
## 2.3 一系列操作转换数据🌟🌟🌟
对与DataStream类型的数据，Flink可以进行的操作有：

1. 基于单条记录：filter、map
1. 基于窗口：window
1. 合并多条流：union、join、connect
1. 拆分单条流：split

DataSet类似
## 2.4 配置数据输出
```java
//输出在文件中
writeAsText(String path)
//控制台输出
print()
```
## 2.5 提交执行
`execute()` 方法提交执行，结果返回一个JobExecutionResult，result中包含一个执行时间和计算结果
# 3.懒执行（Lazy Evaluation）
当Flink程序主方法执行的时候,数据加载和数据转换并没有立即执行，当execute()触发后执行

# 4.指定key (DataStream)
注意⚠️：Flink的数据模型并不是基于key-value键值对，主要是为了根据key 进行分区
> Keys are “virtual”: they are defined as functions over the actual data to guide the grouping operator.

## 4.1 根据字段位置进行指定
```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
//指定元组中第一个字段为key
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0)
//指定元组中第一、第二个字段为key
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1)
```
如果DataStream的类型是一个嵌套的元组（如下），那么使用 key（0）方法则指定的是整个元组Tuple2
```java
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
```
## 4.2 根据字段名称指定
```java
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
```
## 4.3 根据key选择器指定
> A key selector function takes a single element as input and returns the key for the element. The key can be of any type and be derived from deterministic computations.

```java
// some ordinary POJO
public class WC {
  public String word;                
  public int count;
}
DataStream<WC> words = // [...]
KeyedStream<WC> keyed = words
  .keyBy(new KeySelector<WC, String>() {
  	//定义KeySelector,重写getkey方法
     public String getKey(WC wc) { return wc.word; }
   });
```
# 5.指定转换函数
很多转换都需要用户自己定义

- 实现接口
```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
```

- 匿名类
```java
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

- Lambdas表达式
```java
data.filter(s -> s.startsWith("http://"));
data.reduce((i1,i2) -> i1 + i2);
```

- Rich functions❓
```
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
```
# 6.Flink支持的数据类型
支持的类型可以分为六种：

## 6.1 Java Tuple

Java API 提供了从Tuple1到Tuple25的类
>  Fields of a tuple can be accessed directly using the field’s name as `tuple.f4`, or using the generic getter method `tuple.getField(int position)`.

```java
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));
wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});
wordCounts.keyBy(0); // also valid .keyBy("f0")
```

## 6.2 Java POJOs

java的类在满足以下条件的时候会被Flink视为特殊的POJO数据类型：

   - 这个类必须是公有的
   - 它必须有一个公有的无参构造函数
   - 所有参数都需要是公有的或提供get/set方法
   - 参数的数据类型必须是Flink支持的

```java
public class WordWithCount {
    public String word;
    public int count;
    public WordWithCount() {}
    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}
DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));
wordCounts.keyBy("word"); // key by field expression "word"
```

## 6.3 Primitive Types

Flink支持Java所有的基本数据类型，例如Integer、String、Double

## 6.4 Regular Classes

Flink 支持大部分的Java类

## 6.5 values
## 6.6 Hadoop Writables
## 6.7 Special Types
