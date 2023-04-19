[TOC]

# 1 openTSDB安装环境
- 安装HDFS集群；
- 安装HBase集群；
- 安装zookeeper集群
# 2 安装步骤

## 2.1 下载openTSDB
https://github.com/OpenTSDB/opentsdb/releases

下载tar.gz包。

## 2.2 解压缩

```bash
tar -zxvf opentsdb-2.3.1.tar.gz -C /usr/local
```

## 2.3 编译
这里下载的不是rpm包，需要编译生成可执行的文件，编译的结果会输出在./build里面

```bash
cd opentsdb
./build.sh
```
可能出现如下错误：

```
./build.sh执行报错
报错如下：
sd/BadRequestException.java ../src/tsd/ConnectionManager.java ../src/tsd/DropCachesRpc.java ../src/tsd/GnuplotException.java ../src/tsd/GraphHandler.java ../src/tsd/HttpJsonSerializer.java ../src/tsd/HttpSerializer.java ../src/tsd/HttpQuery.java ../src/tsd/HttpRpc.java ../src/tsd/HttpRpcPlugin.java ../src/tsd/HttpRpcPluginQuery.java ../src/tsd/LineBasedFrameDecoder.java ../src/tsd/LogsRpc.java ../src/tsd/PipelineFactory.java ../src/tsd/PutDataPointRpc.java ../src/tsd/QueryExecutor.java ../src/tsd/QueryRpc.java ../src/tsd/RpcHandler.java ../src/tsd/RpcPlugin.java ../src/tsd/RpcManager.java ../src/tsd/RpcUtil.java ../src/tsd/RTPublisher.java ../src/tsd/SearchRpc.java ../src/tsd/StaticFileRpc.java ../src/tsd/StatsRpc.java ../src/tsd/StorageExceptionHandler.java ../src/tsd/SuggestRpc.java ../src/tsd/TelnetRpc.java ../src/tsd/TreeRpc.java ../src/tsd/UniqueIdRpc.java ../src/tsd/WordSplitter.java ../src/uid/FailedToAssignUniqueIdException.java ../src/uid/NoSuchUniqueId.java ../src/uid/NoSuchUniqueName.java ../src/uid/RandomUniqueId.java ../src/uid/UniqueId.java ../src/uid/UniqueIdFilterPlugin.java ../src/uid/UniqueIdInterface.java ../src/utils/ByteArrayPair.java ../src/utils/ByteSet.java ../src/utils/Config.java ../src/utils/DateTime.java ../src/utils/Exceptions.java ../src/utils/FileSystem.java ../src/utils/JSON.java ../src/utils/JSONException.java ../src/utils/Pair.java ../src/utils/PluginLoader.java ../src/utils/Threads.java ../src/tools/BuildData.java ./src/net/opentsdb/query/expression/parser/*.java
javac: file not found: ./src/net/opentsdb/query/expression/parser/*.java
Usage: javac <options> <source files>
use -help for a list of possible options
make[1]: *** [.javac-stamp] Error 2
make[1]: Leaving directory `/usr/local/opentsdb-2.3.0/build'
make: *** [all] Error 2
```

这个报错【好像】是因为找不到某个jar包。

解决方法，使用如下命令：

```bash
mkdir build
cp -r third_party ./build
./build.sh
```

然后编译成功。</br>
ps：【编译成功的标志是在openTSDB父目录下生成build文件夹，并且会生成一个jar包：tsdb-2.3.0.jar】

## 2.4 创建openTSDB所需要的表
接着执行脚本，创建openTSDB所需要的表，这些命令均已写好，只需要执行相应脚本即可。

```bash
cd opentsdb
env COMPRESSION=NONE HBASE_HOME=/usr/local/hbase-2.0.0 ./src/create_table.sh
```

## 2.5 修改配置文件
将openTSDB文件目录下的opentsdb.conf文件拷贝到./build目录下，

```bash
mv opentsdb.conf build/opentsdb.conf
vim build/opentsdb.conf
```

配置内容如下【这里只列举出需要修改的配置项】：

```bash
# tsdb页面ui文件的地址，这里就写build文件夹下面的staticroot文件夹
tsd.http.staticroot = /usr/local/opentsdb/build/staticroot
# tsdb的缓存文件存放地址
tsd.http.cachedir=/tmp/opentsdb
# zookeeper保存hbase数据的-ROOT-所在的路径
tsd.storage.hbase.zk_basedir = /hbase
# hbase依赖的zookeeper地址
tsd.storage.hbase.zk_quorum=127.0.0.1:2181
```
> ps:`tsd.storage.hbase.zk_basedir = /hbase`这个值是zookeeper保存hbase数据的-ROOT-所在的路径。这个路径一般默认都是存放在/hbase路径下。**这个/hbase路径是一个虚拟路径**，并非物理真实存在。可以启动zkCli.sh服务查看。如下：
> 
> ```bash
> [zk: localhost:2181(CONNECTED) 1] ls /    #根下的文件【这里就是-root-所在的地方】
> [cluster, controller_epoch, brokers, zookeeper, admin, isr_change_notification, consumers, log_dir_event_notification, latest_producer_id_block, config, hbase]
> 
> [zk: localhost:2181(CONNECTED) 2] ls /hbase   #/hbase下的文件
> [replication, meta-region-server, rs, splitWAL, backup-masters, table-lock, flush-table-proc, master-maintenance, region-in-transition, online-snapshot, switch, master, running, recovering-regions, draining, namespace, hbaseid, table]
> 
> [zk: localhost:2181(CONNECTED) 3] ls /hbase/table     #真实存在hbase中的表
> [hbase:meta, hbase:namespace, tsdb-tree, tsdb, tsdb-uid, tsdb-meta]
> [zk: localhost:2181(CONNECTED) 4] 
> ```

## 2.6 启动
运行脚本`./build/tsdb tsd --port=4242` 即可启动tsdb，端口4242指opentsdb服务的端口。当启动成功，打开http://localhost:4242即可看到opentsdb的web ui

# 3 openTSDB效果展示
打开http://localhost:4242,访问opentsdb的web ui

[![opentsdb_ui.png](https://p.pstatp.com/origin/ffaf0002259800d75903)](https://www.z4a.net/images/2020/07/14/opentsdb_ui.png)

- 1：这个是监控数据的起始时间。可以选择
- 2：这个是监控数据的终止时间。这里设置成now()，即直至此刻的数据
- 3：勾选这个选项，可以设置成自动加载数据
- 4：与part3对应，这里是自动加载数据的频数
- 5：这里就是openTSDB的精华所在，一个是metric，一个是tag。这个需要与相应的数据来设置。
- 6.图形展示页面可以放大/缩小—>相应的导致时间点的前移/后退。从而在页面上的展示就是图形的变化。

## 3.1 写入数据
往tsdb表中存放数据，可以尝试执行如下命令：

```bash
curl -i -X POST -d '{"metric":"mytest.cpu","timestamp":1531486607,"value":7,"tags":{"host":"localhost"}}' http://localhost:4242/api/put?details。
```
 需要注意的是更改timestamp的值。因为时序数据是一个二维数据点，每个时间戳对应一个数据点。但是我此刻的数据点可能不是你现在系统的时间点，最好改成你现在的时间点。【这个坑一定要注意，否则极不容易展示出数据】 如果不知道你此刻的时间戳，可以去执行./tsdb tsd脚本所在的窗口中查找。

```bash
2018-07-13 22:52:07,825 INFO  [AsyncHBase I/O Worker #2] TsdbQuery: 
TsdbQuery(start_time=1531492080, end_time=1531493527, metric=[0, 0, 1], filters=[],
rate=false, aggregator=sum, group_bys=()) matched 0 rows in 0 spans in 2.62747ms
```

