[TOC]

链接：https://www.cnblogs.com/yinzhengjie/p/9884552.html


# 1. 查看kafka集群的broker的堆内存使用情况
## 1.1 使用jstat查看gc的信息


```bash
[root@kafka116 ~]# jstat -gc 12698 1s 30


参数说明：S0C：第一个幸存区的大小
　　S1C：第二个幸存区的大小
　　S0U：第一个幸存区的使用大小
　　S1U：第二个幸存区的使用大小
　　EC：伊甸园区的大小
　　EU：伊甸园区的使用大小
　　OC：老年代大小
　　OU：老年代使用大小
　　MC：方法区大小
　　MU：方法区使用大小
　　CCSC:压缩类空间大小
　　CCSU:压缩类空间使用大小
　　YGC：年轻代垃圾回收次数
　　YGCT：年轻代垃圾回收消耗时间
　　FGC：老年代垃圾回收次数
　　FGCT：老年代垃圾回收消耗时间
　　GCT：垃圾回收消耗总时间
```

![](https://i0.wp.com/i.loli.net/2020/11/23/9trS2b6ZKnDBNCE.png)
## 1.2 使用jmap查看kafka当前的堆内存信息

```bash
[root@kafka116 bin]# jmap -heap 12698
```


![](https://i0.wp.com/i.loli.net/2020/11/23/JUZSbd5zu7yYK2V.png)

　　经过上面两个图的分析，我们要观察伊甸区，幸存区以及年老代总体的使用量，发现他们的使用率都是80%以上呢！而且gc的评论是74万多次，过多的gc会将服务器的性能降低。因此考虑调大Kafka集群的堆内存（heap）是刻不容缓的事情。好，接下来我们如何去调试呢？以及将对内存调大应该注意那些事项呢？
1) 第一：kafka集群不要集体修改，要一台一台的去调整，由于我有5台broker，它允许我挂掉2台broker
2) 第二：修改kafka-server-start.sh启动脚本，建议先改成15G（我的kafka集群的配置相对较低，32G内存，32core，80T硬盘），如果还是不够的话可以考虑继续加大heap内存的配置；
# 2. 对kafka进行调优案例实操
## 2.1 查看默认的配置

![](https://i0.wp.com/i.loli.net/2020/11/23/YZHRe92GfQmuzth.png)

## 2.2 修改kafka启动脚本的配置文件

![](https://i0.wp.com/i.loli.net/2020/11/23/xHJ3eronzWXhB9Y.png)
## 2.3 重启当前broker的Kafka服务


```bash
[root@kafka116 bin]# kafka-server-stop.sh  #停止当前的kafka进程
[root@kafka116 bin]# 
[root@kafka116 bin]# kafka-server-start.sh -daemon /soft/kafka/config/server.properties #启动当前的kafka进程
[root@kafka116 bin]# 
[root@kafka116 bin]# 
[root@kafka116 bin]# jps #查看kafka进程是否启动
5460 Kafka
4246 ProdServerStart
23014 Jps
```


## 2.4 查看调优后的内存

![](https://i0.wp.com/i.loli.net/2020/11/23/qScruEyh7ogvIOU.png)

## 2.5 查看调整后的JVM使用情况

![](https://i0.wp.com/i.loli.net/2020/11/23/lVG3Ln8Orqew1gK.png)

