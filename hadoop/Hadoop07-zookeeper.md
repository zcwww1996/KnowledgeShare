[TOC]
# 1. zookeeper安装
## 1.1. zookeeper的集群部署
### 1.1.1 准备环境
1、上传安装包到集群服务器

2、解压zookeeper-3.4.6.tar.gz包

### 1.1.2 修改配置文件

进入zookeeper的安装目录的conf目录

**cp zoo_sample.cfg zoo.cfg**

**vi zoo.cfg**

```bash
# The number of milliseconds of each tick

tickTime=2000

initLimit=10

syncLimit=5

dataDir=/root/zkdata

clientPort=2181


#autopurge.purgeInterval=1

server.1=linux01:2888:3888

server.2=linux02:2888:3888

server.3=linux03:2888:3888
```


 

对3台节点，都创建目录 mkdir /root/zkdata

对3台节点，<font color="red">在工作目录中生成myid文件</font>，但内容要分别为各自的id：1,2,3

linux01上：  echo 1 > /root/zkdata/myid

linux02上：  echo 2 > /root/zkdata/myid

linux03上：  echo 3 > /root/zkdata/myid

 

### 1.1.3 远程复制配置文件
linux01上scp安装目录到其他两个节点

scp -r zookeeper-3.4.6/ Linux02:$PWD

scp -r zookeeper-3.4.6/ Linux03:$PWD

 

### 1.1.4 启动zookeeper集群

zookeeper没有提供自动批量启动脚本，需要手动一台一台地起zookeeper进程

在每一台节点上，运行命令：


```bash
bin/zkServer.sh start
```


启动后，用jps应该能看到一个进程：**QuorumPeerMain**

但是，光有进程不代表zk已经正常服务，需要用命令检查状态：


```bash
bin/zkServer.sh status
```

能看到角色模式：为**leader**或**follower**，即正常了。

# 2. zookeeper选举机制

- SID：<font style="color: red;">服务器ID</font>。用来唯一标识一台ZooKeeper集群中的机器，每台机器不能重复，<font style="color: red;">和myid一致</font>。
- ZXID：事务ID。<font style="color: red;">ZXID是一个事务ID，用来标识一次服务器状态的变更</font>。在某一时刻，集群中的每台机器的ZXID值不一定完全一致，这和ZooKeeper服务器对于客户端“更新请求”的处理逻辑有关。
- Epoch：<font style="color: red;">每个Leader任期的代号</font>。没有Leader时同一轮投票过程中的逻辑时钟值是相同的。每投完一次票这个数据就会增加

## 2.1 zookeeper第一次启动
[![zookeeper第一次启动](https://pic.imgdb.cn/item/622724be5baa1a80ab4ab83c.png "zookeeper第一次启动")](https://www.helloimg.com/images/2022/03/08/R5TuS6.png)

**选举步骤**：

1) 服务器1启动，发起一次选举。服务器1投自己一票。此时服务器1票数一票，不够半数以上（3票），选举无法完成，服务器1状态保持为LOOKING；

2) 服务器2启动，再发起一次选举。服务器1和2分别投自己一票并交换选票信息：<font style="color: red;">此时服务器1发现服务器2的myid比自己目前投票推举的（服务器1）大</font>，更改选票为推举服务器2。此时服务器1票数0票，服务器2票数2票，没有半数以上结果，选举无法完成，服务器1，2状态保持LOOKING；

3) 服务器3启动，发起一次选举。此时服务器1和2都会更改选票为服务器3。此次投票结果：服务器1为0票，服务器2为0票，服务器3为3票。此时服务器3的票数已经超过半数，服务器3当选Leader。服务器1，2更改状态为FOLLOWING，服务器3更改状态为LEADING；

4) 服务器4启动，发起一次选举。此时服务器1，2，3已经不是LOOKING状态，不会更改选票信息。交换选票信息结果：服务器3为3票，服务器4为1票。此时服务器4服从多数，更改选票信息为服务器3，并更改状态为FOLLOWING；

5) 服务器5启动，同4一样当小弟。

## 2.2 zookeeper非第一次启动

[![zookeeper非第一次启动](https://pic.imgdb.cn/item/622724be5baa1a80ab4ab841.png "zookeeper非第一次启动")](https://www.helloimg.com/images/2022/03/08/R5Typn.png)

**选举步骤**：

1) 当ZooKeeper集群中的一台服务器出现以下两种情况之一时，就会开始进入Leader选举：
   - 服务器初始化启动。
   - 服务器运行期间无法和Leader保持连接。

2) 而当一台机器进入Leader选举流程时，当前集群也可能会处于以下两种状态：
   - **集群中本来就已经存在一个Leader**。<br>
对于第一种已经存在Leader的情况，机器试图去选举Leader时，会被告知当前服务器的Leader信息，对于该机器来说，仅仅需要和Leader机器建立连接，并进行状态同步即可。
   - **集群中确实不存在Leader**。<br>
假设ZooKeeper由5台服务器组成，Epoch分别为1、1、1、1、1，SID分别为1、2、3、4、5，ZXID分别为8、8、8、7、7，并且此时SID为3的服务器是Leader。某一时刻，3和5服务器出现故障，因此开始进行Leader选举。

SID为1、2、4的机器情况：

|SID|（EPOCH，ZXID，SID ）|
|---|---|
| 1 |（1，8，1）|
| 2 |（1，8，2）|
| 4 |（1，7，4）|

最终，`SID_2` 为Leader，`SID_1` 、`SID_4` 为Folower

**选举Leader规则**：
- **<font style="color: red;">(1) EPOCH大的直接胜出</font>**
- **<font style="color: red;">(2) EPOCH相同，zxid大的胜出</font>**
- **<font style="color: red;">(3) zxid相同，sid大的胜出**

http://www.jasongj.com/zookeeper/fastleaderelection/

https://blog.csdn.net/wangshouhan/article/details/89919404
