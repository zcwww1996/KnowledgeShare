[TOC]

参考：<br>
https://blog.csdn.net/weixin_42529806/article/details/88067433
# 第一章 flink简介

## 1.1 初始flink
Flink 项目的理念是:“==**Apache Flink 是为分布式、高性能、随时可用以及准确的流处理应用程序打造的开源流处理框架**==”。

## 1.2 Flink 的重要特点

### 1.2.1 事件驱动型(Event-driven)

[![Event-driven](https://ae01.alicdn.com/kf/U46a8f7b25b444f2989460f04b1d95cc4U.jpg)](https://i0.wp.com/img.vim-cn.com/4d/a6c8e4586a220cdde3ed6746ff11b3217cd122.png)

### 1.2.2 流与批的世界观
**==批处理==** 的特点是**有界、持久、大量**，非常适合需要访问全套记录才能完成的计算工作，一般用于离线统计

**==流处理==** 的特点是**无界、实时, 无需针对整个数据集执行操作**，而是对通过系统
传输的每个数据项执行操作，一般用于实时统计。

在 flink 的世界观中，一切都是由流组成的，离线数据是有界限的流，实时数
据是一个没有界限的流，这就是所谓的有界流和无界流

**无界数据流**:无界数据流有一个开始但是没有结束，它们不会在生成时终止并
提供数据，必须连续处理无界流，也就是说必须在获取后立即处理 event。

**有界数据流**:有界数据流有明确定义的开始和结束，可以在执行任何计算之前
通过获取所有数据来处理有界流，处理有界流不需要有序获取，因为可以始终对有
界数据集进行排序，有界流的处理也称为批处理。

[![bounded-unbounded.png](https://s3.ax1x.com/2021/01/26/sjJ8QP.png)](https://images.weserv.nl/?url=https://i.loli.net/2020/08/27/zNeW9UOKHSq63uA.png)

### 1.2.3 分层 api

[![Flink_API.png](https://s3.ax1x.com/2021/01/26/sjJ1zt.png)](https://images.weserv.nl/?url=https://i.loli.net/2020/08/27/rCzEn72FGsQux4B.png)

### 1.2.4 flink的其他特点

- 支持事件时间（event-time）和处理时间（processing-time）
语义
- 精确一次（exactly-once）的状态一致性保证
- 低延迟，每秒处理数百万个事件，毫秒级延迟
- 与众多常用存储系统的连接
- 高可用，动态扩展，实现7*24小时全天候运行

### 1.2.5 flink VS Spark Streaming

- 流（stream）和微批（micro-batching）<br>
[![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581436642126-eb0d62b6-a7ed-414c-a8e3-fa1e5124edce.png)](https://img.imgdb.cn/item/600fe2703ffa7d37b38a0d55.png)

- 数据模型
  - spark 采用RDD模型，spark streaming的DStream实际上也就是一组组小批数据RDD的集合
  - flink基本数据模型是数据流，以及事件（Event）序列
- 运行时架构
  - spark是批计算，将DAG划分为不同的stage，一个完成后才可以计算下一个
  - flink是标准的流执行模式，一个事件在一个节点处理完后可以直接发往下一个节点进行处理