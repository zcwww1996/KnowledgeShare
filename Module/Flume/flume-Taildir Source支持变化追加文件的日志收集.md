[TOC]
官文文档：

> **Taildir Source**
> 
> **Note   This source is provided as a preview feature. It does not work on Windows.**
>
> Watch the specified files, and tail them in nearly real-time once detected new lines appended to the each files. If the new lines are being written, this source will retry reading them in wait for the completion of the write.
> 
> This source is reliable and will not miss data even when the tailing files rotate. It periodically writes the last read position of each files on the given position file in JSON format. If Flume is stopped or down for some reason, it can restart tailing from the position written on the existing position file.
> 
> In other use case, this source can also start tailing from the arbitrary position for each files using the given position file. When there is no position file on the specified path, it will start tailing from the first line of each files by default.
> 
> Files will be consumed in order of their modification time. File with the oldest modification time will be consumed first.
> 
> This source does not rename or delete or do any modifications to the file being tailed. Currently this source does not support tailing binary files. It reads text files line by line.

|Property Name|Default|Description|
|---|---|---|
|channels|–| |
|type|–|The component type name, needs to be TAILDIR.|
|filegroups|–|Space-separated list of file groups. Each file group indicates a set of files to be tailed.|
|filegroups.<filegroupName>|–|Absolute path of the file group. Regular expression (and not file system patterns) can be used for filename only.|
|positionFile|~/.|flume/taildir_position.json	File in JSON format to record the inode, the absolute path and the last position of each tailing file.|
|headers.<filegroupName>.<headerKey>|–|Header value which is the set with header key. Multiple headers can be specified for one file group.|
|byteOffsetHeader|false|Whether to add the byte offset of a tailed line to a header called ‘byteoffset’.|
|skipToEnd|false|Whether to skip the position to EOF in the case of files not written on the position file.|
|idleTimeout|120000|Time (ms) to close inactive files. If the closed file is appended new lines to, this source will automatically re-open it.|
|writePosInterval|3000|Interval time (ms) to write the last position of each file on the position file.|
|batchSize|100|Max number of lines to read and send to the channel at a time. Using the default is usually fine.|
|backoffSleepIncrement|1000|The increment for time delay before reattempting to poll for new data, when the last attempt did not find any new data.|
|maxBackoffSleep|5000|The max time delay between each reattempt to poll for new data, when the last attempt did not find any new data.|
|cachePatternMatching|true|Listing directories and applying the filename regex pattern may be time consuming for directories containing thousands of files.| Caching the list of matching files can improve performance.| The order in which files are consumed will also be cached.| Requires that the file system keeps track of modification times with at least a 1-second granularity.|
|fileHeader|false|Whether to add a header storing the absolute path filename.|
|fileHeaderKey|file|Header key to use when appending absolute path filename to event header.|

Example for agent named a1:

```properties
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = TAILDIR
a1.sources.r1.channels = c1
a1.sources.r1.positionFile = /var/log/flume/taildir_position.json
a1.sources.r1.filegroups = f1 f2
a1.sources.r1.filegroups.f1 = /var/log/test1/example.log
a1.sources.r1.headers.f1.headerKey1 = value1
a1.sources.r1.filegroups.f2 = /var/log/test2/.*log.*
a1.sources.r1.headers.f2.headerKey1 = value2
a1.sources.r1.headers.f2.headerKey2 = value2-2
a1.sources.r1.fileHeader = true
```

一个配正在跑的配置文件例子：

```properties
# Name the components on this agent  
a1.sources = r1  
a1.sinks = k1  
a1.channels = c1  
  
# Describe/configure the source  
a1.sources.r1.type = TAILDIR  
a1.sources.r1.channels = c1  
a1.sources.r1.positionFile = /home/web_admin/opt/v2_flume-apache170/logfile_stats/x1/taildir_position.json    
a1.sources.r1.filegroups = f1                            
a1.sources.r1.filegroups.f1 = /home/zl/xsvr/server/xgame_1/logs/act/zl_war.*log.*  
a1.sources.r1.headers.f1.headerKey1 = value1               
a1.sources.r1.fileHeader = true  
  
  
# Describe the sink  
a1.sinks.k1.type = com.flume.dome.mysink.DBsqlSink  
a1.sinks.k1.hostname = jdbc:postgresql://192.168.20.243:5432  
#a1.sinks.k1.port = 5432  
a1.sinks.k1.databaseName = game_log  
a1.sinks.k1.tableName = zl_log_info  
a1.sinks.k1.user = game  
a1.sinks.k1.password = game123  
a1.sinks.k1.serverId = 1  
a1.sinks.k1.channel = c1  
a1.sinks.k1.josnTo = true  
  
# Use a channel which buffers events in memory  
a1.channels.c1.type = memory  
a1.channels.c1.capacity = 5000  
a1.channels.c1.transactionCapacity = 5000
```

介绍：

# 1. flume监控文件
对文件实时监控，如果发现文件有新日志，立刻收集并发送，通过sink进行入库收集。
缺点：如果中断，不会记录文件的当前收集状态，重启后只会收集新的日志，会造成数据丢失。但速度快
配置示例：

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/zl/xsvr/server/xgame_1/logs/act/zl_war.log
#a1.sources.tailsource-1.command = for i in /path/*.log; do cat $i; done
a1.sources.r1.channels = c1

# Describe the sink
a1.sinks.k1.type = com.flume.dome.mysink.DBsqlSink
a1.sinks.k1.hostname = jdbc:postgresql://192.168.20.243:5432
#a1.sinks.k1.port = 5432
a1.sinks.k1.databaseName = game_log
a1.sinks.k1.tableName = zl_log
a1.sinks.k1.user = game
a1.sinks.k1.password = game123
a1.sinks.k1.serverId = 1
a1.sinks.k1.channel = c1
a1.sinks.k1.josnTo = ture  

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
```

重点在：

```properties
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /home/zl/xsvr/server/xgame_1/logs/act/zl_war.log
```

支持比配多个文件方式

`a1.sources.tailsource-1.command = for i in /path/*.log; do cat $i; done`

# 2. flume监控目录
对指定目录进行实时监控，如发现目录新增文件，立刻收集并发送
缺点：不能对目录文件进行修改，如果有追加内容的文本文件，不允许，
配置示例：

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /home/rui/log/flumespool
a1.sources.r1.fileHeader = true

# Describe the sink
a1.sinks.k1.type = com.flume.dome.mysink.DBsqlSink
a1.sinks.k1.hostname = jdbc:postgresql://192.168.20.243:5432
#a1.sinks.k1.port = 5432
a1.sinks.k1.databaseName = game_log
a1.sinks.k1.tableName = zl_log
a1.sinks.k1.user = game
a1.sinks.k1.password = game123
a1.sinks.k1.serverId = 4
a1.sinks.k1.channel = c1
a1.sinks.k1.josnTo = true 

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 1000
a1.channels.c1.transactionCapacity = 100
```

重点在于 ：

```properties
a1.sources.r1.type = spooldir
a1.sources.r1.spoolDir = /home/rui/log/flumespool
```

# 3.flume监控目录，支持文件修改，并记录文件状态

TAILDIR  flume 1.7目前最新版新增类型，支持目录变化的文件，如遇中断，并以json数据记录目录下的每个文件的收集状态，
目前我们收集日志方式，已升级为TAILDIR  
配置示例：

```properties
# Name the components on this agent
a1.sources = r1
a1.sinks = k1
a1.channels = c1

# Describe/configure the source
a1.sources.r1.type = TAILDIR
a1.sources.r1.channels = c1
a1.sources.r1.positionFile = /home/web_admin/opt/v2_flume-apache170/logfile_stats/x1/taildir_position.json  
a1.sources.r1.filegroups = f1                          
a1.sources.r1.filegroups.f1 = /home/zl/xsvr/server/xgame_1/logs/act/zl_war.log  
a1.sources.r1.headers.f1.headerKey1 = value1             
a1.sources.r1.fileHeader = true


# Describe the sink
a1.sinks.k1.type = com.flume.dome.mysink.DBsqlSink
a1.sinks.k1.hostname = jdbc:postgresql://192.168.20.243:5432
#a1.sinks.k1.port = 5432
a1.sinks.k1.databaseName = game_log
a1.sinks.k1.tableName = zl_log_info
a1.sinks.k1.user = game
a1.sinks.k1.password = game123
a1.sinks.k1.serverId = 1
a1.sinks.k1.channel = c1
a1.sinks.k1.josnTo = true

# Use a channel which buffers events in memory
a1.channels.c1.type = memory
a1.channels.c1.capacity = 5000
a1.channels.c1.transactionCapacity = 5000
```

重点在：

```properties
a1.sources.r1.type = TAILDIR
a1.sources.r1.channels = c1
a1.sources.r1.positionFile = /home/web_admin/opt/v2_flume-apache170/logfile_stats/x1/taildir_position.json   //记录收集状态
a1.sources.r1.filegroups = f1
a1.sources.r1.filegroups.f1 = /home/zl/xsvr/server/xgame_1/logs/act/zl_war.log   //目录地址，支持单个文件或正则匹配如 dir/* 或 dir/.*log.* 
a1.sources.r1.headers.f1.headerKey1 = value1
a1.sources.r1.fileHeader = true
```

flume启动脚本:

```bash
#!/bin/bash
nohup ../bin/flume-ng agent --conf ../conf --conf-file ../conf/x1_dir_to_db_flume.conf --name a1 -Dflume.root.logger=INFO,console > x1nohup.out 2>&1 &
```

写在sh脚本，方便启动维护，以后台进程方式启动，并把日志输出到指定文件中，方便查看日志和调试

级采用 supervisor 来维护所有项目的后台进程