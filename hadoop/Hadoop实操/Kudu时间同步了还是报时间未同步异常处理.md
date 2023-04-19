[TOC]

参考：https://mp.weixin.qq.com/s/lYXN0Wwe5tCYsN_NNeGE5w

# 1. 文档编写目的

Kudu对时间同步有严格的要求，本文档描述了一次集群已经使用NTP进行时间同步，Kudu组件还是报时间未同步问题处理流程。

**测试环境**

1.CDH和CM版本：CDP7.1.4和CM7.1.4

2.集群启用Kerbeos+OpenLDAP+Ranger

# 2. 问题描述

1) 如下集群所有Kudu实例异常

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYsxkicjo6fvy0qibUxEh0IahXKEYluEMsiazF3golJLWbbRZgIVT3breWw/640?wx_fmt=png)

2) 查看日志报时间未同步相关异常

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYOAGBaf9QibkJ4UwCmt0UMT74NNtgpCew4lMib3DWcr8PIdXKdvXB1bZA/640?wx_fmt=png)

3) 查看我们已经使用NTP进行时间正常同步，而且集群其他服务都没有问题，就Kudu组件有问题

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYaJ3mhfdykGcbNuCRwXm6E6OcNXM0MtrYURwsiavxiaibqZXLKL8Mjzx9Q/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYGo7LC0Vlgj8PQicL1tZI1rkfHbNJ8BRC4lUHadVnx3fdX6EJFuqzyMQ/640?wx_fmt=png)

# 3. 问题分析

1) 日志里面Kudu有报could not find executable :chrony异常，按照如下KB介绍【1】，排查我们并没有使用chronyc。

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYNScTvjdgOVI0eYUd3beJYDqTuMz3QHhkcaicpwzERQx3bhJCEFho4iag/640?wx_fmt=png)

【1】

```
https://my.cloudera.com/knowledge/Kudu-service-shows-error-quot-Cannot-initialize-clock-Error?id=74857
```

2) 于是尝试按照Kudu官网的介绍【2】  

【2】

```
https://kudu.apache.org/releases/1.13.0/docs/troubleshooting.html#_monitoring_clock_synchronization_status_with_the_ntp_suite
```

执行如下命令，收集节点NTP时间同步相关信息，发现NTP同步信息一切正常。

```bash
ntptime
ntpq -nc lpeers
ntpq -nc opeers
```

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYv0xiaV2GiaKNEKGQRoeicXiazslpgmPCr3XjibElxc0Ove6WtFgcpN9jSiag/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYLMTQOkcNEz76iar6dOhoC4phhTdiaHjlrJOMdnewSiaR4RItG3tEMBt6A/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYYkE6lzaVVWJgiaq7rqulyUocmStJsMiaLrKENBc90psNHko0wcv14qibg/640?wx_fmt=png)

3) 再检查ntpd进程是否用“-x“这个选项启动，如果是的话， 请移除这个选项，重新启动ntpd

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYb0iaQnNfytSRVN9J7SiaSg3TE2ZYxaTPIXvYqXZSFzQ5dzcUaLI7lNRw/640?wx_fmt=png)

# 4. 问题解决

1) 修改Kudu节点的/etc/sysconfig/ntpd文件，把-x参数删除，重启ntpd、 cloudera-scm-agent、Kudu实例，问题解决。

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYA5hUp5EastoJkwl6RiaErZCsztL5GkOM4AY8yTxmC9cIHBHNsSyXaAQ/640?wx_fmt=png)

修改并且重启ntpd后，ntpd 进程不带-x参数

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYumd3uM96nJB3g5Q4YLqnes69hhEAaIq3lDxTOsiabEbWqP21Cd07uBw/640?wx_fmt=png)

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYO41Y2p6rpE0yIexseybxH8UVHWmMnIRCuY2gaHBlqJuHx0pLqm31sQ/640?wx_fmt=png)

# 5. 总结

1) 根据KB【2】的解释，NTP启动中有-x和没有-x是如何影响Kudu tablet servers已经很清楚了。

【2】

```
https://kudu.apache.org/releases/1.13.0/docs/troubleshooting.html#_monitoring_clock_synchronization_status_with_the_ntp_suite
```

![](https://mmbiz.qpic.cn/mmbiz_png/THumz4762QC0yNwzWtOFKU9Q7CXBN6AYI7AfDwN2yzrRj6WF2C1JYEZEv3RYxyuy2lmuScXQ4x7WwyAFAQhnNQ/640?wx_fmt=png)

2) ntpd服务的方式，有两种策略，一种是平滑、缓慢的渐进式调整（adjusts the clock in small steps所谓的微调）；一种是步进式调整（跳跃式调整）。两种策略的区别就在于，微调方式在启动NTP服务时加了个“-x”的参数，而默认的是不加“-x”参数。<br>
假如使用了-x选项，那么ntpd只做微调，不跳跃调整时间，但是要注意，-x参数的负作用：当时钟差大的时候，同步时间将花费很长的时间。-x也有一个阈值，就是600s，当系统时钟与标准时间差距大于600s时，ntpd会使用较大“步进值”的方式来调整时间，将时钟“步进”调整到正确时间。假如不使用-x选项，那么ntpd在时钟差距小于128ms时，使用微调方式调整时间，当时差大于128ms时，使用“跳跃”式调整。