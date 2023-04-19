[TOC]
# 1. 磁盘容量不足

**【现象】** 查看到yarn监控页面上有十几个Unhealthy 节点，分别进去Unhealthy Nodes查看各个目录的占用磁盘情况，发现是HDFS的有关目录占用过多了，这是因为有很多临时文件占用了Hdfs。

```bash
tmp_users=`hdfs dfs -ls /tmp/ | awk '{print $8}' | cut -d"/" -f3 | xargs `
[root@wps-datanode-051 conf.empty]# echo $tmp_users 	# 总共有243个用户使用hive on yarn
```

## 1.1 批量清理每个用户在hdfs临时目录下产生的文件

```bash
for user in $tmp_users; do hdfs dfs -rm -r /tmp/${user}/.staging/*; done
```

日志输出：

> ...
> 18/07/30 20:36:30 INFO fs.TrashPolicyDefault: Moved: 'hdfs://hdfs-ha/tmp/mengfanhui/.staging/job_1523945420036_927376' to trash at: hdfs://hdfs-ha/user/root/.Trash/Current/tmp/guojingjing/.staging/job_1523945420036_927376
> ...

此时，同时查看yarn页面的Unhealthy Node数目，会有变化：随着清理的进行，Unhealthy Node数目会越来越小，从一开始的19个到12个、5个…，这是因为那些不健康的node manager节点恢复到了默认yarn配置项yarn.nodemanager.disk-health-checker.min-healthy-disks的90%以下。

## 1.2 如果有spark on yarn的作业，那么需要清理过期文件
过期文件可以从spark-defaults.conf中的属性spark.eventLog.dir中获得。我本机的目录是在HDFS上的/spark2-history/

## 1.3 需要清理HDFS上的回收站

```bash
for user in $tmp_users; do hdfs dfs -rm -r /user/root/.Trash/Current/tmp/${user}/.staging/*; echo CLEANED ${user} Trash on HDFS; done
```

日志输出：

> Deleted /user/root/.Trash/Current/tmp/fujiaqi/.staging/job_1523945420036_698345
> Deleted /user/root/.Trash/Current/tmp/fujiaqi/.staging/job_1523945420036_886570
> CLEANED fujiaqi Trash on HDFS

清理后，发现一个规律：hdfs上可能出现空间不足的目录，是在hive-site.xml中的属性hive.exec.scratchdir的值，它指定HDFS中hive临时文件存放的目录，当通过hive提交作业到yarn运行时产生的日志就放在该目录，当启动hive时HDFS即自动创建该目录。我这里是/mnt/log/hive/scratch/，只要清理它下面的过期临时文件即可，清理后的过期文件会被移到 /user/root/.Trash/Current/mnt/log/hive/scratch/ 目录下，所以此Trash目录也要清理。

```bash
#开始使用批量脚本清理
[root@wps-datanode-009 ~]# tobe_cleaned=`hdfs dfs -du -h /user/root/.Trash/Current/mnt/log/hive/scratch/ | cut -d"/" -f 10 | xargs`
[root@wps-datanode-009 ~]# for user in $tobe_cleaned; do hdfs dfs -rm -r /user/root/.Trash/Current/mnt/log/hive/scratch/${user}/*; echo Finished cleaning ${user} ______; done
```

输出日志节选：
> ... ...
> Deleted /user/root/.Trash/Current/mnt/log/hive/scratch/zouyinfeng/d4b36da0-c8bd-4098-8396-3b721b18d4e8
> Deleted /user/root/.Trash/Current/mnt/log/hive/scratch/zouyinfeng/d5282303-bd5e-4ed0-b1c7-dd7b9ff9c54c
> Deleted /user/root/.Trash/Current/mnt/log/hive/scratch/zouyinfeng/e5828d94-abaa-4303-b731-c1647339f2cc
> Deleted /user/root/.Trash/Current/mnt/log/hive/scratch/zouyinfeng/f2bdb19a-31ac-416c-9914-2cd90d83d4db
> Finished cleaning tony______
> ... ...

**删除掉的文件被移到了回收站里，需要进一步设置回收站的文件保留时间，以便迅速从回收站移除，从而彻底清理**

清理完以后，检查/mnt目录的磁盘空间使用率发现已经在90%以下了。

清理完以后，发现YARN 8088页面的 wps-datanode-003 仍然处于Unhealthy状态，于是去配置文件查看有关yarn的磁盘健康检查的间隔时长，也就是 yarn.nodemanager.disk-health-checker.interval-ms 这个配置项，但是我配置文件里没有这个项，我没配置，这就意味着定期检查的间隔时长默认为2分钟。

过了2分钟后，我再刷新yarn 8088页面，仍然显示Unhealthy，这是什么原因呢？

联想到resource manager(RM)是负责向yarn 8088页面提供每个节点健康数据的，而每个节点的健康信息是由各节点的node manager(NM)上报给RM的，此时还未来得及上报给RM。所以只要重启NodeManager即可，NM会重新把本节点的健康情况上报给RM。重启后再查看yarn 8088页面，刚才Unhealthy状态的节点消失，问题解决。

具体重启NM的流程如下:

```bash
[root@wps-datanode-003 sbin]# pwd
/usr/hadoop/hadoop-yarn/sbin
[root@wps-datanode-003 sbin]# ls
yarn-daemon.sh  yarn-daemons.sh
[root@wps-datanode-003 sbin]# jps
4354 Jps
20182 DataNode
23018 NodeManager
[root@wps-datanode-003 sbin]# kill -9 23018
[root@wps-datanode-003 sbin]# ./yarn-daemon.sh start nodemanager
starting nodemanager, logging to /mnt/log/hadoop-yarn/root/yarn-root-nodemanager-wps-datanode-003.out
[root@wps-datanode-003 sbin]# jps
4547 Jps
20182 DataNode
4463 NodeManager
```

再尝试提交一个大作业：查询一个月的数据量

[通过hue的交互页面提交到Hive On Yarn]
```sql
SELECT dt, ft1, ft2, ft4, ft5, count(1) AS pv, count(distinct hdid) AS uv
FROM mail_logs.`senders_mail`
WHERE dt BETWEEN '2018-07-01'
      AND '2018-07-31'
      AND ft4 = 'preview'
      AND ft5 = 'display'
GROUP BY dt, ft1,ft2, ft4, ft5
```
**[结果]** 之前资源不足(磁盘空间不够)需要55分钟跑完的，这次20分钟就跑完了

# 2. 资源占用

如果经过上述步骤清理了一遍，但是UnhealthyNodes数目仍然大于0，那么就需要去检查一下yarn页面中正在running的大作业，大作业就是占用资源过多或者执行时间过长的作业：

[![](https://ftp.bmp.ovh/imgs/2019/12/4281ac76ebec2c36.png)](https://upload.cc/i1/2019/12/04/zHfDNE.png)

对这类作业的分析：

1. 很可能是由聚合算子导致了数据倾斜，大量数据堆积在少数节点，这个可以通过在命令行杀掉作业：yarn application -kill [your_application_id]
2. 也可能由于处理的数据量过大导致产生过多的临时文件，而作业仍然在running,大量的临时文件无法及时释放,从而对计算节点的/mnt磁盘占用过多，一旦占用超过yarn-site.xml中的yarn.nodemanager.disk-health-checker.min-healthy-disks默认90%临界值，就被标记成Unhealthy状态，就不会给该节点分配spark on yarn的计算任务。解决办法请参考本文第一部分。

原文链接：https://blog.csdn.net/qq_31598113/article/details/81316411