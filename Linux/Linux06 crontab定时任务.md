[TOC]
# 1. crontab状态
CentOS默认会将crontab设置为开启的状态；

可查看其状态或开启：service crond status/start/stop/restart/reload

## 1.1 crontab任务
 可以用以下命令来操作crontab：
- crontab -u #设定某个用户的cron服务，一般root用户在执行这个命令的时候需要此参数,如果没有加上这个参数的话就会开启自己的crontab
- crontab -l #查看List 
- crontab -e #编辑Edit
- crontab -r #清空Remove

例子：

（1）root查看自己的cron设置:`crontab -u root -l`

（2）root想删除hadoop的cron设置:`crontab -u hadoop -r`
## 1.2 时间格式
```shell
crontab的规则（cat /etc/crontab）如下：
# Example of job definition:
# .---------------- minute (0 - 59)
# | .------------- hour (0 - 23)
# | | .---------- day of month (1 - 31)
# | | | .------- month (1 - 12) OR jan,feb,mar,apr ...
# | | | | .---- day of week (0 - 6) (Sunday=0 or 7) OR sun,mon,tue,wed,thu,fri,sat
# | | | | |
# * * * * * user-name command to be executed
```

从左至右，每个* 分别代表：分，时，日，月，周；最后为要执行的命令； 其中* / - , 这些通配符的含义为：

|通配符|含义|
|:---:|---|
| * |表示所有数值，如第一位使用 * 表示每分钟|
| / |表示每，如第一位使用 */5 表示每5分钟|
| - |表示数值范围，如第二位使用2-4表示2点到4点|
| , |表示离散的多个数值，如第2位使用6,8 表示6点和8点|
|指定"步长"|8-14/2 表示8，10，12，14|
|指定列表|比如 "1,2,3,4,10,11,12″->"0-4,10-12″|

# 2. cron的权限

**/etc/cron.allow**

**/etc/cron.deny**

(1)**系统首先判断是否有cron.allow这个文件**

(2)如果有这个文件的话，系统会判断这个使用者有没有在cron.allow的名单里面

(3)如果在名单里面的话，就可以使用cron机制。如果这个使用者没有在cron.allow名单里面的话，就不能使用cron机制。

(4)**如果系统里面没有cron.allow这个文件的话，系统会再判断是否有cron.deny这个文件**

(5)如果有cron.deny这个文件的话，就会判断这个使用者有没有在cron.deny这个名单里面

(6)如果这个使用者在cron.deny名单里面的话，将不能使用cron机制。如果这个使用者没有在cron.deny这个名单里面的话就可以使用cron机制。

(7)如果系统里这两个文件都没有的话，就可以使用cron机制

# 3. crontab文件格式

cron_root.cron文件内容

```bash
00 * */1 * * sh  /data/iptv/releaseCache.sh
```
执行cron文件

```bash
crontab cron_root.cron
```

可以通过`crontab -l`查看添加的定时任务