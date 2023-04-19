[TOC]

# 1. 为什么Kafka不支持读写分离
在 Kafka 中，生产者写入消息、消费者读取消息的操作都是与 leader 副本进行交互的，从 而实现的是一种主写主读的生产消费模型。数据库、Redis 等都具备主写主读的功能，与此同时还支持主写从读的功能，主写从读也就是读写分离，为了与主写主读对应，这里就以主写从读来称呼。**==Kafka 并不支持主写从读，这是为什么呢?==**

## 1.1 主写从读的缺点
从代码层面上来说，虽然增加了代码复杂度，但在 Kafka 中这种功能完全可以支持。对于 这个问题，我们可以从“收益点”这个角度来做具体分析。主写从读可以让从节点去分担主节 点的负载压力，预防主节点负载过重而从节点却空闲的情况发生。但是**主写从读也有2个很明显的缺点**:

1) **数据一致性问题**。数据从主节点转到从节点必然会有一个延时的时间窗口，这个时间 窗口会导致主从节点之间的数据不一致。某一时刻，在主节点和从节点中 A 数据的值都为 X， 之后将主节点中 A 的值修改为 Y，那么在这个变更通知到从节点之前，应用读取从节点中的 A 数据的值并不为最新的 Y，由此便产生了数据不一致的问题。
    
2) **延时问题**。类似 Redis 这种组件，数据从写入主节点到同步至从节点中的过程需要经 历网络→主节点内存→网络→从节点内存这几个阶段，整个过程会耗费一定的时间。而在 Kafka 中，主从同步会比 Redis 更加耗时，它需要经历网络→主节点内存→主节点磁盘→网络→从节 点内存→从节点磁盘这几个阶段。对延时敏感的应用而言，主写从读的功能并不太适用。
    

现实情况下，很多应用既可以忍受一定程度上的延时，也可以忍受一段时间内的数据不一 致的情况，那么对于这种情况，Kafka 是否有必要支持主写从读的功能呢?

主读从写可以均摊一定的负载却不能做到完全的负载均衡，比如对于数据写压力很大而读 压力很小的情况，从节点只能分摊很少的负载压力，而绝大多数压力还是在主节点上。而在 Kafka 中却可以达到很大程度上的负载均衡，而且这种均衡是在主写主读的架构上实现的。

## 1.2 Kafka生产消费模型
我们来看一下 Kafka 的生产消费模型，如下图所示。

[![Kafka的生产消费模型.jpg](https://huawei.best/2020/11/20/a59b86b0b7d4e.jpg)](https://www.helloimg.com/images/2020/11/06/Kafka8d41ef27e83dce30.jpg)

在 Kafka 集群中有 3 个分区，每个分区有 3 个副本，正好均匀地分布在 3个 broker 上，蓝色阴影的代表 leader 副本，非蓝色阴影的代表 follower 副本，虚线表示 follower 副本从 leader 副本上拉取消息。当生产者写入消息的时候都写入 leader 副本，对于图中的 情形，每个 broker 都有消息从生产者流入;当消费者读取消息的时候也是从 leader 副本中读取 的，对于图中的情形，每个 broker 都有消息流出到消费者。

我们很明显地可以看出，每个 broker 上的读写负载都是一样的，这就说明 **Kafka 可以通过 主写主读实现主写从读实现不了的负载均衡**。

## 1.3 Kafka负载不均衡现象
上图展示是一种理想的部署情况，有以下几种 情况(包含但不仅限于)会造成一定程度上的负载不均衡:

1. **broker端的分区分配不均**。当创建主题的时候可能会出现某些 broker 分配到的分区数 多而其他 broker 分配到的分区数少，那么自然而然地分配到的 leader 副本也就不均。

2. **生产者写入消息不均**。生产者可能只对某些 broker 中的 leader 副本进行大量的写入操作，而对其他 broker 中的 leader 副本不闻不问。

3. **消费者消费消息不均**。消费者可能只对某些 broker 中的 leader 副本进行大量的拉取操 作，而对其他 broker 中的 leader 副本不闻不问。

4. **leader 副本的切换不均**。在实际应用中可能会由于 broker 宕机而造成主从副本的切换， 或者分区副本的重分配等，这些动作都有可能造成各个 broker 中 leader 副本的分配不均。
    

对此，我们可以做一些防范措施。针对第一种情况，在主题创建的时候尽可能使分区分配 得均衡，好在 Kafka 中相应的分配算法也是在极力地追求这一目标，如果是开发人员自定义的 分配，则需要注意这方面的内容。对于第二和第三种情况，主写从读也无法解决。对于第四种情况，Kafka 提供了优先副本的选举来达到 leader 副本的均衡，与此同时，也可以配合相应的 监控、告警和运维平台来实现均衡的优化。

在实际应用中，配合监控、告警、运维相结合的生态平台，在绝大多数情况下 Kafka 都能 做到很大程度上的负载均衡。

## 1.4 Kafka只支持主写主读优点
总的来说，==Kafka只支持主写主读有几个优点==:

1. 可以简化代码的 实现逻辑，减少出错的可能;
2. 将负载粒度细化均摊，与主写从读相比，不仅负载效能更好，而 且对用户可控;
3. 没有延时的影响;
4. 在副本稳定的情况下，不会出现数据不一致的情况。

为此， Kafka 又何必再去实现对它而言毫无收益的主写从读的功能呢?这一切都得益于 Kafka 优秀的 架构设计，从某种意义上来说，主写从读是由于设计上的缺陷而形成的权宜之计。

# 2. kafka 的数据是放在磁盘上还是内存上，为什么速度会快？

**kafka 使用的是磁盘存储**。速度快是因为：

- **顺序写入**：因为硬盘是机械结构，每次读写都会寻址->写入，其中寻址是一个“机械动作”，它是耗时的。所以硬盘 “讨厌”随机I/O， 喜欢顺序I/O。为了提高读写硬盘的速度，Kafka 就是使用顺序I/O。Memory Mapped Files（内存映射文件）：64位操作系统中一般可以表示20G 的数据文件，它的工作原理是直接利用操作系统的Page 来实现文件到物理内存的直接映射。完成映射之后你对物理内存的操作会被同步到硬盘上。
- **Kafka 高效文件存储设计**： Kafka 把topic 中一个parition 大文件分成多个小文件段，通过多个小文件段，就容易定期清除或删除已经消费完文件，减少磁盘占用。通过索引信息可以快速定位message 和确定response 的 大 小。通过index 元数据全部映射到memory（内存映射文件），可以避免segment file 的IO 磁盘操作。通过索引文件稀疏存储，可以大幅降低index 文件元数据占用空间大小。


# 3. Kafka 数据怎么保障不丢失？
分三个点说，一个是生产者端，一个消费者端，一个broker 端。

## 3.1 生产者数据的不丢失
kafka 的ack 机制：在kafka 发送数据的时候，每次发送消息都会有一个确认反馈机制，确保消息正常的能够被收到，其中状态有0，1，-1。

如果是同步模式：<br>
ack 设置为0，风险很大，一般不建议设置为0。即使设置为1，也会随着leader 宕机丢失数据。所以如果要严格保证生产端数据不丢失，可设置为-1。

如果是异步模式：<br>
也会考虑ack 的状态，除此之外，异步模式下的有个buffer，通过buffer 来进行控制数据的发送，有两个值来进行控制，时间阈值与消息的数量阈值，如果buffer 满了数据还没有发送出去，有个选项是配置是否立即清空buffer。可以设置为-1，永久阻塞，也就数据不再生产。异步模式下，即使设置为-1。也可能因为程序员的不科学操作，操作数据丢失，比如kill -9，但这是特别的例外情况。

> 注：<br>
> - ack=0：producer 不等待broker 同步完成的确认，继续发送下一条(批)信息。
> - ack=1（默认）：producer 要等待leader 成功收到数据并得到确认，才发送下一条message。
> - ack=-1：producer 得到follwer 确认，才发送下一条数据。

## 3.2 消费者数据的不丢失
通过offset commit 来保证数据的不丢失，kafka 自己记录了每次消费的offset 数值，下次继续消费的时候，会接着上次的offset 进行消费。而offset 的信息在kafka0.8 版本之前保存在zookeeper 中，在0.8 版本之后保存到topic 中，即使消费者在运行过程中挂掉了，再次启动的时候会找到offset 的值，找到之前消费消息的位置，接着消费，由于 offset 的信息写入的时候并不是每条消息消费完成后都写入的，所以这种情况有可能会造成重复消费，但是不会丢失消息。

唯一例外的情况是，我们在程序中给原本做不同功能的两个consumer 组设置KafkaSpoutConfig.bulider.setGroupid 的时候设置成了一样的groupid，这种情况会导致这两个组共享同一份数据，就会产生组A 消费partition1，partition2 中的消息，组B 消费partition3 的消息，这样每个组消费的消息都会丢失，都是不完整的。为了保证每个组都独享一份消息数据，groupid 一定不要重复才行。

## 3.3 kafka 集群中的broker 的数据不丢失
每个broker 中的partition 我们一般都会设置有replication（副本）的个数，生产者写入的时候首先根据分发策略（有partition 按partition，有key 按key，都没有轮询）写入到leader 中，follower（副本）再跟leader 同步数据，这样有了备份，也可以保证消息数据的不丢失。

# 4. 重启是否会导致数据丢失？
kafka 是将数据写到磁盘的，一般数据不会丢失。但是在重启kafka 过程中，如果有消费者消费消息，那么kafka 如果来不及提交offset，可能会造成数据的不准确（丢失或者重复消费）。

# 5. kafka 宕机了如何解决？
**先考虑业务是否受到影响**<br>
kafka 宕机了，首先我们考虑的问题应该是所提供的服务是否因为宕机的机器而受到影响，如果服务提供没问题，如果实现做好了集群的容灾机制，那么这块就不用担心了。

**节点排错与恢复**<br>
想要恢复集群的节点，主要的步骤就是通过日志分析来查看节点宕机的原因，从而解决，重新恢复节点。

# 6. Kafka 消息数据积压，Kafka消费能力不足怎么处理？
**如果是Kafka 消费能力不足**，则可以考虑增加Topic 的分区数，并且同时提升消费组的消费者数量，消费者数=分区数。（两者缺一不可）

**如果是下游的数据处理不及时**：
提高每批次拉取的数量。批次拉取数据过少（拉取数据/处理时间<生产速度），使处理的数据小于生产的数据，也会造成数据积压。

# 7. kafka 的数据offset读写流程
## 7.1 读数据
1) 连接ZK集群，从ZK中拿到对应topic的partition信息和partition的Leader的相关信息
2) 连接到对应Leader对应的broker
3) consumer将自己保存的offset发送给Leader
4) Leader根据offset等信息定位到segment（索引文件和日志文件）
5) 根据索引文件中的内容，定位到日志文件中该偏移量对应的开始位置读取相应长度的数据并返回给consumer

## 7.2 写数据
[![image](https://oscimg.oschina.net/oscnet/4b2e6c9987543a6b420c5cb218c1127d473.jpg)](https://i.loli.net/2021/04/09/VmfIdxkh14qK3RG.png)

1) 连接ZK集群，从ZK中拿到broker-list的节点partition的Leader的相关信息(对应topic的partition信息和partition的Leader的相关信息)
2) 连接到对应Leader对应的broker
3) 将消息发送到partition的Leader上
4) leader收到消息后，将消息写入本地log；
5) followers从leader中pull消息，实现replication的副本备份机制，同样写入本地log；
6) replication写入本地log后向leader发送ack（确认）；
7) leader收到所有的replication的ack之后，向producer发送ack
8) producer收到leader的ack，才完成提交，整个写过程结束
# 8. kafka 内部如何保证顺序，结合外部组件如何保证消费者的顺序？
kafka 只能保证partition 内是有序的，但是partition 间的有序是没办法的。

爱奇艺的搜索架构，是从业务上把需要有序的打到同⼀一个partition

可以将Kafka的数据添加时间戳，将数据拉取后在业务上根据时间戳排序

# 9. 高可用

## 9.1 replication(复制)

topic的每个partition都有1到N个分区，每个分区有多个replica，多个replica中有一个是Leader，其他都是Follower，Leader负责响应producer和consumer的读写请求。一旦有数据写到Leader,则所有的Follower都会从Leader中去同步数据，但并非所有Follower都能及时同步，所以kafka将所有的replica分成两个组：ISR和OSR。ISR是与Leader数据同步的Follower，而OSR是与Leader数据不同步的Follower

## 9.2 Leader failover(Leader失败恢复)

为了保证数据一致性，当Leader挂了之后，kafka的controller默认会从ISR中选择一个replica作为Leader继续工作，选择的条件是：新Leader必须有挂掉Leader的所有数据。

如果为了系统的可用性，而容忍降低数据的一致性的话，可以将"unclean.leader.election.enable = true" ，开启kafka的"脏Leader选举"。当ISR中没有replica，则会从OSR中选择一个replica作为Leader继续响应请求，如此操作提高了Kafka的分区容忍度，但是数据一致性降低了。

## 9.3 broker failover(broker失败恢复)

broker挂了比单个partition的Leader挂了要做的事情多很多，因为一个broker上面有很多partition和多个Leader。因此至少需要处理如下内容：

1.更新该broker上所有Follower的状态

2.重新在其他broker上给原来挂掉的broker上的partition（原来是leader的partition）选举Leader

3.选举完成后，要更新partition的状态，比如谁是Leader等

kafka集群启动后，所有的broker都会被controller监控，一旦有broker宕机，zk的监听机制会通知到controller，controller拿到挂掉broker中所有的partition，以及它上面的存在的leader，然后从partition的ISR中选择一个Follower作为Leader，更改partition的follower和leader状态。

## 9.4 contoller failover(controller失败恢复)

当 controller 宕机时会触发 controller failover。每个 broker 都会在 zookeeper 的 "/controller" 节点注册 watcher，当 controller 宕机时 zookeeper 中的临时节点消失，所有存活的 broker 收到 fire 的通知，每个 broker 都尝试创建新的 controller path，只有一个竞选成功并当选为 controller。