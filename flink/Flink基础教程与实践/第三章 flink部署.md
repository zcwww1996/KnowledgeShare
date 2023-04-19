[TOC]

# 1. Standalone 模式

## 1.1 安装

解压缩 flink-1.7.2-bin-hadoop27-scala_2.11.tgz，进入 conf 目录中。

### 1.1.1 修改 flink/conf/flink-conf.yaml 文件:
修改为主机名：hadoop1<br/>
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581565609544-06f826a2-6ea9-49d0-a3b5-56a2f36ca9c0.png)


### 1.1.2 修改 /conf/slave 文件:
在slave 文件里添加slave节点<br/>
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581565674350-d74e72c4-93af-4331-8c1f-77e222c7265c.png)

### 1.1.3 分发给另外两台机子:
将修改好的flink 分发到另外两个节点上

```bash
[bigdatachadoop1conf] xsyncflink-1.7.0
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581565734998-13f0c200-18a5-4450-8871-81a7ebd082b5.png)

### 1.1.4 启动
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581565804464-caffa395-7ec2-4537-a457-1fb0a507ed56.png)

访问http://localhost:8081可以对 flink 集群和任务进行监控管理。

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581565843792-db81840d-5c0c-4cb1-8e5f-5cb2d231d203.png)

## 1.2 提交任务

### 1.2.1 准备数据文件
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566003347-5b67a45d-4e18-4df6-8b59-ba9273cea0c0.png)

### 1.2.2 分发数据到taskmanage 机器中
把含数据文件的文件夹，分发到 taskmanage 机器中

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566060613-ec83cc5d-375a-418d-bafe-a0e9fa9049eb.png)<br/>

由于读取数据是从本地磁盘读取，实际任务会被分发到 taskmanage 的机器中，所以要把目标文件分发。

### 1.2.3 执行程序

```bash
./flink run -c com.atguigu.flink.app.BatchWcApp 
/ext/flinkTest-1.0-SNAPSHOT.jar 
--input /applog/flink/input.txt
--output /applog/flink/output.csv
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566141198-0ba90fa9-2555-4f0f-9cd1-222088c2a2e5.png)

### 1.2.4 到目标文件夹中查看计算结果
注意:计算结果根据会保存到 taskmanage 的机器下，不会在 jobmanage 下。<br>
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566179143-4c1f4c0b-473d-4a59-89d8-2605676528e1.png)

### 1.2.5 在 webui 控制台查看计算过程
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566215219-15b08419-a3e0-4c51-b223-402458fff583.png)


## 3.2  Yarn 模式
以 Yarn 模式部署 Flink 任务时，要求 Flink 是有 Hadoop 支持的版本，Hadoop环境需要保证版本在 2.2 以上，并且集群中安装有 HDFS 服务。

### 3.2.1 启动hadoop集群(略)

### 3.2.2 启动yarn-session

```bash
./yarn-session.sh -n 2 -s 2 -jm 1024 -tm 1024 -nm test -d

其中:
-n(--container):TaskManager 的数量。
-s(--slots): 每个 TaskManager 的 slot 数量，默认一个 slot 一个 core，默认每个
taskmanager 的 slot 的个数为 1，有时可以多一些 taskmanager，做冗余。
-jm:JobManager 的内存(单位 MB)。
-tm:每个 taskmanager 的内存(单位 MB)。
-nm:yarn 的 appName(现在 yarn 的 ui 上的名字)。
-d:后台执行。
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566406538-969f8554-499d-4d23-9376-76da536bb94d.png)

### 3.2.3 执行任务

```bash
./flink run -m yarn-cluster 
-c com.atguigu.flink.app.BatchWcApp 
/ext/flink0503-1.0-SNAPSHOT.jar 
--input /applog/flink/input.txt 
--output /applog/flink/output5.csv
```

![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566468734-c850bdd8-77e8-4bf4-8f97-c2e37dd158bd.png)


### 3.2.4 去yarn控制台查看任务状态
![image.png](https://cdn.nlark.com/yuque/0/2020/png/766178/1581566497706-7d8e65c0-e399-44d7-8481-6e8d01d6f985.png)

## 3.3 Kubernetes 部署
容器化部署时目前业界很流行的一项技术，基于 Docker 镜像运行能够让用户更加方便地对应用进行管理和运维。容器管理工具中最为流行的就是 Kubernetes(k8s)，而 Flink 也在最近的版本中支持了 k8s 部署模式。


### 3.3.1 搭建 Kubernetes 集群(略)

### 3.3.2 配置各组件的 yaml 文件
在 k8s 上构建 Flink Session Cluster，需要将 Flink 集群的组件对应的 docker 镜像分别在 k8s 上启动，包括 JobManager、TaskManager、JobManagerService 三个镜像服务。每个镜像服务都可以从中央镜像仓库中获取。


### 3.3.3 启动 Flink Session Cluster

```bash
// 启动 jobmanager-service 服务
kubectl create -f jobmanager-service.yaml
// 启动 jobmanager-deployment 服务
kubectl create -f jobmanager-deployment.yaml // 启动 taskmanager-deployment 服务
kubectl create -f taskmanager-deployment.yaml
```

### 3.3.4 访问 Flink UI 页面
集群启动后，就可以通过 JobManagerServicers 中配置的 WebUI 端口，用浏览器输入以下 url 来访问 Flink UI 页面了:
    
```bash
http://{JobManagerHost:Port}/api/v1/namespaces/default/services/flink-jobmanage
r:ui/proxy
```