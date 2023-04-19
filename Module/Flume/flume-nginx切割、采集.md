来源：https://www.cnblogs.com/tonychai/p/4528478.html

nginx的日志文件没有rotate功能。如果你不处理，日志文件将变得越来越大，还好我们可以写一个nginx日志切割脚本来自动切割日志文件。

第一步：重命名日志文件，不用担心重命名后nginx找不到日志文件而丢失日志。在你未重新打开原名字的日志文件前，nginx还是会向你重命名的文件写日志，linux是靠文件描述符而不是文件名定位文件。

第二步：nginx主进程发送USR1信号。
nginx主进程接到信号后会从配置文件中读取日志文件名称，重新打开日志文件(以配置文件中的日志名称命名)，并以工作进程的用户作为日志文件的所有者。
重新打开日志文件后，nginx主进程会关闭重名的日志文件并通知工作进程使用新打开的日志文件。
工作进程立刻打开新的日志文件并关闭重名名的日志文件。
然后你就可以处理旧的日志文件了。

nginx_log_division.sh代码：

```bash
#!/bin/bash    
#设置日志文件存放目录  
logs_path="/usr/local/nginx/nginxlog/"  
#设置pid文件  
pid_path="/usr/local/nginx/nginx-1.7.3/logs/nginx.pid"  
#日志文件    
filepath=${logs_path}"access.log"  
# Source function library.    
#重命名日志文件  
mv ${logs_path}access.log ${logs_path}access_$(date -d '-1 day' '+%Y-%m-%d').log  
#向nginx主进程发信号重新打开日志  
kill -USR1 `cat ${pid_path}`
```

保存以上脚本nginx_log.sh，设置定时执行。

设置上面的shell脚本文件加入到定时任务中去。crontab是linux下面一个定时任务进程。开机此进程会启动，它每隔一定时间会去自己的列表中看是否有需要执行的任务。


`crontab  -e`
```bash
0 0 * * * /data/wwwlogs/nginx_log_division.sh
```

会打开一个文件，加入上面的代码

格式为 "分 时 日 月 星期几  要执行的shell文件路径"。用*可以理解成“每”,每分钟，每个小时，每个月等等。

我设置是在凌晨0点0分运行nginx_log_division.sh，脚本放到flume中bin文件夹下，脚本的内容就是重新生成一个新的日志文件。

flume-ng配置：

```properties
# A single-node Flume configuration  
  
# Name the components on this agent  
  
agent1.sources = source1  
agent1.sinks = sink1  
agent1.channels = channel1  
  
     
# Describe/configure source1  
  
agent1.sources.source1.type = exec  
agent1.sources.source1.command = tail -n +0 -F /logs/access.log  
agent1.sources.source1.channels = channel1  
  
  
# Describe sink1  
agent1.sinks.sink1.type = file_roll  
agent1.sinks.sink1.sink.directory=/var/log/data  
  
# Use a channel which buffers events in memory  
agent1.channels.channel1.type = file  
agent1.channels.channel1.checkpointDir=/var/checkpoint  
agent1.channels.channel1.dataDirs=/var/tmp  
agent1.channels.channel1.capacity = 1000  
agent1.channels.channel1.transactionCapactiy = 100  
   
  
# Bind the source and sink to the channel  
agent1.sources.source1.channels = channel1  
agent1.sinks.sink1.channel = channel1  
```
