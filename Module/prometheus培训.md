[TOC]

# 1. prometheus介绍

开源的监控组件

时间序列数据存储

[![](https://s6.jpg.cm/2021/12/30/LJRYXE.png)](https://www.z4a.net/images/2021/12/30/image.png)

特点

1）由度量名称和key/value对标识的时间序列数据的多维数据模型

2）PromQL多维度灵活查询（类sql）

3）不依赖分布式存储，单个程序可运行（在k8s上是无敌的存在）

4）通过HTTP上的拉取模型进行时间序列收集（pull）

5）通过中介网关支持推送时间序列（clickhouse）

6）目标通过服务发现或静态配置发现 （刮擦任务）

7）支持多种图形和仪表板模式（配合grafana）

# 2. prometheus安装

## 2.1 Linux安装

1）创建监控程序目录

```bash
[root@bigdata3 opt]# mkdir /opt/monitor
```

2）导入软件

```bash
[root@bigdata3 monitor]# ll
总用量 321032
-rw-r--r--. 1 root root  48497454 7月   3 2019 prometheus-2.10.0.linux-amd64.tar.gz
[root@bigdata3 monitor]# pwd
/opt/monitor
```


3）解压软件

```bash
[root@bigdata3 monitor]# tar xf  prometheus-2.10.0.linux-amd64.tar.gz
```

4）重命名文件夹

```bash
[root@bigdata3 monitor]# mv prometheus-2.10.0.linux-amd64 prometheus
```

5）启动prometheus

```bash
/opt/monitor/prometheus/prometheus --config.file="/opt/monitor/prometheus/prometheus.yml"> /opt/monitor/prometheus/prometheus.log --web.enable-lifecycle 2>&1 &
```

## 2.2 mac安装

1）安装brew

```bash
/bin/zsh -c "$(curl -fsSL https://gitee.com/cunkai/HomebrewCN/raw/master/Homebrew.sh)"
```

2）更新brew

```bash
brew update
```

3）安装prometheus

```bash
brew install prometheus
```

4）启动prometheus

```bash
brew services start prometheus
```

5）带参数启动

```bash
prometheus --config.file=/usr/local/etc/prometheus.yml  --web.enable-lifecycle 2>&1 &
```

建议使用第五步启动方式，找到配置文件加上--web.enable-lifecycle，此参数的意义在于我们修改了prometheus.yml后直接远程热加载即可，不用重启服务，使用下面的命令即可。

```bash
curl -XPOST http://localhost:9090/-/reload
```

6）拓展启动

scree命令用法 [https://www.runoob.com/linux/linux-comm-screen.html](https://www.runoob.com/linux/linux-comm-screen.html)﻿

# 3. prometheus配置

原始解压完毕的配置如下

```properties
# my global config
########################控制prometheus行为的全局配置########################################
global:
  #下面的参数用来指定应用程序或服务抓取数据的时间间隔，这个值是时间序列的颗粒度，即该序列中每个数据点所覆盖的时间段
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  #用来指定prometheus评估规则的频率，目前两种记录规则与报警规则。
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
######################用来设置prometheus的告警#############################################
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
######################配置告警规则的文件####################################################
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
####################用来指定prometheus抓取的所有目标##########################################
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'
    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.
    static_configs:
    - targets: ['localhost:9090']
```

官方文档：

详细解说总配置文件

```properties
global:
  # 默认情况下刮取目标的频率
  [ scrape_interval: <duration> | default = 1m ]

  # 刮取目标的超时时间.
  [ scrape_timeout: <duration> | default = 10s ]

  # 评估规则的频率.
  [ evaluation_interval: <duration> | default = 1m ]

  # 与外部系统（联邦、远程存储、警报管理器）通信时要添加到任何时间序列或警报的标签。
  external_labels:
    [ <labelname>: <labelvalue> ... ]

  # PromQL查询记录到的文件.
  # 从新加载配置从新打开文件.
  [ query_log_file: <string> ]
```

```properties
# 规则文件指定全局的列表。从中读取规则和警报所有匹配的文件。
rule_files:
  [ - <filepath_glob> ... ]
```

```properties
# 刮擦的配置列表.
scrape_configs:
  [ - <scrape_config> ... ]
```

```properties
# 报警指定与alertmanager相关的配置.
alerting:
  alert_relabel_configs:
    [ - <relabel_config> ... ]
  alertmanagers:
    [ - <alertmanager_config> ... ]
```

```properties
# 远程写入相关的配置，比如后端时间序列数据库对接clickhouse等.
remote_write:
  [ - <remote_write> ... ]
```

```properties
# 远程读取功能相关的配置.
remote_read:
  [ - <remote_read> ... ]
```

详细配置解说，官方文档最权威[https://prometheus.io/docs/prometheus/latest/configuration/configuration/](https://prometheus.io/docs/prometheus/latest/configuration/configuration/)﻿

# 4. prometheus刮擦job

其实有很多发现job的方式，下面说下日常使用的两种，更多的发现方式去查官网。

基于文件发现，定期扫描文件抓取地址。

```properties
# List of file service discovery configurations.
file_sd_configs:
  [ - <file_sd_config> ... ]
```
基于静态配置，定期扫描地址端口。

```plain
# List of labeled statically configured targets for this job.
static_configs:
  [ - <static_config> ... ]
```
直接说线上生产的prometheus配置。
```plain
#全局变量
global:
  scrape_interval: 30s
  scrape_timeout: 10s
  evaluation_interval: 15s
#告警地址
alerting:
  alertmanagers:
  - scheme: http
    timeout: 10s
    api_version: v1
    static_configs:
    - targets: []
#作业抓取
scrape_configs:
    #prometheus自己的状态监控
- job_name: prometheus
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - 10.xx.xx.97:9090
    labels:
      instance: prometheus
      #mysql状态监控
- job_name: mysql
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - 10.xx.xx.95:9104
    labels:
      instance: mysql_cm
      #b节点yarn队列监控
- job_name: b_yarn
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  file_sd_configs:
  - files:
    - /data01/app/prometheus/monitor_config/yarn/*.yml
    refresh_interval: 5s
    #es监控
- job_name: es
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - 10.xx.xx.244:9104
    #主机node监控
- job_name: node
  honor_timestamps: true
  scrape_interval: 30s
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  static_configs:
  - targets:
    - 10.xx.xx.176:9100
    - 10.xx.xx.177:9100
    - 10.xx.xx.178:9100
    - 10.xx.xx.179:9100
    - 10.xx.xx.180:9100
    - 10.xx.xx.181:9100
    ......
    ......
    - 10.xx.xx.194:9100
```
从简洁与维护的角度讲，上边的配置文件有需要优化的地方，基于静态发现的弊端就是当需要监控的点比较多时这个yml文件就及其复杂庞大了。
所以在一批机器上进行相同的监控时，基于配置文件发现类型的监控就会简化分解yml文件。
做法：
在prometheus程序文件夹中建立monitor_config文件夹在文件夹中归类监控项目文件夹

```plain
[root@bigdata3 prometheus]# mkdir /opt/monitor/prometheus/monitor_config/
[root@bigdata3 prometheus]# mkdir /opt/monitor/prometheus/monitor_config/hosts
[root@bigdata3 prometheus]# mkdir /opt/monitor/prometheus/monitor_config/yarn
[root@bigdata3 prometheus]# mkdir /opt/monitor/prometheus/monitor_config/mysql
[root@bigdata3 prometheus]# ls /opt/monitor/prometheus/monitor_config/
hosts  yarn mysql
```
然后在对应的目录里写入被监控的对象地址文件即可,配置文件为yml结尾
如：
```
- targets: [ "xx.xx.xx.xx:9100" ]
  labels:
    group: "hosts"
    kind: "B"
```
具体可参考b节点yarn队列监控配置。

# 5. prometheus命令

prometheus是有很多命令的，我们进入到程序里可以使用

```properties
[root@bigdata01 prometheus]# ./prometheus -h
usage: prometheus [<flags>]

The Prometheus monitoring server

Flags:
  -h, --help                     Show context-sensitive help (also try --help-long and --help-man).
      --version                  Show application version.
      --config.file="prometheus.yml"  
                                 Prometheus configuration file path.
      --web.listen-address="0.0.0.0:9090"  
                                 Address to listen on for UI, API, and telemetry.
      --web.read-timeout=5m      Maximum duration before timing out read of the request, and closing idle connections.
      --web.max-connections=512  Maximum number of simultaneous connections.
      --web.external-url=<URL>   The URL under which Prometheus is externally reachable (for example, if Prometheus is served via a reverse proxy). Used for generating relative and absolute links back to Prometheus itself. If the URL has a path portion, it will be used to prefix
                                 all HTTP endpoints served by Prometheus. If omitted, relevant URL components will be derived automatically.
      --web.route-prefix=<path>  Prefix for the internal routes of web endpoints. Defaults to path of --web.external-url.
      --web.user-assets=<path>   Path to static asset directory, available at /user.
      --web.enable-lifecycle     Enable shutdown and reload via HTTP request.
      --web.enable-admin-api     Enable API endpoints for admin control actions.
      --web.console.templates="consoles"  
                                 Path to the console template directory, available at /consoles.
      --web.console.libraries="console_libraries"  
                                 Path to the console library directory.
      --web.page-title="Prometheus Time Series Collection and Processing Server"  
                                 Document title of Prometheus instance.
      --web.cors.origin=".*"     Regex for CORS origin. It is fully anchored. Example: 'https?://(domain1|domain2)\.com'
      --storage.tsdb.path="data/"  
                                 Base path for metrics storage.
      --storage.tsdb.retention=STORAGE.TSDB.RETENTION  
                                 [DEPRECATED] How long to retain samples in storage. This flag has been deprecated, use "storage.tsdb.retention.time" instead.
      --storage.tsdb.retention.time=STORAGE.TSDB.RETENTION.TIME  
                                 How long to retain samples in storage. When this flag is set it overrides "storage.tsdb.retention". If neither this flag nor "storage.tsdb.retention" nor "storage.tsdb.retention.size" is set, the retention time defaults to 15d.
      --storage.tsdb.retention.size=STORAGE.TSDB.RETENTION.SIZE  
                                 [EXPERIMENTAL] Maximum number of bytes that can be stored for blocks. Units supported: KB, MB, GB, TB, PB. This flag is experimental and can be changed in future releases.
      --storage.tsdb.no-lockfile  
                                 Do not create lockfile in data directory.
      --storage.tsdb.allow-overlapping-blocks  
                                 [EXPERIMENTAL] Allow overlapping blocks, which in turn enables vertical compaction and vertical query merge.
      --storage.remote.flush-deadline=<duration>  
                                 How long to wait flushing sample on shutdown or config reload.
      --storage.remote.read-sample-limit=5e7  
                                 Maximum overall number of samples to return via the remote read interface, in a single query. 0 means no limit.
      --storage.remote.read-concurrent-limit=10  
                                 Maximum number of concurrent remote read calls. 0 means no limit.
      --rules.alert.for-outage-tolerance=1h  
                                 Max time to tolerate prometheus outage for restoring "for" state of alert.
      --rules.alert.for-grace-period=10m  
                                 Minimum duration between alert and restored "for" state. This is maintained only for alerts with configured "for" time greater than grace period.
      --rules.alert.resend-delay=1m  
                                 Minimum amount of time to wait before resending an alert to Alertmanager.
      --alertmanager.notification-queue-capacity=10000  
                                 The capacity of the queue for pending Alertmanager notifications.
      --alertmanager.timeout=10s  
                                 Timeout for sending alerts to Alertmanager.
      --query.lookback-delta=5m  The maximum lookback duration for retrieving metrics during expression evaluations.
      --query.timeout=2m         Maximum time a query may take before being aborted.
      --query.max-concurrency=20  
                                 Maximum number of queries executed concurrently.
      --query.max-samples=50000000  
                                 Maximum number of samples a single query can load into memory. Note that queries will fail if they try to load more samples than this into memory, so this also limits the number of samples a query can return.
      --log.level=info           Only log messages with the given severity or above. One of: [debug, info, warn, error]
      --log.format=logfmt        Output format of log messages. One of: [logfmt, json]
```

我们在启动时可以加的热启动参数就是在这里找到的。还有定义存储天数，启动文件指定，端口指定都可以。

具体中文解释看下面的表格

|参数名称|含义|备注|
|------------------------------------------------------------|------------------------------------------------------------|------------------------------------------------------------|
|--version|显示应用的版本信息||
|**配置文件参数**|||
|--config.file="prometheus.yml"|Prometheus配置文件路径||
|**WEB服务参数**|||
|--web.listen-address="0.0.0.0:9090"|UI、API、遥测（telemetry）监听地址||
|--web.read-timeout=5m|读取请求和关闭空闲连接的最大超时时间|默认值：5m|
|--web.max-connections=512|最大同时连接数|默认值：512|
|--web.external-url=<url></url>|可从外部访问普罗米修斯的URL|如果Prometheus存在反向代理时使用，用于生成相对或者绝对链接，返回到Prometheus本身，如果URL存在路径部分，它将用于给Prometheus服务的所有HTTP端点加前缀，如果省略，将自动派生相关的URL组件。|
|--web.route-prefix=<path></path>|Web端点的内部路由|默认路径：--web.external-url|
|--web.user-assets=<path></path>|静态资产目录的路径|在/user路径下生效可用|
|--web.enable-lifecycle|通过HTTP请求启用关闭（shutdown）和重载（reload）||
|--web.enable-admin-api|启用管理员行为API端点||
|--web.console.templates="consoles"|总线模板目录路径|在/consoles路径下生效可用|
|--web.console.libraries="console_libraries"|总线库文件目录路径||
|--web.page-title="PrometheusTimeSeriesCollectionandProcessingServer"|Prometheus实例的文档标题||
|--web.cors.origin=".*"|CORS来源的正则Regex，是完全锚定的|例如：'https?://(domain1\|domain2).com'|
|**数据存储参数**|||
|--storage.tsdb.path="data/"|指标存储的根路径||
|--storage.tsdb.retention=STORAGE.TSDB.RETENTION|[DEPRECATED]样例存储时间|此标签已经丢弃，用"storage.tsdb.retention.time"替代|
|--storage.tsdb.retention.time=STORAGE.TSDB.RETENTION.TIME|存储时长，如果此参数设置了，会覆盖"storage.tsdb.retention"参数；如果设置了"storage.tsdb.retention"或者"storage.tsdb.retention.size"参数，存储时间默认是15d（天），单位：y,w,d,h,m,s,ms||
|--storage.tsdb.retention.size=STORAGE.TSDB.RETENTION.SIZE|[EXPERIMENTAL]试验性的。存储为块的最大字节数，需要使用一个单位，支持：B,KB,MB,GB,TB,PB,EB|此标签处于试验中，未来版本会改变|
|--storage.tsdb.no-lockfile|不在data目录下创建锁文件||
|--storage.tsdb.allow-overlapping-blocks|[EXPERIMENTAL]试验性的。允许重叠块，可以支持垂直压缩和垂直查询合并。||
|--storage.tsdb.wal-compression|压缩tsdb的WAL|WAL(Write-aheadlogging,预写日志)，WAL被分割成默认大小为128M的文件段（segment），之前版本默认大小是256M，文件段以数字命名，长度为8位的整形。WAL的写入单位是页（page），每页的大小为32KB，所以每个段大小必须是页的大小的整数倍。如果WAL一次性写入的页数超过一个段的空闲页数，就会创建一个新的文件段来保存这些页，从而确保一次性写入的页不会跨段存储。|
|--storage.remote.flush-deadline=<duration></duration>|关闭或者配置重载时刷新示例的等待时长||
|--storage.remote.read-sample-limit=5e7|在单个查询中通过远程读取接口返回的最大样本总数。0表示无限制。对于流式响应类型，将忽略此限制。||
|--storage.remote.read-concurrent-limit=10|最大并发远程读取调用数。0表示无限制。||
|--storage.remote.read-max-bytes-in-frame=1048576|在封送处理之前，用于流式传输远程读取响应类型的单个帧中的最大字节数。请注意，客户机可能对帧大小也有限制。|默认情况下，protobuf建议使用1MB。|
|**告警规则相关参数**|||
|--rules.alert.for-outage-tolerance=1h|允许prometheus中断以恢复“for”警报状态的最长时间。||
|--rules.alert.for-grace-period=10m|警报和恢复的“for”状态之间的最短持续时间。这仅对配置的“for”时间大于宽限期的警报进行维护。||
|--rules.alert.resend-delay=1m|向Alertmanager重新发送警报之前等待的最短时间。||
|**告警管理中心相关参数**|||
|--alertmanager.notification-queue-capacity=10000|挂起的Alertmanager通知的队列容量。|默认值：10000|
|--alertmanager.timeout=10s|发送告警到Alertmanager的超时时间|默认值：10s|
|**数据查询参数**|||
|--query.lookback-delta=5m|通过表达式解析和联合检索指标的最大反馈时间|默认值：5m|
|--query.timeout=2m|查询中止前可能需要的最长时间。|默认值：2m|
|--query.max-concurrency=20|并发（concurrently）执行查询的最大值||
|--query.max-samples=50000000|单个查询可以加载到内存中的最大样本数。注意，如果查询试图将更多的样本加载到内存中，则会失败，因此这也限制了查询可以返回的样本数。|数量级：5千万|
|**日志信息参数**|||
|--log.level=info|仅记录给定的日志级别及以上的信息|可选参数值：[debug,info,warn,error]，其中之一|
|--log.format=logfmt|日志信息输出格式|可选参数值：[logfmt,json]，其中之一|


# 6. PromQL

PromQL (Prometheus Query Language) 是 Prometheus 自己开发的数据查询 DSL 语言，语言表现力非常丰富，内置函数很多，在日常数据可视化以及rule 告警中都会使用到它。

在页面 http://localhost:9090/graph 中，输入下面的查询语句，查看结果，例如：

```properties
http_requests_total{code="200"}
```

字符串和数字

字符串: 在查询语句中，字符串往往作为查询条件 labels 的值，和 Golang 字符串语法一致，可以使用 "", '', 或者 \`\` , 格式如：

```properties
"this is a string"
'these are unescaped: \n \\ \t'
`these are not unescaped: \n ' " \t`
```
正数，浮点数: 表达式中可以使用正数或浮点数，例如：

```properties
3
-2.4
```


## 查询结果类型

PromQL 查询结果主要有 3 种类型：

瞬时数据 (Instant vector): 包含一组时序，每个时序只有一个点，例如：http\_requests\_total

区间数据 (Range vector): 包含一组时序，每个时序有多个点，例如：http\_requests\_total\[5m\]

纯量数据 (Scalar): 纯量只有一个数字，没有时序，例如：count(http\_requests\_total)

## 查询条件

Prometheus 存储的是时序数据，而它的时序是由名字和一组标签构成的，其实名字也可以写出标签的形式，例如 http\_requests\_total 等价于 {name=”http\_requests\_total”}。

一个简单的查询相当于是对各种标签的筛选，例如：

```properties
http_requests_total{code="200"}// 表示查询名字为 http_requests_total，code 为 "200" 的数据
```

查询条件支持正则匹配，例如：

```properties
http_requests_total{code!="200"}// 表示查询 code 不为 "200" 的数据
http_requests_total{code=～"2.."}// 表示查询 code 为 "2xx" 的数据
http_requests_total{code!=～"2.."}// 表示查询 code 不为 "2xx" 的数据
```

操作符

Prometheus 查询语句中，支持常见的各种表达式操作符，例如

算术运算符:

支持的算术运算符有 +，-，\*，/，%，^。

```properties
例如 http_requests_total * 2 表示将 http_requests_total 所有数据 double 一倍。
```

比较运算符:

支持的比较运算符有 ==，!=，>，<，>=，<=。

```properties
例如 http_requests_total > 100 表示 http_requests_total 结果中大于 100 的数据。
```

逻辑运算符:

支持的逻辑运算符有 and，or，unless。

```properties
例如 http_requests_total == 5 or http_requests_total == 2 表示 http_requests_total 结果中等于 5 或者 2 的数据。
```

聚合运算符:

支持的聚合运算符有 sum（求和），min（最小），max（最大），avg（平均），stddev（标准差），stdvar（方差），count（统计），count\_values（值统计），bottomk（后n条时序），topk（前n条时序），quantile（分布统计）

```properties
例如 max(http_requests_total) 表示 http_requests_total 结果中最大的数据。
```

注意，和四则运算类型，Prometheus 的运算符也有优先级，它们遵从（^）> (\*, /, %) > (+, -) > (==, !=, <=, <, >=, >) > (and, unless) > (or) 的原则。

## 内置函数

Prometheus 内置不少函数，方便查询以及数据格式化，例如将结果由浮点数转为整数的 floor 和 ceil，

```properties
floor(avg(http_requests_total{code="200"}))
```

```plain
ceil(avg(http_requests_total{code="200"}))
```

查看 http\_requests\_total 5分钟内，平均每秒数据

```properties
rate(http_requests_total[5m])
```

## 下面以mac电脑cpu使用率来描述下promql使用。

1）启动node\_exporter

```properties
brew services start node_exporter
```

2）查看下我的prometheus配置文件

```properties
wanghongxing@Bob-MacBook-Pro ~ % cat /usr/local/etc/prometheus.yml
global:
  scrape_interval: 15s

  external_labels:
       monitor: 'codelab-monitor'
       origin_prometheus: 'whx'
scrape_configs:
  - job_name: "prometheus"
    static_configs:
    - targets: ["127.0.0.1:9090"]

  - job_name: "node"
    static_configs:
    - targets: ["127.0.0.1:9100"]

  - job_name: "yarn"
    file_sd_configs:
    - files: ["/usr/local/etc/monitor_config/*.yml"]
      refresh_interval: 5s
```

3）看下我的node监控已经拉到数据。

[![LJRsah.png](https://s6.jpg.cm/2021/12/30/LJRsah.png)](https://www.z4a.net/images/2021/12/30/2.png)

可以看到有三个job监控，node是我刚才配置的node\_exporter监控

4）分析cpu

在指标搜索框里输入cpu，可以看到节点cpu信息，这个是按照秒级汇总。

[![LJRmK2.png](https://s6.jpg.cm/2021/12/30/LJRmK2.png)](https://www.helloimg.com/images/2021/12/30/GpcDwo.png)

PromQL有一个名为irate的函数，用于计算距离向量中时间序列的每秒瞬时增长率。让我们在node\_cpu\_seconds\_total度量上使用irate函数。

在查询框中输入:

node\_cpu\_seconds\_total可以看到我部署采集的电脑上各个CPU各个的指标，因为我的电脑是16核所以这边可以看到0-15个线不同维度的状态。

[![LJRrvQ.png](https://s6.jpg.cm/2021/12/30/LJRrvQ.png)](https://www.z4a.net/images/2021/12/30/image_2.png)

完善上面的语句，查看每秒瞬时增长。job的标签是我们在prometheus.yml文件中配置的刮擦名称，可以在target里看到

irate(node\_cpu\_seconds\_total{job="node"}\[5m\])

[![LJRFeS.png](https://s6.jpg.cm/2021/12/30/LJRFeS.png)](https://www.z4a.net/images/2021/12/30/image_3.png)

现在将irate函数封装在avg聚合中，并添加了一个by子句，该子句通过实例标签聚合。这将产生三个新的指标，使用来自所有CPU和所有模式的值来平均主机的CPU使用情况。

avg(irate(node\_cpu\_seconds\_total{job="node"}\[5m\])) by (instance)

avg (irate(node\_cpu\_seconds\_total{job="node",mode="idle"}\[5m\])) by (instance) \* 100

[![LJRNZT.png](https://s6.jpg.cm/2021/12/30/LJRNZT.png)](https://www.z4a.net/images/2021/12/30/image_4.png)

在这里，查询中添加了一个值为idle的mode标签。这只查询空闲数据。我们通过实例求出结果的平均值，并将其乘以100。现在我们在每台主机上都有5分钟内空闲使用的平均百分比。我们可以把这个变成百分数用这个值减去100，就像这样:

100 - avg (irate(node\_cpu\_seconds\_total{job="node",mode="idle"}\[5m\])) by (instance) \* 100

[![LJRvWH.png](https://s6.jpg.cm/2021/12/30/LJRvWH.png)](https://www.helloimg.com/images/2021/12/30/Gpc8qS.png)

# 7. 对比mysql的SQL

与 SQL 对比

下面将以 Prometheus server 收集的 http\_requests\_total 时序数据为例子展开对比。

MySQL 数据准备

```sql
mysql>
# 创建数据库
create database prometheus_practice;
use prometheus_practice;
# 创建 http_requests_total 表
CREATE TABLE http_requests_total (
    code VARCHAR(256),
    handler VARCHAR(256),
    instance VARCHAR(256),
    job VARCHAR(256),
    method VARCHAR(256),
    created_at DOUBLE NOT NULL,
    value DOUBLE NOT NULL) ENGINE=InnoDB DEFAULT CHARSET=utf8;
ALTER TABLE http_requests_total ADD INDEX created_at_index (created_at);
# 初始化数据
# time at 2017/5/22 14:45:27
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "query_range", "localhost:9090", "prometheus", "get", 1495435527, 3);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("400", "query_range", "localhost:9090", "prometheus", "get", 1495435527, 5);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "prometheus", "localhost:9090", "prometheus", "get", 1495435527, 6418);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "static", "localhost:9090", "prometheus", "get", 1495435527, 9);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("304", "static", "localhost:9090", "prometheus", "get", 1495435527, 19);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "query", "localhost:9090", "prometheus", "get", 1495435527, 87);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("400", "query", "localhost:9090", "prometheus", "get", 1495435527, 26);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "graph", "localhost:9090", "prometheus", "get", 1495435527, 7);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "label_values", "localhost:9090", "prometheus", "get", 1495435527, 7);
# time at 2017/5/22 14:48:27
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "query_range", "localhost:9090", "prometheus", "get", 1495435707, 3);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("400", "query_range", "localhost:9090", "prometheus", "get", 1495435707, 5);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "prometheus", "localhost:9090", "prometheus", "get", 1495435707, 6418);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "static", "localhost:9090", "prometheus", "get", 1495435707, 9);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("304", "static", "localhost:9090", "prometheus", "get", 1495435707, 19);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "query", "localhost:9090", "prometheus", "get", 1495435707, 87);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("400", "query", "localhost:9090", "prometheus", "get", 1495435707, 26);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "graph", "localhost:9090", "prometheus", "get", 1495435707, 7);
INSERT INTO http_requests_total (code, handler, instance, job, method, created_at, value) values ("200", "label_values", "localhost:9090", "prometheus", "get", 1495435707, 7);
```

数据初始完成后，通过查询可以看到如下数据：



```sql
mysql>
mysql> select * from http_requests_total;
+------+--------------+----------------+------------+--------+------------+-------+
| code | handler      | instance       | job        | method | created_at | value |
+------+--------------+----------------+------------+--------+------------+-------+
| 200  | query_range  | localhost:9090 | prometheus | get    | 1495435527 |     3 |
| 400  | query_range  | localhost:9090 | prometheus | get    | 1495435527 |     5 |
| 200  | prometheus   | localhost:9090 | prometheus | get    | 1495435527 |  6418 |
| 200  | static       | localhost:9090 | prometheus | get    | 1495435527 |     9 |
| 304  | static       | localhost:9090 | prometheus | get    | 1495435527 |    19 |
| 200  | query        | localhost:9090 | prometheus | get    | 1495435527 |    87 |
| 400  | query        | localhost:9090 | prometheus | get    | 1495435527 |    26 |
| 200  | graph        | localhost:9090 | prometheus | get    | 1495435527 |     7 |
| 200  | label_values | localhost:9090 | prometheus | get    | 1495435527 |     7 |
| 200  | query_range  | localhost:9090 | prometheus | get    | 1495435707 |     3 |
| 400  | query_range  | localhost:9090 | prometheus | get    | 1495435707 |     5 |
| 200  | prometheus   | localhost:9090 | prometheus | get    | 1495435707 |  6418 |
| 200  | static       | localhost:9090 | prometheus | get    | 1495435707 |     9 |
| 304  | static       | localhost:9090 | prometheus | get    | 1495435707 |    19 |
| 200  | query        | localhost:9090 | prometheus | get    | 1495435707 |    87 |
| 400  | query        | localhost:9090 | prometheus | get    | 1495435707 |    26 |
| 200  | graph        | localhost:9090 | prometheus | get    | 1495435707 |     7 |
| 200  | label_values | localhost:9090 | prometheus | get    | 1495435707 |     7 |
+------+--------------+----------------+------------+--------+------------+-------+
18 rows in set (0.00 sec)
```

基本查询对比

假设当前时间为 2017/5/22 14:48:30

查询当前所有数据

```sql
// PromQL
http_requests_total
// MySQL
SELECT * from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710;
```

我们查询 MySQL 数据的时候，需要将当前时间向前推一定间隔，比如这里的 10s (Prometheus 数据抓取间隔)，这样才能确保查询到数据，而 PromQL 自动帮我们实现了这个逻辑。

条件查询
```sql
// PromQL
http_requests_total{code="200", handler="query"}
// MySQL
SELECT * from http_requests_total WHERE code="200" AND handler="query" AND created_at BETWEEN 1495435700 AND 1495435710;
```

模糊查询: code 为 2xx 的数据

```sql
// PromQL
http_requests_total{code~="2xx"}
// MySQL
SELECT * from http_requests_total WHERE code LIKE "%2%" AND created_at BETWEEN 1495435700 AND 1495435710;
```

比较查询: value 大于 100 的数据

```sql
// PromQL
http_requests_total > 100
// MySQL
SELECT * from http_requests_total WHERE value > 100 AND created_at BETWEEN 1495435700 AND 1495435710;
```

范围区间查询: 过去 5 分钟数据

```sql
// PromQL
http_requests_total[5m]
// MySQL
SELECT * from http_requests_total WHERE created_at BETWEEN 1495435410 AND 1495435710;
```

聚合, 统计高级查询

count 查询: 统计当前记录总数

```sql
// PromQL
count(http_requests_total)
// MySQL
SELECT COUNT(*) from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710;
```

sum 查询: 统计当前数据总值

```sql
// PromQL
sum(http_requests_total)
// MySQL
SELECT SUM(value) from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710;
```

avg 查询: 统计当前数据平均值

```sql
// PromQL
avg(http_requests_total)
// MySQL
SELECT AVG(value) from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710;
```

top 查询: 查询最靠前的 3 个值

```sql
// PromQL
topk(3, http_requests_total)
// MySQL
SELECT * from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710 ORDER BY value DESC LIMIT 3;
```

irate 查询，过去 5 分钟平均每秒数值

```sql
// PromQL
irate(http_requests_total[5m])
// MySQL
SELECT code, handler, instance, job, method, SUM(value)/300 AS value from http_requests_total WHERE created_at BETWEEN 1495435700 AND 1495435710  GROUP BY code, handler, instance, job, method;
```

总结

通过以上一些示例可以看出，在常用查询和统计方面，PromQL 比 MySQL 简单和丰富很多，而且查询性能也高不少。

# 8. 集群模式

**Hierarchical federation**

Hierarchical federation allows Prometheus to scale to environments with tens of data centers and millions of nodes. In this use case, the federation topology resembles a tree, with higher-level Prometheus servers collecting aggregated time series data from a larger number of subordinated servers.

For example, a setup might consist of many per-datacenter Prometheus servers that collect data in high detail (instance-level drill-down), and a set of global Prometheus servers which collect and store only aggregated data (job-level drill-down) from those local servers. This provides an aggregate global view and detailed local views.

**Cross-service federation**

In cross-service federation, a Prometheus server of one service is configured to scrape selected data from another service's Prometheus server to enable alerting and queries against both datasets within a single server.

For example, a cluster scheduler running multiple services might expose resource usage information (like memory and CPU usage) about service instances running on the cluster. On the other hand, a service running on that cluster will only expose application-specific service metrics. Often, these two sets of metrics are scraped by separate Prometheus servers. Using federation, the Prometheus server containing service-level metrics may pull in the cluster resource usage metrics about its specific service from the cluster Prometheus, so that both sets of metrics can be used within that server.

[![LJRksW.png](https://s6.jpg.cm/2021/12/30/LJRksW.png)](https://www.helloimg.com/images/2021/12/30/GpcIfD.png)
