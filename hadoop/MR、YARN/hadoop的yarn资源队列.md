[TOC]

来源：https://blog.csdn.net/qq_21383435/article/details/82659947

# 1. 起因

试想一下，你现在所在的公司有一个hadoop的集群。但是A项目组经常做一些定时的BI报表，B项目组则经常使用一些软件做一些临时需求。那么他们肯定会遇到同时提交任务的场景，这个时候到底如何分配资源满足这两个任务呢？是先执行A的任务，再执行B的任务，还是同时跑两个？

如果你存在上述的困惑，可以多了解一些yarn的资源调度器。

在Yarn框架中，调度器是一块很重要的内容。有了合适的调度规则，就可以保证多个应用可以在同一时间有条不紊的工作。最原始的调度规则就是FIFO，即按照用户提交任务的时间来决定哪个任务先执行，但是这样很可能一个大任务独占资源，其他的资源需要不断的等待。也可能一堆小任务占用资源，大任务一直无法得到适当的资源，造成饥饿。所以FIFO虽然很简单，但是并不能满足我们的需求。
# 2. 查看
## 2.1 web查看
`http://localhost:8088/cluster/nodes`

[![yarn web页面](https://images.weserv.nl/?url=files.catbox.moe/wjkxoe "yarn web页面")](https://ftp.bmp.ovh/imgs/2019/12/6254940f514c443b.png)

## 2.2 命令行查看某一个队列

```bash
lcc@lcc ~$ yarn queue -status default
18/12/15 10:55:10 INFO client.RMProxy: Connecting to ResourceManager at /0.0.0.0:8032
18/12/15 10:55:10 WARN util.NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Queue Information :
Queue Name : default
	State : RUNNING
	Capacity : 100.0%
	Current Capacity : .0%
	Maximum Capacity : 100.0%
	Default Node Label expression :
	Accessible Node Labels : *
```

# 3. 调度器的选择

在Yarn中有三种调度器可以选择：FIFO Scheduler ，Capacity Scheduler，FairS cheduler。

## 3.1 FIFO Scheduler

FIFO Scheduler把应用按提交的顺序排成一个队列，这是一个先进先出队列，在进行资源分配的时候，先给队列中最头上的应用进行分配资源，待最头上的应用需求满足后再给下一个分配，以此类推。

FIFO Scheduler是最简单也是最容易理解的调度器，也不需要任何配置，但它并不适用于共享集群。大的应用可能会占用所有集群资源，这就导致其它应用被阻塞。在共享集群中，更适合采用Capacity Scheduler或Fair Scheduler，这两个调度器都允许大任务和小任务在提交的同时获得一定的系统资源。

下面“Yarn调度器对比图”展示了这几个调度器的区别，从图中可以看出，在FIFO 调度器中，小任务会被大任务阻塞。

## 3.2 Capacity Scheduler

而对于Capacity调度器，有一个专门的队列用来运行小任务，但是为小任务专门设置一个队列会预先占用一定的集群资源，这就导致大任务的执行时间会落后于使用FIFO调度器时的时间。

## 3.3 FairS cheduler

在Fair调度器中，我们不需要预先占用一定的系统资源，Fair调度器会为所有运行的job动态的调整系统资源。如下图所示，当第一个大job提交时，只有这一个job在运行，此时它获得了所有集群资源；当第二个小任务提交后，Fair调度器会分配一半资源给这个小任务，让这两个任务公平的共享集群资源。

需要注意的是，在下图Fair调度器中，从第二个任务提交到获得资源会有一定的延迟，因为它需要等待第一个任务释放占用的Container。小任务执行完成之后也会释放自己占用的资源，大任务又获得了全部的系统资源。最终的效果就是Fair调度器即得到了高的资源利用率又能保证小任务及时完成。

[![](https://ftp.bmp.ovh/imgs/2019/12/a0b63037f16b30f4.png "yarn调度器")](https://ftp.bmp.ovh/imgs/2019/12/a0b63037f16b30f4.png)

# 4. capacity调度器详解
## 4.1 什么是capacity调度器

Capacity Schedule调度器以队列为单位划分资源。简单通俗点来说，就是一个个队列有独立的资源，队列的结构和资源是可以进行配置的，如下图：

[![](https://ftp.bmp.ovh/imgs/2019/12/8452c075f3cc77b6.png "队列结构")](https://ftp.bmp.ovh/imgs/2019/12/8452c075f3cc77b6.png)

default队列占30%资源，analyst和dev分别占40%和30%资源；类似的，analyst和dev各有两个子队列，子队列在父队列的基础上再分配资源。

队列以分层方式组织资源,设计了多层级别的资源限制条件以更好的让多用户共享一个Hadoop集群，比如队列资源限制、用户资源限制、用户应用程序数目限制。队列里的应用以FIFO方式调度，每个队列可设定一定比例的资源最低保证和使用上限，同时，每个用户也可以设定一定的资源使用上限以防止资源滥用。而当一个队列的资源有剩余时，可暂时将剩余资源共享给其他队列。

特性
Capacity调度器具有以下的几个特性：

1. 层次化的队列设计，这种层次化的队列设计保证子队列可以使用父队列设置的全部资源。这样通层次化的管理，更容易合理分配和限制资源的使。
2. 容量保证，队列上都会设置一个资源的占比，这可以保证每个队列都不会占用整个集群的资源。
3. 安全，每个队列又严格的访问控制。用户只能向己的队列里面提交任务，而且不能修改或者访问他队列的任务。
4. 弹性分配，空闲的资源可以被分配给任何队列。多个队列出现争用的时候，则会按照比例进行平。
5. 多租户租用，通过队列的容量限制，多个用户就以共享同一个集群，同时保证每个队列分配到自的容量，提高利用率。
6. 操作性，yarn支持动态修改调整容量、权限等的分配，可以在运行时直接修改。还提供给管理员界面，来显示当前的队列状况。管理员可以在运行时，添加一个队列；但是不能删除一个队列。管理员还可以在运行时暂停某个队列，这样可以保证当前的队列在执行过程中，集群不会接收其他的任务。如果一个队列被设置成了stopped，那么就不能向他或者子队列上提交任务了。

Capacity 调度器允许多个组织共享整个集群，每个组织可以获得集群的一部分计算能力。通过为每个组织分配专门的队列，然后再为每个队列分配一定的集群资源，这样整个集群就可以通过设置多个队列的方式给多个组织提供服务了。除此之外，队列内部又可以垂直划分，这样一个组织内部的多个成员就可以共享这个队列资源了，在一个队列内部，资源的调度是采用的是先进先出(FIFO)策略。

通过上面那幅图，我们已经知道一个job可能使用不了整个队列的资源。然而如果这个队列中运行多个job，如果这个队列的资源够用，那么就分配给这些job，如果这个队列的资源不够用了呢？其实Capacity调度器仍可能分配额外的资源给这个队列，这就是“弹性队列”(queue elasticity)的概念。

在正常的操作中，Capacity调度器不会强制释放Container，当一个队列资源不够用时，这个队列只能获得其它队列释放后的Container资源。当然，我们可以为队列设置一个最大资源使用量，以免这个队列过多的占用空闲资源，导致其它队列无法使用这些空闲资源，这就是”弹性队列”需要权衡的地方。

Capacity调度器说的通俗点，可以理解成一个个的资源队列。这个资源队列是用户自己去分配的。比如我大体上把整个集群分成了AB两个队列，A队列给A项目组的人来使用。B队列给B项目组来使用。但是A项目组下面又有两个方向，那么还可以继续分，比如专门做BI的和做实时分析的。那么队列的分配就可以参考下面的树形结构

```bash
root
------a[60%]
      |---a.bi[40%]
      |---a.realtime[60%]
------b[40%]
```

a队列占用整个资源的60%，b队列占用整个资源的40%。a队列里面又分了两个子队列，一样也是2:3分配。

虽然有了这样的资源分配，但是并不是说a提交了任务，它就只能使用60%的资源，那40%就空闲着。只要资源实在空闲状态，那么a就可以使用100%的资源。但是一旦b提交了任务，a就需要在释放资源后，把资源还给b队列，直到ab平衡在3:2的比例。

粗粒度上资源是按照上面的方式进行，在每个队列的内部，还是按照FIFO的原则来分配资源的。

## 4.2 特性

capacity调度器具有以下的几个特性：

1. 层次化的队列设计，这种层次化的队列设计保证了子队列可以使用父队列设置的全部资源。这样通过层次化的管理，更容易合理分配和限制资源的使用。
2. 容量保证，队列上都会设置一个资源的占比，这样可以保证每个队列都不会占用整个集群的资源。
3. 安全，每个队列又严格的访问控制。用户只能向自己的队列里面提交任务，而且不能修改或者访问其他队列的任务。
4. 弹性分配，空闲的资源可以被分配给任何队列。当多个队列出现争用的时候，则会按照比例进行平衡。
5. 多租户租用，通过队列的容量限制，多个用户就可以共享同一个集群，同事保证每个队列分配到自己的容量，提高利用率。
6. 操作性，yarn支持动态修改调整容量、权限等的分配，可以在运行时直接修改。还提供给管理员界面，来显示当前的队列状况。管理员可以在运行时，添加一个队列；但是不能删除一个队列。管理员还可以在运行时暂停某个队列，这样可以保证当前的队列在执行过程中，集群不会接收其他的任务。如果一个队列被设置成了stopped，那么就不能向他或者子队列上提交任务了。
7. 基于资源的调度，协调不同资源需求的应用程序，比如内存、CPU、磁盘等等。

## 4.3 调度器的配置
### 4.3.1 配置调度器

在ResourceManager中配置它要使用的调度器，hadoop资源分配的默认配置

在搭建完成后我们发现对于资源分配方面，yarn的默认配置是这样的,也就是有一个默认的队列
事实上，是否使用CapacityScheduler组件是可以配置的，但是默认配置就是这个CapacityScheduler，如果想显式配置需要修改 conf/yarn-site.xml 内容如下：

```xml
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

可以看到默认是`org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler`这个调度器，那么这个调度器的名字是什么呢？

我们可以在/user/local/hadoop/hadoop-2.7.4/etc/hadoop/capacity-scheduler.xml文件中看到

```xml
<property>
    <name>yarn.scheduler.capacity.root.queues</name>
    <value>default</value>
    <description>
      The queues at the this level (root is the root queue).
    </description>
  </property>
```

可以看到默认的队列名字为**default**(知道名字有什么用？我们可以根据nameNode地址，和调度器名称，获取机器相关的信息，比如内存，磁盘，cpu等资源用了多少，还剩下多少)
### 3.3.2 配置队列

调度器的核心就是队列的分配和使用了，修改`conf/capacity-scheduler.xml`可以配置队列。

Capacity调度器默认有一个预定义的队列-root,所有的队列都是它的子队列。队列的分配支持层次化的配置，使用.来进行分割，比如`yarn.scheduler.capacity.<queue-path>.queues.`

下面是配置的样例，比如root下面有三个子队列:

```xml
<property>
  <name>yarn.scheduler.capacity.root.queues</name>
  <value>a,b,c</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.a.queues</name>
  <value>a1,a2</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>

<property>
  <name>yarn.scheduler.capacity.root.b.queues</name>
  <value>b1,b2,b3</value>
  <description>The queues at the this level (root is the root queue).
  </description>
</property>
```
上面配置的结构类似于

```bash
root
------a[队列名字]
      |---a1[子队列名字]
      |---a2[子队列名字]
------b[队列名字]
	  |---b1[子队列名字]
      |---b2[子队列名字]
      |---b3[子队列名字]
------c[队列名字]
```

### 4.3.3 队列属性

1. `yarn.scheduler.capacity..capacity`
它是队列的资源容量占比(百分比)。系统繁忙时，每个队列都应该得到设置的量的资源；当系统空闲时，该队列的资源则可以被其他的队列使用。同一层的所有队列加起来必须是100%。
2. `yarn.scheduler.capacity..maximum-capacity`
队列资源的使用上限。由于系统空闲时，队列可以使用其他的空闲资源，因此最多使用的资源量则是该参数控制。默认是-1，即禁用。
3. `yarn.scheduler.capacity..minimum-user-limit-percent`
每个任务占用的最少资源。比如，你设置成了25%。那么如果有两个用户提交任务，那么每个任务资源不超过50%。如果3个用户提交任务，那么每个任务资源不超过33%。如果4个用户提交任务，那么每个任务资源不超过25%。如果5个用户提交任务，那么第五个用户需要等待才能提交。默认是100，即不去做限制。
4. `yarn.scheduler.capacity..user-limit-factor`
每个用户最多使用的队列资源占比，如果设置为50.那么每个用户使用的资源最多就是50%。

### 4.3.4 运行和提交应用限制

1. `yarn.scheduler.capacity.maximum-applications / yarn.scheduler.capacity..maximum-applications`
设置系统中可以同时运行和等待的应用数量。默认是10000.
2. `yarn.scheduler.capacity.maximum-am-resource-percent / yarn.scheduler.capacity..maximum-am-resource-percent`
设置有多少资源可以用来运行app master，即控制当前激活状态的应用。默认是10%。

### 3.3.5 队列管理

1. `yarn.scheduler.capacity..state`
队列的状态，可以使RUNNING或者STOPPED.如果队列是STOPPED状态，那么新应用不会提交到该队列或者子队列。同样，如果root被设置成STOPPED，那么整个集群都不能提交任务了。现有的应用可以等待完成，因此队列可以优雅的退出关闭。
2. `yarn.scheduler.capacity.root..acl_submit_applications`
访问控制列表ACL控制谁可以向该队列提交任务。如果一个用户可以向该队列提交，那么也可以提交任务到它的子队列。
3. `yarn.scheduler.capacity.root..acl_administe_queue`
设置队列的管理员的ACL控制，管理员可以控制队列的所有应用程序。同样，它也具有继承性。

注意：ACL的设置是`user1,user2 group1,group2`这种格式。如果是`*`则代表任何人。`空格`表示任何人都不允许。默认是`*.`

### 4.3.6 其他属性

1. `yarn.scheduler.capacity.resource-calculator`
资源计算方法，默认是`org.apache.hadoop.yarn.util.resource.DefaultResourseCalculator`,它只会计算内存。`DominantResourceCalculator`则会计算内存和CPU。
2. `yarn.scheduler.capacity.node-locality-delay`
调度器尝试进行调度的次数。一般都是跟集群的节点数量有关。默认40(一个机架上的节点数)

一旦设置完这些队列属性，就可以在web ui上看到了。可以访问下面的连接：

### 4.3.7 web接口
`http://localhost:8088/ws/v1/cluster/scheduler`

```xml
<?xml version="1.0" encoding="utf-8"?>

<scheduler> 
  <schedulerInfo xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:type="capacityScheduler">  
    <capacity>100.0</capacity>  
    <usedCapacity>0.0</usedCapacity>  
    <maxCapacity>100.0</maxCapacity>  
    <queueName>root</queueName>  
    <queues> 
      <queue xsi:type="capacitySchedulerLeafQueueInfo"> 
        <capacity>100.0</capacity>  
        <usedCapacity>0.0</usedCapacity>  
        <maxCapacity>100.0</maxCapacity>  
        <absoluteCapacity>100.0</absoluteCapacity>  
        <absoluteMaxCapacity>100.0</absoluteMaxCapacity>  
        <absoluteUsedCapacity>0.0</absoluteUsedCapacity>  
        <numApplications>0</numApplications>  
        <queueName>default</queueName>  
        <state>RUNNING</state>  
        <resourcesUsed> 
          <memory>0</memory>  
          <vCores>0</vCores> 
        </resourcesUsed>  
        <hideReservationQueues>false</hideReservationQueues>  
        <nodeLabels>*</nodeLabels>  
        <numActiveApplications>0</numActiveApplications>  
        <numPendingApplications>0</numPendingApplications>  
        <numContainers>0</numContainers>  
        <maxApplications>10000</maxApplications>  
        <maxApplicationsPerUser>10000</maxApplicationsPerUser>  
        <userLimit>100</userLimit>  
        <users/>  
        <userLimitFactor>1.0</userLimitFactor>  
        <AMResourceLimit> 
          <memory>1024</memory>  
          <vCores>1</vCores> 
        </AMResourceLimit>  
        <usedAMResource> 
          <memory>0</memory>  
          <vCores>0</vCores> 
        </usedAMResource>  
        <userAMResourceLimit> 
          <memory>1024</memory>  
          <vCores>1</vCores> 
        </userAMResourceLimit>  
        <preemptionDisabled>true</preemptionDisabled> 
      </queue> 
    </queues> 
  </schedulerInfo> 
</scheduler>
```

这里`<queueName>default</queueName>` 队列名称很重要，我们可以根据它获取相关信息，自己写调度器

### 4.3.8 修改队列配置

如果想要修改队列或者调度器的配置，可以修改

```bash
vi $HADOOP_CONF_DIR/capacity-scheduler.xml
```

修改完成后，需要执行下面的命令：

```bash
$HADOOP_YARN_HOME/bin/yarn rmadmin -refreshQueues
```
### 4.3.9 注意

1. 队列不能被删除，只能新增。
2. 更新队列的配置需要是有效的值
3. 同层级的队列容量限制想加需要等于100%。

# 5. 队列实例

[![](https://ftp.bmp.ovh/imgs/2019/12/13cc84e914334d0e.png "队列总资源")](https://ftp.bmp.ovh/imgs/2019/12/13cc84e914334d0e.png)

[![](https://ftp.bmp.ovh/imgs/2019/12/dfbdbbb92c3230fe.png "yarn web页面 队列")](https://ftp.bmp.ovh/imgs/2019/12/dfbdbbb92c3230fe.png)