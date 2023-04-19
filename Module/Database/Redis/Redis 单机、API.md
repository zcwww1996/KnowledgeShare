[TOC]
# Redis 单机、API
## 单机版安装

1. 下载redis3的稳定版本，下载地址 http://download.redis.io/releases/redis-3.2.11.tar.gz

2. 上传redis-3.2.11.tar.gz到服务器

3. 解压redis源码包

`tar -zxvf redis-3.2.11.tar.gz -C /usr/local/src/`

4. 进入到源码包中，编译并安装redis

`cd /usr/local/src/redis-3.2.11/`

`make && make install`

5. 报错，缺少依赖的包

缺少gcc依赖（c的编译器）

6. 配置本地YUM源并安装redis依赖的rpm包

`yum -y install gcc`

7. 编译并安装

`make && make install`

8. 报错，原因是没有安装jemalloc内存分配器，可以安装jemalloc或直接输入

`make MALLOC=libc && make install`

9. 重新编译安装

`make MALLOC=libc && make install`

10. 在所有机器的/usr/local/下创建一个redis目录，然后拷贝redis自带的配置文件redis.conf到/usr/local/redis

`mkdir /usr/local/redis`

`cp /usr/local/src/redis-3.2.11/redis.conf /usr/local/redis`

11. 修改所有机器的配置文件redis.conf

```shell
daemonize yes  #redis后台运行
appendonly yes  #开启aof日志，它会每次写操作都记录一条日志
bind 192.168.134.113
requirepass foobared 改为requirepass 123456
```
12. 启动所有的redis节点

`cd /usr/local/redis`

`redis-server redis.conf`

13. 查看redis进程状态

`ps -ef | grep redis`

14. 使用命令行客户的连接redis

`redis-cli -p 6379`

**redis-cli -h 192.168.134.113 -p 6379 -a 123456**

需要密码认证
`auth 123456`

15. 关闭redis

`redis-cli shutdown`

绑定了ip后<br/>
`redis-cli -h 192.168.134.113 shutdown`

远程关闭redis<br/>
**redis-cli -h 192.168.134.113 -p 6379 -a 123456 shutdown**

## Redis淘汰机制(Eviction policies)

首先，需要设置最大内存限制

```bash
maxmemory 100mb
```

选择策略

```bash
maxmemory-policy noeviction
```

解释：
- noeviction：默认策略，不淘汰，如果内存已满，添加数据是报错。
- allkeys-lru：在所有键中，选取最近最少使用的数据抛弃。
- volatile-lru：在设置了过期时间的所有键中，选取最近最少使用的数据抛弃。
- allkeys-random：在所有键中，随机抛弃。
- volatile-random： 在设置了过期时间的所有键，随机抛弃。
- volatile-ttl：在设置了过期时间的所有键，抛弃存活时间最短的数据。

## Linux命令
```shell
SELECT 3 切换到指定的数据库3
DBSIZE 返回当前数据库的key的数量
info clients可以查看当前的redis连接数
```
## 客户端API
### String
#### Set/Get

Set：设置指定 key 的值

Get：获取指定 key 的值。
```java
jedis.set("key", "value");
String v = jedis.get("key");
System.out.println("结果:" + v);

结果:value</span>
```
#### setex：设置超时时间
将值 value 关联到 key ，并将 key 的过期时间设为 seconds (以秒为单位)。

```java
String v1 = jedis.get("key1");
System.out.println("结果:" + v1);
Thread.sleep(2000);
v1 = jedis.get("key1");
System.out.println("结果:" + v1);
结果:value1
结果:null</span>
```
#### incr/incrBy
incr：将 key 中储存的数字值增一。<br/>
incrBy：将 key 所储存的值加上给定的增量值（increment）。<br/>
```java
jedis.incr("key3");
String v3 = jedis.get("key3");
System.out.println("结果:" + v3);
jedis.incrBy("key3", 5);
v3 = jedis.get("key3");
System.out.println("结果:" + v3);

结果:1
结果:6
```
#### decr/decrBy
decr：将 key 中储存的数字值减一。<br/>
decrBy：key 所储存的值减去给定的减量值（decrement）<br/>

```java
jedis.decr("key4");
String v4 = jedis.get("key4");
System.out.println("结果:" + v4);
jedis.decrBy("key4", 5);
v4 = jedis.get("key4");
System.out.println("结果:" + v4);

结果:-1
结果:-6
```
### Hash
#### hset/hget
hset：将哈希表 key 中的字段 field 的值设为 value 。<br/>
hget：获取存储在哈希表中指定字段的值。

```java
jedis.hset("key5", "name", "zhangsan");
jedis.hset("key5", "age", "15");
jedis.hset("key5", "sex", "boy");
String sexValue = jedis.hget("key5","sex");
System.out.println("结果:" + sexValue);
结果:boy
```
#### hkeys/hvals
hkeys：返回 hash 的所有 field<br/>
hvals：返回 hash 的所有 value

共同点：当key不存在时，返回一个空表

```shell
hkeys myhash

1) "field2"
2) "field"
3) "field3"

hvals myhash

1) "World"
2) "Hello"
3) "12"
```

#### hdel/hgetAll
hdel：删除一个或多个哈希表字段<br/>
hgetAll：获取在哈希表中指定 key 的所有字段和值

```java
jedis.hset("key5", "sex2", "girl");
jedis.hdel("key5", "sex");
Map<String, String> hgetAll = jedis.hgetAll("key5");
for (Map.Entry<String, String> map : hgetAll.entrySet()) {
    System.out.println("key:" + map.getKey() + ",value:" + map.getValue());
}

key:age,value:15
key:name,value:zhangsan
key:sex2,value:girl
```
#### hmset/hmget
hmset：同时将多个 field-value (字段-值)对插入或更新（覆盖）到哈希表中。如果哈希表不存在，会创建一个空哈希表，并执行 HMSET 操作。

hmget：获取哈希表中全部指定的filed(字段)，如果不存在，那么返回一个 nil值。

```java
Map map = new HashMap<String, String>();
map.put("name", "lisi");
map.put("age", "15");
map.put("sex", "girl");
jedis.hmset("key6", map);
if(jedis.hexists("key6", "name")){
    List<String> list = jedis.hmget("key6", "name");
    System.out.println("结果为：" + list.get(0));
}
结果为：lisi
```