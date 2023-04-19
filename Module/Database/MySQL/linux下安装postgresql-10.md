https://blog.csdn.net/baidu_22576911/article/details/106922640?%3E


Linux下PostgreSQL开机启动配置方法
将启动命令加入到初始化脚本中

将su postgres -c 'pg_ctl start -D /usr/local/pgsql/data -l /usr/local/pgsql/server.log' 添加到/etc/rc.local中

```bash
vi /etc/rc.local
#!/bin/sh
#
# This script will be executed *after* all the other init scripts.
# You can put your own initialization stuff in here if you don't
# want to do the full Sys V style init stuff.

touch /var/lock/subsys/local
su postgres -c 'pg_ctl start -D /usr/local/pgsql/data -l server.log'
```


参考资料：https://www.postgresql.org/docs/10/static/server-start.html