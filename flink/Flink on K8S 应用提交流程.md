[TOC]

来源：https://mp.weixin.qq.com/s/j1b2goTGfT5Z5lk6Q1lDMw

# 1. Flink on docker
Pod 是 K8S 集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于 Pod 中。

一个 Pod 中可以存在一个或者多个容器，容器在创建时，会先新建 containerd-shim，containerd-shim 会建出来最终的 docker 容器。所以 Flink 要被部署到 K8S集群上，需要先被加载到 Docker 镜像中。

## 1.1 Docker 启动 Flink 方式
本篇文章，我们使用 standalone的方式，详细讲解如何在 docker 和 K8S 中部署Flink 集群。

可以回顾下 Flink 的启动方式。在传统的物理机或者虚拟机中部署 Flink，如果配置了免密和 slave 节点信息，我们可以使用 Flink 的 `start-cluster.sh`，一条命令启动整个集群。

但是在 docker 中，我们无法这样操作。在镜像中通常不会安装 ssh 服务。此外，docker 镜像的主进程要求使用前台的方式运行，否则该 container 会运行完毕退出。

所以 在 Docker 中需要单独启动 Flink job manager 和 task manager。

## 1.2 修改 Flink 配置
在 mini1 master 主节点根目录新建 lyz 文件夹。

1. 因为 Flink Standalone 集群依赖 JDK 就可以，所以将 flink-1.14.0-bin-
scala_2.11.tgz,jdk‐8u202‐linux‐x64.tar.gz 安装包下载好，放入 lyz 目录下，如下截图：

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6FSj0WzfZptbcUqicBaKQEwib3ol6cmArUotaCtc6WPUp4BWlsBXMRsYg/)](https://z3.ax1x.com/2021/11/04/IZFK4f.png)

2. 解压 flink-1.14.0-bin-scala_2.11.tgz,修改 conf 目录的配置文件：

```properties
#1. 配置 jobmanager rpc 地址
jobmanager.rpc.address: 192.168.244.

#2. 修改 taskmanager 内存大小，可改可不改
taskmanager.memory.process.size: 2048 m

#3. 修改一个 taskmanager 中对于的 taskslot 个数，可改可不改
taskmanager.numberOfTaskSlots: 4

修改并行度，可改可不改
parallelism.default: 4
```

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6gcv5lPeicoy9vGgFGcjGM775vc1sJUhCcsrGrvrL7Myjolak1K7Lia0A/)](https://z3.ax1x.com/2021/11/04/IZEVu4.png)

3. 配置 master
    
```
#修改主节点ip地址  
192.168.244.131:8081  
```

4. 配置 work
    

```
#修改从节点ip，因为是standalone，所以主从一样  
192.168.244.131  
```

5. 将修改好的 flink 文件 重新打成 flink-1.14.0-bin-scala\_2.11.tgz 压缩包
    

```
tar -zcvf flink-1.14.0-bin-scala_2.11.tar.gz  flink-1.14.0/
```

6. 删除 解压后的 flink 文件夹，最终截图如下：
    
[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6FSj0WzfZptbcUqicBaKQEwib3ol6cmArUotaCtc6WPUp4BWlsBXMRsYg/)](https://z3.ax1x.com/2021/11/04/IZFK4f.png)

## 1.3 Flink Dockerfile 镜像制作

由于新打的镜像不包含 Linux 的基础命令，所有我们需要使用 **centos7** 作为基础镜像。

1. 在 lyz 文件夹下 编写 Dockerfile
    

```bash
# Dockerfile 
[root@mini1 lyz]# vi Dockerfile

FROM centos:7
MAINTAINER lyz
ADD flink-1.14.0-bin-scala_2.11.tar.gz  /root
ADD jdk‐8u202‐linux‐x64.tar.gz  /root
ARG FLINK_VERSION=1.14.0
ARG SCALA_VERSION=2.11
ENV FLINK_HOME=/root/flink-1.14.0
ENV PATH=$FLINK_HOME/bin:$PATH
ENV JAVA_HOME=/root/jdk1.8.0_202
ENV PATH=$JAVA_HOME/bin:$PATH

RUN cd $FLINK_HOME  \
       && echo "env.java.home: /root/jdk1.8.0_202" >> $FLINK_HOME/conf/flink-conf.yaml \
       && echo "FLINK_HOME=/root/flink-1.14.0" >> /etc/profile \
       && echo "PATH=$FLINK_HOME/bin:$PATH" >> /etc/profile \
       && echo "JAVA_HOME=/root/jdk1.8.0_202" >> /etc/profile \
       && echo "PATH=$JAVA_HOME/bin:$PATH" >> /etc/profile \
       && source /etc/profile

EXPOSE 8081
```


上述文件将 JDK 和 Flink1.14.0的安装包加载到 Docker的根目录下，同时会在 Profile 中添加 JDK 和 Flink 的环境变量。

2. 使用 Dockerfile 文件将压缩包打成镜像放入 docker 中

`docker build -t flink:1.14.0 . -f Dockerfile`

如下图所示，已经打包成功

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6nEXvb81kMaNhByfBU1bmLCbTmps1ZrK031AlEAA4nlxtvEKXe8Re8Q/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZEAvF.png)

3. 通过 docker images 查看 docker 中的镜像，发现 flink 镜像已被加载成功。

`docker images`

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y60yrl811VEB2gnA29U2fwqJYGISvibibQRnFnMW7NMI5NRJFqcq6pACUQ/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZEZDJ.png)

## 1.4 Flink 集群启动

通过 docker run 命令，使用外部端口 9999 映射容器中 flink 集群的 8081 端口，将端口暴露在外边。

`docker run -it -p 9999:8081 flink:1.14.0 /bin/bash`

进入容器后，可以看到 jdk 和 flink 包全部存放在/root 根目录下。

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6Rb8qxMsXllHphiaSaqh9rrnNDo9XhuEUewQfDS1j0Kc0icmAQHvSg1Ow/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZejGq.png)

1. 刷新环境配置

`source /etc/profile`

2. 启动 jobmanager、taskmanager


```bash
[root@6437e358ba1d ~]# cd flink-1.14.0/bin
[root@6437e358ba1d bin]# ./jobmanager.sh start

Starting standalonesession daemon on host 6437e358ba1d.

[root@6437e358ba1d bin]# ./taskmanager.sh start

Starting taskexecutor daemon on host 6437e358ba1d.

[root@6437e358ba1d bin]# jps
672 TaskManagerRunner
712 Jps
334 StandaloneSessionClusterEntrypoint
```

启动结果如下图所示：

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y667VETLDiaI95jCT8SaKSur6EWGbgPWUAIcea6yTI8DEsdQrYWGpNbLw/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZexzV.png)

3. 使用浏览器打开 flink 的管理页面 192.168.244.131:9999。
    

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6HhalKUZTvFAsjKCk0YIC2tbHJYWyfZuibxakKjQTOIrluRMKGYxGz4A/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZevR0.png)

## 1.5 Flink on docker 提交任务


使用 Flink 自带 WordCount.jar 提交应用任务，并输入外部映射的IP 和 端口。

```bash
[root@e7732776891e bin]# flink run -m 192.168.244.131:9999 \
../examples/batch/WordCount.jar --input aaa.txt --output bbb.txt/
```


[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6xFHZhZ8drBUr9bVHR3bibFyzA3ibSms2Jvw877GejRRFB0VDBzGfuNdA/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZeXin.png)

## 2. Flink on k8s
## 2.1 Flink on k8s 流程分析

上述 Flink on Docker 跑通后，下面我们重点研究一下 Flink on K8S Session 的提交流程、安装部署、应用提交案例。

Flink On K8s Session 模式执行原理图如下：

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6wF1qORZRtiazYTdMao7d1vfuK5BwbQXiajoxWI4a9WKTC03kDFjl3usw/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZmSMT.png)

在 K8S 集群上使用 Session 模式提交 Flink 作业的过程会分为 3 个阶段，首先在 K8S 上启动 Flink Session 集群；其次通过 Flink Client 提交作业；最后进行作业调度执行。具体步骤如下：

**1 启动集群**

1. Flink 客户端使用 Kubectl 或者 K8s 的 Dashboard 提交 Flink 集群的资源描述文件，包含:
    

> flink-configuration-configmap.yaml
> 
> jobmanager-service.yaml
> 
> jobmanager-rest-service.yaml #可选
> 
> jobmanager-deployment.yaml
> 
> taskmanager-deployment.yaml

通过这些文件请求 K8s Master。

2. K8s Master 根据这些资源文件将请求分发给 Slave 节点，创建 Flink Master Deployment、TaskManager Deployment、ConfigMap、SVC 四个角色。同时初始化 Dispatcher 和 KubernetesResourceManager。并通过 K8S 服务对外暴露 Flink Master 端口。

以上两个步骤执行完后，Session模式集群创建成功。但此时还没有 JobMaster和 TaskManager,当提交作业需要执行时，才会按需创建。

**2 作业提交**

3. Client 用户使用 Flink run 命令，通过指定 Flink Master 的地址，将相应任务提交上来，用户的 Jar 和 JobGrapth 会在 Flink Client 生成，通过 SVC 传给 Dispatcher。
    
4. Dispatcher 收到 JobGraph后，会为每个作业启动一个JobMaster,将JobGraph 交给JobMaster进行调度。

**3 作业调度**

5. JobMaster 会向 KubernetesResourceManager 申请资源，请求Slot。
    
6. KubernetesResourceManager 从 K8S 集群分配 TaskManager。每个 TaskManager 都是具有唯一标识的 Pod。KubernetesResourceManager 会为 TaskManager 生成一份新的配置文件，里面有 Flink Master 的 service name 作为地址，保障在 Flink Master failover 后，TaskManager 仍然可以重新连接上。
    
7. K8S 集群分配一个新的 Pod 后，在上面启动 TaskManager。
    
8. TaskManager 启动后注册到 SlotManager。
    
9. SlotManager 向 TaskManager 请求 Slot。
    
10. TaskManager 提供 Slot 给 JobManager,然后任务被分配到 Slot 上运行。
    

## 2.2 Flink on k8s Session 集群部署

通过上述流程图，我们已经详细了解了 Flink on K8S Session 模式的提交流程，接下来根据提交流程完成集群部署。

**本文采用三节点进行部署，mini1 代表 Master 节点，mini2、mini3 代表 node 节点**。

总共需要在 mini1 节点创建 5 个文件，其中一个为可选项

|角色|IP|地址组件|
|---|---|---|
|mini1|192.168.244.131|docker，kubectl，kubeadm，kubelet|
|mini2|192.168.244.132|docker，kubectl，kubeadm，kubelet|
|mini3|192.168.244.133|docker，kubectl，kubeadm，kubelet|


截图如下：

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6RV3ytIHSbxG5uEoO20zPOdtyiaWxcSPBWSnAIY0omFib72vGvQfgkvgQ/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZmpsU.png)

### 2.2.1 创建 ConfigMap

ConfigMap 是一个 K-V 数据结构。通常的用法是将 ConfigMap 挂载到 Pod ，作为配置文件提供 Pod 里新的进程使用。在 Flink 中可以将 Log4j 文件或者是 flink-conf 文件写到 ConfigMap 里面，在 JobManager 或者 TaskManger 起来之前将它挂载到 Pod 里，然后 JobManager 去读取相应的 conf 文件，加载其配置，进而再正确地拉起 JobManager 一些相应的组件。

```yaml
# 新建根目录 k8s_flink
[root@mini1 ~]# mkdir k8s_flink
# 进入根目录
[root@mini1 ~]# cd k8s_flink/
# 新建 flink-configuration-configmap.yaml
[root@mini1 k8s_flink]# vim flink-configuration-configmap.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: flink-config
  namespace: lyz
  labels:
    app: flink
data:
  flink-conf.yaml: |+
    jobmanager.rpc.address: 192.168.244.131
    taskmanager.numberOfTaskSlots: 4
    blob.server.port: 6124
    jobmanager.rpc.port: 6123
    taskmanager.rpc.port: 6122
    queryable-state.proxy.ports: 6125
    jobmanager.memory.process.size: 1600m
    taskmanager.memory.process.size: 2048m
    taskmanager.numberOfTaskSlots: 4
    parallelism.default: 4
```


### 2.2.2 创建 Service

通过Serveice 对外暴露服务。Service 可以看作是一组同类 Pod 对外的访问接口。借助 Service，应用可以方便地实现服务发现和负载均衡。


```yaml
# 创建 jobmanager-service.yaml
[root@mini1 k8s_flink]# vim jobmanager-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager
  namespace: lyz
spec:
  type: ClusterIP
  ports:
  - name: rpc
    port: 6123
  - name: blob-server
    port: 6124
  - name: webui
    port: 8081
  selector:
    app: flink
    component: jobmanager
```

### 2.2.3 创建 rest Service(可选)

可选的 service，该 service 将 jobmanager 的 rest 端口暴露为公共 Kubernetes node 的节点端口

```yaml
# 创建 jobmanager-rest-service.yaml
[root@mini1 k8s_flink]# vim jobmanager-rest-service.yaml

apiVersion: v1
kind: Service
metadata:
  name: flink-jobmanager-rest
  namespace: lyz
spec:
  type: NodePort
  ports:
  - name: rest
    port: 8081
    targetPort: 8081
    nodePort: 30081
  selector:
    app: flink
    component: jobmanager
```

### 2.2.4 创建 jobmanager session deployment

创建 jobmanager 运行的 pod 控制器

```yaml
# 创建 jobmanager-session-deployment.yaml
[root@mini1 k8s_flink]# vim jobmanager-session-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-jobmanager
  namespace: lyz
spec:
  replicas: 1
  selector:
    matchLabels:
      app: flink
      component: jobmanager
  template:
    metadata:
      labels:
        app: flink
        component: jobmanager
    spec:
      containers:
      - name: jobmanager
        image: flink:1.14.0
        args: ["jobmanager"]
        ports:
        - containerPort: 6123
          name: rpc
        - containerPort: 6124
          name: blob-server
        - containerPort: 8081
          name: webui
        livenessProbe:
          tcpSocket:
            port: 6123
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf
        securityContext:
          runAsUser: 9999  
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
```

### 2.2.5 创建 taskmanager session deployment

创建 taskmanager 运行的 pod 控制器

```yaml
# 创建 taskmanager-session-deployment.yaml
[root@mini1 k8s_flink]# vim taskmanager-session-deployment.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: flink-taskmanager
  namespace: lyz
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flink
      component: taskmanager
  template:
    metadata:
      labels:
        app: flink
        component: taskmanager
    spec:
      containers:
      - name: taskmanager
        image: flink:1.14.0
        args: ["taskmanager"]
        ports:
        - containerPort: 6122
          name: rpc
        - containerPort: 6125
          name: query-state
        livenessProbe:
          tcpSocket:
            port: 6122
          initialDelaySeconds: 30
          periodSeconds: 60
        volumeMounts:
        - name: flink-config-volume
          mountPath: /opt/flink/conf/
        securityContext:
          runAsUser: 9999  
      volumes:
      - name: flink-config-volume
        configMap:
          name: flink-config
          items:
          - key: flink-conf.yaml
            path: flink-conf.yaml
```

## 2.3 Flink on k8s Session 集群启动


1. 分别执行下面创建的五个文件:

```bash
# Configuration 和 service 的定义
[root@mini1 k8s_flink]# kubectl create -f flink-configuration-configmap.yaml
[root@mini1 k8s_flink]# kubectl create -f jobmanager-service.yaml
[root@mini1 k8s_flink]# kubectl create -f jobmanager-rest-service.yaml

# 为集群创建 deployment
[root@mini1 k8s_flink]# kubectl create -f jobmanager-session-deployment.yaml
[root@mini1 k8s_flink]# kubectl create -f taskmanager-session-deployment.yaml
```


[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6MCUV7eiaiaw1uOT4lPsS5BdcnW247lve0ZOQd9Fz3G5ZOibzibIPqeXWNQ/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZm9LF.png)

2. 查看 svc,pod 服务是否启动成功

`kubectl get svc,pods -n lyz -o wide`

通过截图可以看到Flink on K8S Session 集群已经全部启动成功，启动 2个 taskmanager 启动在 mini2,mini3 服务器上。jobmanager 启动在mini3 服务器上。对外暴露的端口号是30081。

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6yFVVPKprjd2sGoLucoGFqCQV6n7tb1ySDibusx1IDXibeGLDjBrDmJKQ/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZmPZ4.png)

3. 通过 DashBoard Web 服务查看启动详情，截图如下：

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6lQI56xqwicpiaP2uJReicZJCFY69hNW93jyX0VQHFaHXcHmYpQkWL0icvQ/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZMJAg.png)

通过 DashBoard Web 可以看到服务都正常启动，登录 Flink WEB 页面查看详情 192.168.244.131:30081

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6ZnCroRTCia9Zr18tHSxLjibo2LQ9W9AlM7yicbgcicwn8YouNpDHAPSIJw/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZM8HS.png)

## 2.4 Flink on k8s 应用提交

通过Flink 官方提供的命令进行应用提交

```bash
./bin/flink run -m 192.168.244.131:8081 ./examples/batch/WordCount.jar --input aaa.txt --output bbb.txt/
```

[![](https://mmbiz.qpic.cn/mmbiz_png/rLGOIHABwEoiaKuMEy7rCkEZo2icibsH8Y6xFHZhZ8drBUr9bVHR3bibFyzA3ibSms2Jvw877GejRRFB0VDBzGfuNdA/640?wx_fmt=png)](https://z3.ax1x.com/2021/11/04/IZM3B8.png)