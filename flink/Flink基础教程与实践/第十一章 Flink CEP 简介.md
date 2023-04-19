[TOC]

# 11.1 什么是复杂事件处理 CEP
一个或多个由简单事件构成的事件流通过一定的规则匹配，然后输出用户想得
到的数据，满足规则的复杂事件。

特征:

- 目标:从有序的简单事件流中发现一些高阶特征

- 输入:一个或多个由简单事件构成的事件流

- 处理:识别简单事件之间的内在联系，多个符合一定规则的简单事件构成复杂事件

- 输出:满足规则的复杂事件

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1582630497464-22c48d8b-1161-4ae4-ac8e-b42015747cc5.png)

CEP 用于分析低延迟、频繁产生的不同来源的事件流。CEP 可以帮助在复杂的、不相关的事件流中找出有意义的模式和复杂的关系，以接近实时或准实时的获得通知并阻止一些行为。

CEP 支持在流上进行模式匹配，根据模式的条件不同，分为连续的条件或不连续的条件;模式的条件允许有时间的限制，当在条件范围内没有达到满足的条件时，
会导致模式匹配超时。


看起来很简单，但是它有很多不同的功能:

- 输入的流数据，尽快产生结果
- 在 2 个 event 流上，基于时间进行聚合类的计算
- 提供实时/准实时的警告和通知
- 在多样的数据源中产生关联并分析模式
- 高吞吐、低延迟的处理


市场上有多种 CEP 的解决方案，例如 Spark、Samza、Beam 等，但他们都没有

提供专门的 library 支持。但是 Flink 提供了专门的 CEP library。



# 2. Flink CEP


Flink 为 CEP 提供了专门的 Flink CEP library，它包含如下组件:

- Event Stream

  - pattern 定义
  - pattern 检测
  - 生成 Alert

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1582630622426-3cabab15-a389-4a29-9f75-1935f478c414.png)



首先，开发人员要在 DataStream 流上定义出模式条件，之后 Flink CEP 引擎进行模式检测，必要时生成告警。

为了使用 Flink CEP，我们需要导入依赖:


```xml
<dependency>
    <groupId>org.apache.flink</groupId> 
    <artifactId>flink-cep_${scala.binary.version}</artifactId> 
    <version>${flink.version}</version>
</dependency>
```

## 2.1 Event Streams

以登陆事件流为例:

```scala
case class LoginEvent(userId: String, ip: String, eventType: 
                      String, eventTi me: String)
val env = StreamExecutionEnvironment.getExecutionEnvironment 
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime) 
env.setParallelism(1)

   val loginEventStream = env.fromCollection(List( 
       LoginEvent("1", "192.168.0.1", "fail", "1558430842"), 
       LoginEvent("1", "192.168.0.2", "fail", "1558430843"), 
       LoginEvent("1", "192.168.0.3", "fail", "1558430844"), 
       LoginEvent("2", "192.168.10.10", "success", "1558430845")
)).assignAscendingTimestamps(_.eventTime.toLong)
```

## 2.2 Pattern API
每个 Pattern 都应该包含几个步骤，或者叫做 state。从一个 state 到另一个 state，通常我们需要定义一些条件，例如下列的代码:


```scala
val loginFailPattern = Pattern.begin[LoginEvent]("begin") 
    .where(_.eventType.equals("fail"))
	.next("next")
	.where(_.eventType.equals("fail")) 
    .within(Time.seconds(10)
```

每个 state 都应该有一个标示:例如.begin[LoginEvent]("begin")中的"begin"

每个 state 都需要有一个唯一的名字，而且需要一个 filter 来过滤条件，这个过滤条件定义事件需要符合的条件，例如:
`.where(_.eventType.equals("fail"))`

我们也可以通过 subtype 来限制 event 的子类型:
`start.subtype(SubEvent.class).where(...);`

事实上，你可以多次调用 subtype 和 where 方法;而且如果 where 条件是不相关
的，你可以通过 or 来指定一个单独的 filter 函数:
`pattern.where(...).or(...);`


之后，我们可以在此条件基础上，通过 next 或者 followedBy 方法切换到下一个state，next 的意思是说上一步符合条件的元素之后紧挨着的元素;而 followedBy 并不要求一定是挨着的元素。这两者分别称为严格近邻和非严格近邻。


```scala
val strictNext = start.next("middle")
val nonStrictNext = start.followedBy("middle")
```
最后，我们可以将所有的 Pattern 的条件限定在一定的时间范围内:


```scala
next.within(Time.seconds(10))
```

这个时间可以是 Processing Time，也可以是 Event Time。


## 2.3 Pattern检测

通过一个 input DataStream 以及刚刚我们定义的 Pattern，我们可以创建一个PatternStream:


```scala
val input = ... 
val pattern = ...
val patternStream = CEP.pattern(input, pattern)
val patternStream = CEP.pattern(loginEventStream.keyBy(_.userId), loginFail Pattern)
```

一旦获得 PatternStream，我们就可以通过 select 或 flatSelect，从一个 Map 序列找到我们需要的警告信息。


## 2.4 select

select 方法需要实现一个 PatternSelectFunction，通过 select 方法来输出需要的
警告。它接受一个 Map 对，包含 string/event，其中 key 为 state 的名字，event 则为
真实的 Event。

```scala
val loginFailDataStream = patternStream
.select((pattern: Map[String, Iterable[LoginEvent]]) => {
val first = pattern.getOrElse("begin", null).iterator.next() val second = pattern.getOrElse("next", null).iterator.next()
Warning(first.userId, first.eventTime, second.eventTime, "warning") })
```

其返回值仅为 1 条记录


## 2.5 flatSelect

通过实现 PatternFlatSelectFunction，实现与 select 相似的功能。唯一的区别就是 flatSelect 方法可以返回多条记录，它通过一个 Collector[OUT]类型的参数来将要输出的数据传递到下游。


## 2.6 超时事件的处理

通过 within 方法，我们的 parttern 规则将匹配的事件限定在一定的窗口范围内。
当有超过窗口时间之后到达的 event，我们可以通过在 select 或 flatSelect 中，实现
PatternTimeoutFunction 和 PatternFlatTimeoutFunction 来处理这种情况


```scala
val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)
val outputTag = OutputTag[String]("side-output")
val result: SingleOutputStreamOperator[ComplexEvent] = patternStream.select
(outputTag){
(pattern: Map[String, Iterable[Event]], timestamp: Long) => TimeoutEvent
() }{
pattern: Map[String, Iterable[Event]] => ComplexEvent() }
val timeoutResult: DataStream<TimeoutEvent> = result.getSideOutput(outputTa g)
```