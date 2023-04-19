[TOC]
# 1. Akka
Akka （Scala 编写的库）

Akka是JAVA虚拟机JVM平台上构建高并发、分布式和容错应用的工具包和运行时。Akka用Scala语言写成，同时提供了Scala和JAVA的开发接口。

Akka处理并发的方法基于Actor模型。在Akka里，Actor之间通信的唯一机制就是消息传递。
# 2. 偏函数
[![Q19gdU.png](https://s2.ax1x.com/2019/12/04/Q19gdU.png "偏函数")](https://i0.wp.com/i.loli.net/2019/12/04/sPfE1OHAWkjJCua.png)

被包在花括号内没有match的一组case语句是一个偏函数，它是PartialFunction[A, B]的一个实例，A代表参数类型，B代表返回类型，常用作输入模式匹配

A、B前不用加var、val


```scala
object PartialFuncDemo  {

  def func1: PartialFunction[String, Int] = {
    case "one" => 1
    case "two" => 2
    case _ => -1
  }

  def func2(num: String) : Int = num match {
    case "one" => 1
    case "two" => 2
    case _ => -1
  }

  def main(args: Array[String]) {
    println(func1("one"))
    println(func2("one"))
  }
}
```

# 3. RPC
## 3.1 Master

```scala
package cn.edu360.rpc

import akka.actor.{Actor, ActorSystem, Props}
import com.typesafe.config.ConfigFactory

import scala.collection.mutable
import scala.concurrent.duration._

class Master extends Actor {
  var CHECK_INTERVAL = 15000

  //用来检测超时Worker
  override def preStart(): Unit = {
    import context.dispatcher
    context.system.scheduler.schedule(0 millis, CHECK_INTERVAL millis, self, CheckTimeOutWorker)
  }

  val id2Workers = new mutable.HashMap[String, WorkerInfo]()

  /**
    * 接收消息
    *
    * @return
    */
  override def receive: Receive = {
    //Worker发送给Master的注册信息
    case RegisterWorker(workerId, memory, cores) => {
      //println(s"Id:$workerId,memory:$memory,cores:$cores")
      //将Worker发过来的消息，先封装，再保存起来
      val workerInfo = new WorkerInfo(workerId, memory, cores)
      if (!id2Workers.contains(workerId)) {
        id2Workers(workerId) = workerInfo
        //Master已经将Worker的信息保存起来的，然后向Worker反馈一个注册成功的消息
        sender() ! RegisteredWorker
      }
    }

    //worker->Master的心跳信息，报活
    case Heartbeat(workerId) => {
      val workerInfo = id2Workers(workerId)
      val currentTime = System.currentTimeMillis()
      workerInfo.lastHeartbeatTime = currentTime
      println(workerInfo)
    }

    case CheckTimeOutWorker => {
      val workers = id2Workers.values
      val currentTime = System.currentTimeMillis()
      val deadWorkers: Iterable[WorkerInfo] = workers.filter(w => currentTime - w.lastHeartbeatTime > CHECK_INTERVAL)
      deadWorkers.foreach(
        dw => {
          id2Workers -= dw.workerId
        }
      )

      println(s"current alive worker num is ${workers.size}")
    }


  }
}

object Master {
  val MASTER_SYSTEM = "MasterSystem"
  val MASTER_NAME = "Master"


  def main(args: Array[String]): Unit = {

    val host = args(0)
    val port = args(1).toInt

    //用ActorSystem创建实例，是单例即可
    val confStr =
      s"""
         |akka.actor.provider = "akka.remote.RemoteActorRefProvider"
         |akka.remote.netty.tcp.hostname = $host
         |akka.remote.netty.tcp.port = $port
      """.stripMargin
    val config = ConfigFactory.parseString(confStr)
    //创建ActorSystem，可以创建并管理Actor
    val actorSystem = ActorSystem(MASTER_SYSTEM, config)
    //利用ActorSystem创建Actor
    val actor = actorSystem.actorOf(Props[Master], MASTER_NAME)
    //actor ! "hello"
  }
}
```

## 3.2 Workers

1. Worker给Master发送心跳，直接发不行，拿不到ref（主叫是Worker，主叫拿不到ref），可以自己给自己发消息，再给Master发
2. 使用定时器，必须**导入隐式转换**
    1) import context.dispatcher
    2) context.system.scheduler.schedule(0 millis,10000 millis,self,SendHeartbeat)

```scala
package cn.edu360.rpc

import java.util.UUID

import akka.actor.{Actor, ActorSelection, ActorSystem, Props}
import com.typesafe.config.ConfigFactory

import scala.concurrent.duration._

class Worker(var masterHost: String, var masterPort: Int, var memory: Int, var cores: Int) extends Actor {
  //Worker创建好后，要跟Master建立连接

  val workerId = UUID.randomUUID().toString
 var masterRef: ActorSelection = _


  override def preStart(): Unit = {
    //Worker向Master建立连接
    masterRef = context.actorSelection(s"akka.tcp://${Master.MASTER_SYSTEM}@$masterHost:$masterPort/user/${Master.MASTER_NAME}")
    //向Master发消息
    masterRef ! RegisterWorker(workerId, memory, cores)
  }


  override def receive: Receive = {
    case RegisteredWorker => {
      //启动一个定时器，定期向Master发送心跳消息
      //导入隐式转换
      import context.dispatcher
    context.system.scheduler.schedule(0 millis,10000 millis,self,SendHeartbeat)
    }

    //自己给自己发送的周期性消息
    case SendHeartbeat => {
      //可以做一些判断，判断Worker的状态是否连接...
      masterRef ! Heartbeat(workerId)
    }
  }
}

object Worker {

  val WORKER_SYSTEM = "WorkerSystem"
  val WORKER_NAME = "Worker"

  def main(args: Array[String]): Unit = {

    //master args
    val masterHost = args(0)
    var masterPort = args(1).toInt

    //worker args
    val workerHost = args(2)
    //var workerPort = args(3)
    val workerMemory = args(3).toInt
    var workerCores = args(4).toInt


    val confStr =
      """
        |akka.actor.provider = "akka.remote.RemoteActorRefProvider"
        |akka.remote.netty.tcp.hostname = localhost
      """.stripMargin
    val config = ConfigFactory.parseString(confStr)
    //创建ActorSystem，用来创建并管理Actor
    val actorSystem = ActorSystem(WORKER_SYSTEM, config)
    actorSystem.actorOf(Props(new Worker(masterHost, masterPort, workerMemory, workerCores)),WORKER_NAME)
  }
}
```

## 3.3 WorkersInfo

```scala
package cn.edu360.rpc

class WorkerInfo (val workerId:String,val memory:Int,val cores:Int){
var lastHeartbeatTime:Long= _
}
```

## 3.4 Message

```scala
package cn.edu360.rpc

//注册消息 worker -> Master
case class RegisterWorker(workerId: String, memory: Int, cores: Int)

  //注册成功消息 master -> worker
  case object RegisteredWorker

  //心跳消息 worker -> Master
  case class Heartbeat(workerId: String)

  //worker自己给自己发消息
  case object SendHeartbeat

  //Master发给自己的消息
  case object CheckTimeOutWorker
```
