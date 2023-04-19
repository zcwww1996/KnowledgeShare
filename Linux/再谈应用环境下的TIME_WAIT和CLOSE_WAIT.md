[TOC]

**参考资料：** 
 [https://blog.csdn.net/shootyou/article/details/6622226](https://blog.csdn.net/shootyou/article/details/6622226)
 
昨天解决了一个 HttpClient 调用错误导致的服务器异常，具体过程如下：

[http://blog.csdn.net/shootyou/article/details/6615051](http://blog.csdn.net/shootyou/article/details/6615051)

里头的分析过程有提到，通过查看服务器网络状态检测到服务器有大量的 CLOSE_WAIT 的状态。

在服务器的日常维护过程中，会经常用到下面的命令：

```bash
netstat -n | awk '/^tcp/ {++S[$NF]} END {for(a in S) print a, S[a]}'
```

它会显示例如下面的信息：

TIME_WAIT 814  
CLOSE_WAIT 1  
FIN_WAIT1 1  
ESTABLISHED 634  
SYN_RECV 2  
LAST_ACK 1  

常用的三个状态是：ESTABLISHED 表示正在通信，TIME_WAIT 表示主动关闭，CLOSE_WAIT 表示被动关闭。

具体每种状态什么意思，其实无需多说，看看下面这种图就明白了，注意这里提到的服务器应该是业务请求接受处理的一方：

[![](https://dingyue.ws.126.net/2020/0916/07f60f28j00qgqs19000hc0009k009rm.jpg)](https://i0.wp.com/files.catbox.moe/o4n665.jpg)

这么多状态不用都记住，只要了解到我上面提到的最常见的三种状态的意义就可以了。一般不到万不得已的情况也不会去查看网络状态，如果服务器出了异常，百分之八九十都是下面两种情况：

1\. 服务器保持了大量 TIME_WAIT 状态

2\. 服务器保持了大量 CLOSE_WAIT 状态

因为 linux 分配给一个用户的文件句柄是有限的（可以参考：[http://blog.csdn.net/shootyou/article/details/6579139](http://blog.csdn.net/shootyou/article/details/6579139)），而 TIME_WAIT 和 CLOSE_WAIT 两种状态如果一直被保持，那么意味着对应数目的通道就一直被占着，而且是 “占着茅坑不使劲”，一旦达到句柄数上限，新的请求就无法被处理了，接着就是大量 Too Many Open Files 异常，tomcat 崩溃。。。

下面来讨论下这两种情况的处理方法，网上有很多资料把这两种情况的处理方法混为一谈，以为优化系统内核参数就可以解决问题，其实是不恰当的，优化系统内核参数解决 TIME_WAIT 可能很容易，但是应对 CLOSE_WAIT 的情况还是需要从程序本身出发。现在来分别说说这两种情况的处理方法：

# 1. 服务器保持了大量 TIME_WAIT 状态

这种情况比较常见，一些爬虫服务器或者 WEB 服务器（如果网管在安装的时候没有做内核参数优化的话）上经常会遇到这个问题，这个问题是怎么产生的呢？

从上面的示意图可以看得出来，TIME_WAIT 是主动关闭连接的一方保持的状态，对于爬虫服务器来说他本身就是 “客户端”，在完成一个爬取任务之后，他就会发起主动关闭连接，从而进入 TIME_WAIT 的状态，然后在保持这个状态 2MSL（max segment lifetime）时间之后，彻底关闭回收资源。为什么要这么做？明明就已经主动关闭连接了为啥还要保持资源一段时间呢？这个是 TCP/IP 的设计者规定的，主要出于以下两个方面的考虑：

1. 防止上一次连接中的包，迷路后重新出现，影响新连接（经过 2MSL，上一次连接中所有的重复包都会消失）  
2. 可靠的关闭 TCP 连接。在主动关闭方发送的最后一个 ack(fin) ，有可能丢失，这时被动方会重新发 fin, 如果这时主动方处于 CLOSED 状态 ，就会响应 rst 而不是 ack。所以主动方要处于 TIME_WAIT 状态，而不能是 CLOSED 。另外这么设计 TIME_WAIT 会定时的回收资源，并不会占用很大资源的，除非短时间内接受大量请求或者受到攻击。

关于 MSL 引用下面一段话：


> MSL 为一个 TCP Segment (某一块 TCP 网路封包) 从来源送到目的之间可续存的时间 (也就是一个网路封包在网路上传输时能存活的时间)，由于 RFC 793 TCP 传输协定是在 1981 年定义的，当时的网路速度不像现在的网际网路那样发达，你可以想像你从鲮览器输入网址等到第一个 byte 出现要等 4 分钟吗？
>
> 在现在的网路环境下几乎不可能有这种事情发生，因此我们大可将 TIME_WAIT 状态的续存时间大幅调低，好让连线埠 (Ports) 能更快空出来给其他连线使用。


再引用网络资源的一段话：


> 值得一说的是，对于基于TCP的HTTP协议，关闭TCP连接的是Server端，这样，Server端会进入TIME_WAIT状态，可 想而知，对于访问量大的Web Server，会存在大量的TIME_WAIT状态，假如server一秒钟接收1000个请求，那么就会积压240*1000=240，000个 TIME_WAIT的记录，维护这些状态给Server带来负担。当然现代操作系统都会用快速的查找算法来管理这些TIME_WAIT，所以对于新的 TCP连接请求，判断是否hit中一个TIME_WAIT不会太费时间，但是有这么多状态要维护总是不好。
>
> HTTP协议1.1版规定default行为是Keep-Alive，也就是会重用TCP连接传输多个 request/response，一个主要原因就是发现了这个问题。


也就是说 HTTP 的交互跟上面画的那个图是不一样的，关闭连接的不是客户端，而是服务器，所以 web 服务器也是会出现大量的 TIME_WAIT 的情况的。

现在来说如何来解决这个问题。

解决思路很简单，就是让服务器能够快速回收和重用那些 TIME_WAIT 的资源。

下面来看一下我们网管对 /etc/sysctl.conf 文件的修改：

```bash

net.ipv4.tcp_syn_retries=2


net.ipv4.tcp_keepalive_time=1200
net.ipv4.tcp_orphan_retries=3

net.ipv4.tcp_fin_timeout=30

net.ipv4.tcp_max_syn_backlog = 4096

net.ipv4.tcp_syncookies = 1


net.ipv4.tcp_tw_reuse = 1

net.ipv4.tcp_tw_recycle = 1


net.ipv4.tcp_keepalive_probes=5

net.core.netdev_max_backlog=3000
```

修改完之后执行 `/ sbin/sysctl -p` 让参数生效。

这里头主要注意到的是
- `net.ipv4.tcp_tw_reuse`
- `net.ipv4.tcp_tw_recycle`
- `net.ipv4.tcp_fin_timeout`
- `net.ipv4.tcp_keepalive_*`

这几个参数。

net.ipv4.tcp_tw_reuse 和 net.ipv4.tcp_tw_recycle 的开启都是为了回收处于 TIME_WAIT 状态的资源。  

net.ipv4.tcp_fin_timeout 这个时间可以减少在异常情况下服务器从 FIN-WAIT-2 转到 TIME_WAIT 的时间。  

net.ipv4.tcp_keepalive\_\*一系列参数，是用来设置服务器检测连接存活的相关配置。  

**\[2015.01.13 更新]**

**注意 tcp_tw_recycle 开启的风险：**[http://blog.csdn.net/wireless\\\_tech/article/details/6405755](http://blog.csdn.net/wireless\_tech/article/details/6405755)

# 2. 服务器保持了大量 CLOSE_WAIT 状态

休息一下，喘口气，一开始只是打算说说 TIME_WAIT 和 CLOSE_WAIT 的区别，没想到越挖越深，这也是写博客总结的好处，总可以有意外的收获。

TIME_WAIT 状态可以通过优化服务器参数得到解决，因为发生 TIME_WAIT 的情况是服务器自己可控的，要么就是对方连接的异常，要么就是自己没有迅速回收资源，总之不是由于自己程序错误导致的。

但是 CLOSE_WAIT 就不一样了，从上面的图可以看出来，如果一直保持在 CLOSE_WAIT 状态，那么只有一种情况，就是在对方关闭连接之后服务器程序自己没有进一步发出 ack 信号。换句话说，就是在对方连接关闭之后，程序里没有检测到，或者程序压根就忘记了这个时候需要关闭连接，于是这个资源就一直被程序占着。个人觉得这种情况，通过服务器内核参数也没办法解决，服务器对于程序抢占的资源没有主动回收的权利，除非终止程序运行。

在那边日志里头我举了个场景，来说明 CLOSE_WAIT 和 TIME_WAIT 的区别，这里重新描述一下：

服务器 A 是一台爬虫服务器，它使用简单的 HttpClient 去请求资源服务器 B 上面的 apache 获取文件资源，正常情况下，如果请求成功，那么在抓取完资源后，服务器 A 会主动发出关闭连接的请求，这个时候就是主动关闭连接，服务器 A 的连接状态我们可以看到是 TIME_WAIT。如果一旦发生异常呢？假设请求的资源服务器 B 上并不存在，那么这个时候就会由服务器 B 发出关闭连接的请求，服务器 A 就是被动的关闭了连接，如果服务器 A 被动关闭连接之后程序员忘了让 HttpClient 释放连接，那就会造成 CLOSE_WAIT 的状态了。  

所以如果将大量 CLOSE_WAIT 的解决办法总结为一句话那就是：查代码。因为问题出在服务器程序里头啊。

**参考资料：** 
 [https://blog.csdn.net/shootyou/article/details/6622226](https://blog.csdn.net/shootyou/article/details/6622226)
