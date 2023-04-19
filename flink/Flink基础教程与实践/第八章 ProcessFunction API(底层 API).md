[TOC]

我们之前学习的转换算子是无法访问事件的时间戳信息和水位线信息的。而这在一些应用场景下，极为重要。例如 MapFunction 这样的 map 转换算子就无法访问时间戳或者当前事件的事件时间。

基于此，DataStream API 提供了一系列的 Low-Level 转换算子。可以访问时间戳、**watermark**以及注册定时事件。还可以输出特定的一些事件，例如超时事件等。

Process Function 用来构建事件驱动的应用以及实现自定义的业务逻辑(使用之前的window 函数和转换算子无法实现)。例如，Flink SQL 就是使用 Process Function 实现的。


Flink 提供了8个 Process Function:

- ProcessFunction
- KeyedProcessFunction
- CoProcessFunction
- ProcessJoinFunction
- BroadcastProcessFunction
- KeyedBroadcastProcessFunction
- ProcessWindowFunction 
- ProcessAllWindowFunction

# 1. KeyedProcessFunction

这里我们重点介绍 KeyedProcessFunction。

KeyedProcessFunction 用来操作 KeyedStream。KeyedProcessFunction 会处理流
的每一个元素，输出为 0 个、1 个或者多个元素。所有的 Process Function 都继承自
RichFunction 接口，所以都有 open()、close()和 getRuntimeContext()等方法。而
KeyedProcessFunction[KEY, IN, OUT]还额外提供了两个方法:

- **`processElement(v: IN, ctx: Context, out: Collector[OUT])`** , 流中的每一个元素都会调用这个方法，调用结果将会放在 Collector 数据类型中输出。**Context**可以访问元素的时间戳，元素的 key，以及 **TimerService**时间服务。**Context**还可以将结果输出到别的流(side outputs)。

- **`onTimer(timestamp: Long, ctx: OnTimerContext, out: Collector[OUT])`** 是一个回调函数。当之前注册的定时器触发时调用。参数 timestamp 为定时器所设定的触发的时间戳。Collector 为输出结果的集合。OnTimerContext 和processElement 的 Context 参数一样，提供了上下文的一些信息，例如定时器触发的时间信息(事件时间或者处理时间)。


# 2. TimerService 和 定时器(Timers)
Context 和 OnTimerContext 所持有的 TimerService 对象拥有以下方法:

-  currentProcessingTime(): Long 返回当前处理时间
-  currentWatermark(): Long 返回当前 watermark 的时间戳
-  registerProcessingTimeTimer(timestamp: Long): Unit 会注册当前 key 的processing time 的定时器。当 processing time 到达定时时间时，触发 timer。
-  registerEventTimeTimer(timestamp: Long): Unit 会注册当前 key 的 event time定时器。当水位线大于等于定时器注册的时间时，触发定时器执行回调函数。
-  deleteProcessingTimeTimer(timestamp: Long): Unit 删除之前注册处理时间定时器。如果没有这个时间戳的定时器，则不执行。
- deleteEventTimeTimer(timestamp: Long): Unit 删除之前注册的事件时间定时器，如果没有此时间戳的定时器，则不执行。

当定时器 timer 触发时，会执行回调函数 onTimer()。注意定时器 timer 只能在keyed streams 上面使用。

下面举个例子说明 KeyedProcessFunction 如何操作 KeyedStream。

需求:监控温度传感器的温度值，如果温度值在一秒钟之内(processing time)连续上升，则报警。


```scala
val warnings = readings 
.keyBy(_.id)
.process(new TempIncreaseAlertFunction)
```

看一下TempIncreaseAlertFunction如何实现, 程序中使用了ValueState这样一个状态变量。


```scala
class TempIncreaseAlertFunction extends KeyedProcessFunction[String, SensorReading, String] { // 保存上一个传感器温度值
lazy val lastTemp: ValueState[Double] = getRuntimeContext.getState(
new ValueStateDescriptor[Double]("lastTemp", Types.of[Double]) )
    
// 保存注册的定时器的时间戳
lazy val currentTimer: ValueState[Long] = getRuntimeContext.getState(
new ValueStateDescriptor[Long]("timer", Types.of[Long])
)
override def processElement(r: SensorReading,
ctx: KeyedProcessFunction[String, SensorReading, String]#Context,
out: Collector[String]): Unit = {
// 取出上一次的温度
val prevTemp = lastTemp.value()
    // 将当前温度更新到上一次的温度这个变量中 lastTemp.update(r.temperature)
val curTimerTimestamp = currentTimer.value()
if (prevTemp == 0.0 || r.temperature < prevTemp) {
// 温度下降或者是第一个温度值，删除定时器 ctx.timerService().deleteProcessingTimeTimer(curTimerTimestamp)
// 清空状态变量
currentTimer.clear()
} else if (r.temperature > prevTemp && curTimerTimestamp == 0) {
// 温度上升且我们并没有设置定时器
val timerTs = ctx.timerService().currentProcessingTime() + 1000 ctx.timerService().registerProcessingTimeTimer(timerTs)
currentTimer.update(timerTs) }
}
override def onTimer(ts: Long,
ctx: KeyedProcessFunction[String, SensorReading, String]#OnTimerContext,
out: Collector[String]): Unit = {
out.collect("传感器 id 为: " + ctx.getCurrentKey + "的传感器温度值已经连续 1s 上升了。")
currentTimer.clear() }
}
```


# 3. 侧输出流(SideOutput)
大部分的 DataStream API 的算子的输出是单一输出，也就是某种数据类型的流。

除了 split 算子，可以将一条流分成多条流，这些流的数据类型也都相同。process function 的 side outputs 功能可以产生多条流，并且这些流的数据类型可以不一样。

一个 side output 可以定义为 `OutputTag[X]`对象，X 是输出流的数据类型。process
function 可以通过 Context 对象发射一个事件到一个或者多个 side outputs。

下面是一个示例程序:


```scala
val monitoredReadings: DataStream[SensorReading] = readings .process(new FreezingMonitor)
monitoredReadings
.getSideOutput(new OutputTag[String]("freezing-alarms")) .print()
readings.print()
```
接下来我们实现 FreezingMonitor 函数，用来监控传感器温度值，将温度值低于`32F`的温度输出到 side output。


```scala
class FreezingMonitor extends ProcessFunction[SensorReading, SensorReading] { // 定义一个侧输出标签
lazy val freezingAlarmOutput: OutputTag[String] =
new OutputTag[String]("freezing-alarms")
override def processElement(r: SensorReading,
ctx: ProcessFunction[SensorReading, SensorReading]#Context,
out: Collector[SensorReading]): Unit = {
// 温度在 32F 以下时，输出警告信息 if (r.temperature < 32.0) {
ctx.output(freezingAlarmOutput, s"Freezing Alarm for ${r.id}") }
// 所有数据直接常规输出到主流
out.collect(r) }
}
```


# 4. CoProcessFunction

对于两条输入流，DataStream API 提供了 CoProcessFunction 这样的 low-level操作. CoProcessFunction 提供了操作每一个输入流的方法: processElement1()和processElement2()。

类似于 ProcessFunction，这两种方法都通过 Context 对象来调用。这个 Context对象可以访问事件数据，定时器时间戳，TimerService，以及 side outputs。

CoProcessFunction 也提供了 onTimer()回调函数。