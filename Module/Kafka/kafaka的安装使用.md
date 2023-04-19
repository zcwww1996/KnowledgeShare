[TOC]

Kafka配置官网：http://kafka.apache.org/documentation.html#streamsconfigs<br>
Kafka中文配置参考：https://kafka.apachecn.org/documentation.html#producerapi
# 1. kafaka
## 1.1 搭建kafaka集群
### 1.1.1 前期配置
**1. 安装JDK &配置JAVA_HOME**

**2. 安装ZooKkeeper,配置ZooKeeper集群**
### 1.1.2 Kafka配置

Kafka配置官网：http://kafka.apache.org/documentation.html#streamsconfigs

**1、 解压kafaka**

**2、修改配置文件config/server.properties、zookeeper.properties**

**vi server.properties**
```shell
# 为依次增长的：0、1、2、3、4，集群中唯一id
broker.id=1
# 监听的主机（每台机器不同）
listeners=PLAINTEXT://linux01:9092
# Kafka 的消息数据存储路径
log.dirs=/root/kafka-logs
# zookeeperServers 列表，各节点以逗号分开
zookeeper.connect=linux01:2181,linux02:2181,linux03:2181
```

**vi zookeeper.properties**
```shell
# 指向你安装的zk 的数据存储目录
dataDir=/root/zkdata
```
**将kafka_2.11-0.10.2.1文件拷贝到其他节点机器**

```shell
scp kafka_2.11-0.10.2.1 linux02:$PWD
scp kafka_2.11-0.10.2.1 linux03:$PWD
```

**3、启动Kafka**
在<font color="red">**每台节点上**</font>启动：
启动前先启动zookeeper
```zkServer.sh start```
```shell
bin/kafka-server-start.sh -daemon config/server.properties
```
### 1.1.3 测试集群
- **1-进入kafka 根目录，创建Topic 名称为： Mytest 的主题**

```shell
bin/kafka-topics.sh --create --zookeeper 192.168.134.111:2181,192.168.134.112:2181,192.168.134.113:2181 --replication-factor 3 --partitions 1 --topic Mytest
```
删除创建的主题( server.properties中设置`delete.topic.enable=true` 否则只是标记删除或者直接重启)
```shell
bin/kafka-topics.sh --zookeeper localhost:2181 --delete --topic myTopic,Mytest
```

- **2-列出已创建的topic列表**

```shell

## topic列表查询
bin/kafka-topics.sh --zookeeper 192.168.134.111:2181 --list

## topic列表查询（支持0.9版本+）
bin/kafka-topics.sh --list --bootstrap-server 192.168.134.111:9092

```

- **3-查看Topic 的详细信息**

```shell
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic Mytest
```

```shell
Topic:test PartitionCount:1 ReplicationFactor:3 Configs:
Topic: test Partition: 0 Leader: 1 Replicas: 1,2,0 Isr: 1,2,0
第一行是对所有分区的一个描述，然后每个分区对应一行，因为只有一个分区所以下面一行。
leader：负责处理消息的读和写，leader 是从所有节点中随机选择的.
replicas：列出了所有的副本节点，不管节点是否在服务中.
isr：是正在服务中的节点.
在例子中，节点1 是作为leader 运行。
```

- **4-模拟客户端去发送消息**

```shell
bin/kafka-console-producer.sh --broker-list 192.168.134.111:9092,192.168.134.112:9092 --topic Mytest

## 新生产者（支持0.9版本+）
bin/kafka-console-producer.sh --broker-list 192.168.134.111:9092,192.168.134.112:9092 --topic Mytest --producer.config config/producer.properties
```

- **5-模拟客户端去接受消息**

```shell
#接收最新的消息（消费者创建后产生的消息）
bin/kafka-console-consumer.sh --bootstrap-server linux01:9092 --topic Mytest

#上面的命令只能获取消费者创建以后的消息，如果要获取之前的消息，需要使用下面的语句：
bin/kafka-console-consumer.sh --bootstrap-server linux01:9092 --from-beginning --topic Mytest

## 0.8 消费者
bin/kafka-console-consumer.sh --zookeeper 192.168.134.111:2181 --topic Mytest
## 新消费者（支持0.9版本+）
bin/kafka-console-consumer.sh --bootstrap-server 192.168.134.111:9092 --topic Mytest --new-consumer --from-beginning --consumer.config config/consumer.properties
```
- **6-查看topic各分区最大能消费到的offset**

```shell
#--time -1偏移量最大值   --time -2偏移量最小值
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --topic Mytest --time -1 --broker-list 192.168.134.111:9092,192.168.134.112:9092,192.168.134.113:9092

#  --partitions 0,1,2,3,4,5可指定分区
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --topic iptv_gd --time -1 --broker-list 172.41.135.24:9091,172.41.135.25:9091,172.41.135.26:9091 --partitions 0,1,2,3,4,5
```

- **7- 查看某个分区最新10条记录**
```shell
bin/ifneed-kafka-console-consumer.sh --bootstrap-server 172.41.135.24:9091,172.41.135.25:9091,172.41.135.26:9091 --topic iptv_gd --partition 0 --offset 3256184965  --max-messages 10

# 高级用法
bin/kafka-simple-consumer-shell.sh --brist 192.168.134.111:9092 --topic Mytest --partition 0 --offset 1234  --max-messages 10
```

- **8-测试一下容错能力**

```shell
Kill -9 pid[leader 节点]
```

另外一个节点被选做了leader,node 1 不再出现在in-sync 副本列表中：
```shell
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic test
Topic:test PartitionCount:1 ReplicationFactor:3 Configs:
Topic: test Partition: 0 Leader: 2 Replicas: 1,2,0 Isr: 2,0
```
虽然最初负责续写消息的leader down 掉了，但之前的消息还是可以消费的：
```shell
bin/kafka-console-consumer.sh --zookeeper localhost:2181 --from-beginning --topic test
```

- **9-平衡leader**
```bash
bin/kafka-preferred-replica-election.sh --zookeeper 192.168.134.111:2181/chroot
```

- **10-kafka自带压测命令**
```bash
bin/kafka-producer-perf-test.sh --topic test --num-records 100 --record-size 1 --throughput 100  --producer-props bootstrap.servers=192.168.134.111:9092
```

### 1.1.4 kafka分区管理
#### 1.1.4.1 分区扩容
```bash
bin/kafka-topics.sh --zookeeper 192.168.134.111:2181 --alter --topic topic1 --partitions 2
```
#### 1.1.4.2 迁移分区
1. 创建规则json

```bash
cat > increase-replication-factor.json <<EOF
{"version":1, "partitions":[
{"topic":"__consumer_offsets","partition":0,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":1,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":2,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":3,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":4,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":5,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":6,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":7,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":8,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":9,"replicas":[0,1]},
{"topic":"__consumer_offsets","partition":10,"replicas":[0,1]}]
}
EOF

```


2. 执行

```bash
bin/kafka-reassign-partitions.sh --zookeeper 192.168.134.111:2181 --reassignment-json-file increase-replication-factor.json --execute
```

3. 验证

```bash
bin/kafka-reassign-partitions.sh --zookeeper 192.168.134.111:2181 --reassignment-json-file increase-replication-factor.json --verify
```


## 1.2 offset存储方式

*   1、在kafka 0.9版本之后，kafka**为了降低zookeeper的io读写，减少network data transf**er，也自己实现了在kafka server上存储consumer，topic，partitions，offset信息将消费的 offset 迁入到了 Kafka 一个名为 **__consumer_offsets** 的Topic中。
*   2、将消费的 offset 存放在 Zookeeper 集群中。
*   3、将offset存放至第三方存储，如Redis, 为了严格实现不重复消费


### 1.2.1  kafka0.8版本消费者状态检查

```bash
bin/kafka-consumer-offset-checker.sh --group flume --topic testTopic --zookeeper 192.168.134.111:2181
```
### 1.2.2 kafka0.9以上版本消费者状态检查
#### 1.2.2.1 kafka维护消费偏移量的情况
（1）查看Kafka有哪些消费者group，如果已知，此步骤可忽略

```bash

## 新消费者列表查询（支持0.9版本+）
bin/kafka-consumer-groups.sh --new-consumer --bootstrap-server 192.168.134.111:9092 --list

## 新消费者列表查询（支持0.10版本+）
bin/kafka-consumer-groups.sh --bootstrap-server 192.168.134.111:9092 --list

Note: This will only show information about consumers that use the Java consumer API (non-ZooKeeper-based consumers).

console-consumer-89764
consumer-dev
```
- 192.168.134.111是kafka server的ip地址，9092是server的监听端口。多个host port之间用逗号隔开
- 上面命令是获取group列表，一般而言，应用是知道消费者group的，通常在应用的配置里，**如果已知，该步骤可以省略**

（2）查看指定group.id的消费详情

以consumer-dev为例，查看详情
```bash
## 显示某个消费组的消费详情（0.9版本 - 0.10.1.0 之前）
bin/kafka-consumer-groups.sh --new-consumer --bootstrap-server 192.168.134.111:9092 --describe --group consumer-dev 

## 显示某个消费组的消费详情（0.10.1.0版本+）
bin/kafka-consumer-groups.sh --bootstrap-server 192.168.134.111:9092 --describe --group consumer-dev 


Note: This will only show information about consumers that use the Java consumer API (non-ZooKeeper-based consumers).


TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG        CONSUMER-ID                                       HOST                      CLIENT-ID
t_topic                 0          641             641             0          consumer-1-c313db2b-7758-4de0-8cbd-025997d1a4cc   /192.168.134.111                 consumer-1
t_topic                 1          632             632             0          consumer-1-c313db2b-7758-4de0-8cbd-025997d1a4cc   /192.168.134.111                 consumer-1
t_topic                 2          699             699             0          consumer-1-c313db2b-7758-4de0-8cbd-025997d1a4cc   /192.168.134.111                 consumer-1

```
其中
- TOPIC：该group里消费的topic名称
- PARTITION：分区编号
- CURRENT-OFFSET：该分区当前消费到的offset
- LOG-END-OFFSET：该分区当前latest offset
- LAG：消费滞后区间，为LOG-END-OFFSET-CURRENT-OFFSET，具体大小需要看应用消费速度和生产者速度，一般过大则可能出现消费跟不上，需要引起应用注意
- CONSUMER-ID：server端给该分区分配的consumer编号
- HOST：消费者所在主机
- CLIENT-ID：消费者id，一般由应用指定

#### 1.2.2.2 zookeeper维护消费偏移量的情况(已经逐渐被废弃)

(1) 查看有那些group ID正在进行消费

```bash
bin/kafka-consumer-groups.sh --zookeeper 192.168.134.111:2181 --list
console-consumer-28542
```

(2) 查看指定group.id的消费者消费情况

```bash
bin/kafka-consumer-groups.sh --zookeeper 192.168.134.111:2181 --group console-consumer-28542 --describe
GROUP                          TOPIC                          PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             OWNER
console-consumer-28542         test_find1                     0          303094          303094          0               console-consumer-28542_master-1539167387803-268319a0-0
console-consumer-28542         test_find1                     1          303068          303068          0               console-consumer-28542_master-1539167387803-268319a0-0
console-consumer-28542         test_find1                     2          303713          303713          0               console-consumer-28542_master-1539167387803-268319a0-0
```

备注：
- CURRENT-OFFSET：该分区当前消费到的offset
- LOG-END-OFFSET：该分区当前latest offset