[TOC]

# 1.前言
先来回顾一下pipeline的原理，redis client与server之间采用的是请求应答的模式，如下所示：

```bash
Client: command1 
Server: response1 
Client: command2 
Server: response2 
…
```

在这种情况下，如果要完成10个命令，则需要20次交互才能完成。因此，即使redis处理能力很强，仍然会受到网络传输影响，导致吞吐量上不去。而在管道（pipeline）模式下，多个请求可以变成这样：

```bash
Client: command1，command2… 
Server: response1，response2…
```


在这种情况下，完成命令只需要2次交互。这样网络传输上能够更加高效，加上redis本身强劲的处理能力，给数据处理带来极大的性能提升。但实际上遇到的问题是，项目上所用到的是Redis集群，初始化的时候使用的类是JedisCluster而不是Jedis。去查了JedisCluster的文档，并没有发现提供有像Jedis一样的获取Pipeline对象的 pipelined()方法。

## 1.1 为什么RedisCluster无法使用pipeline?

我们知道，Redis 集群的键空间被分割为 16384 个槽（slot），集群的最大节点数量也是 16384 个。每个主节点都负责处理 16384 个哈希槽的其中一部分。具体的redis命令，会根据key哈希取模计算出一个槽位slot(CRC16(key) mod 16384),然后根据槽位去特定的节点redis上执行操作。如下所示：


```bash
master1（slave1）： 0~5460
master2（slave2）：5461~10922
master3（slave3）：10923~16383
```


集群有三个master节点组成，其中master1分配了 0~5460的槽位，master2分配了 5461~10922的槽位，master3分配了 10923~16383的槽位。

一次pipeline会批量执行多个命令，那么每个命令都需要根据“key”运算一个槽位（JedisClusterCRC16.getSlot(key)），然后根据槽位去特定的机器执行命令，也就是说一次pipeline操作会使用多个节点的redis连接，而目前JedisCluster是无法支持的。

如果还想要使用批量读取应该怎么办呢？目前了解到的解决方法有两种:
- 一种是使用hash_tag模式读写。简单说就是使用”{}”来将要hash的key的部分包裹起来，rediscluster写入数据时只会对key中被”{}”包裹部分进行哈希取模计算slot位置。即存入时使用 “a{123}”和”b{123}”是在同一个slot上。这样就可以批量读取存放在同一个slot上的数据。

- 第二种方法是在批量读取时，先计算所有数据的存放节点。具体做法是，我们已经知道了redisCluster对数据哈希取模的算法，可以先计算数据存放的slot位置，然后我们又可以很容易知道每个节点分管的slot段。这样，我们就可以通过key来计算出数据存放在哪个节点上。然后根据不同的节点将数据分成多批。对不同批的数据进行分批pipeline处理。


# 2. 代码
## 2.1 redisCluster对数据哈希取模的算法
参考：
http://www.bubuko.com/infodetail-1106789.html<br>
https://my.oschina.net/u/1266221/blog/894308


核心部分是getBatch、initJedisNodeMap、getJedisByKey、initSlotHostMap 这几个方法，其中getBatch是主入口
```java

    /**
     * JedisPool和Keys的映射关系
     */
    private Map<JedisPool, ArrayList<String>> jedisPoolKeysMap;


    /**
     * 批量查询数据
     *
     * @param keys
     * @return
     */
    public Map<String, Object> getBatch(String... keys) {

        // 返回的结果，包括正确的keys的value的集合；和不存在的keys的集合
        Map<String, Object> result = new HashMap<>(16);

        // 正确的keys的value的集合
        Map<String, Map<String, Double>> existResult = new HashMap<>(16);

        // 错误的key的集合
        Set<String> errorKeys = new HashSet<>(16);



        // JedisPool和Keys的映射关系
        jedisPoolKeysMap = new HashMap<JedisPool, ArrayList<String>>();

        for (String key : keys) {
            JedisPool jedisPool = getJedisByKey(key);
            if (jedisPool == null){
                continue;
            }
            if (!jedisPoolKeysMap.keySet().contains(jedisPool)) {
                ArrayList<String> keysList = new ArrayList<>();
                keysList.add(key);
                jedisPoolKeysMap.put(jedisPool, keysList);
            } else {
                ArrayList<String> keysList = jedisPoolKeysMap.get(jedisPool);
                keysList.add(key);
            }
        }

        for (JedisPool jedisPool : jedisPoolKeysMap.keySet()) {
            Jedis jedis = jedisPool.getResource();

            Pipeline pipeline = jedis.pipelined();
            try {
                if (pipeline != null) {
                    pipeline.clear();

                    ArrayList<String> keysList = jedisPoolKeysMap.get(jedisPool);
                    for (String key : keysList) {
                        pipeline.get(key);
                    }
                    List<Object> results = pipeline.syncAndReturnAll();

                    for (int index = 0; index < results.size(); index++) {
                        if (results.get(index) == null) {
                            errorKeys.add(keysList.get(index));
                        } else {
                            existResult.put(keysList.get(index), stringToMap(results.get(index).toString()));
                        }
                    }
                }
                returnResource(jedis);
            }catch(Exception e){
                e.printStackTrace();
            }
        }
        result.put("error", errorKeys);
        result.put("exist", existResult);
        return result;
    }


    /**
     * 通过key来获取对应的jedisPool对象
     *
     * @param key
     * @return
     */
    public JedisPool getJedisByKey(String key) {
        int slot = JedisClusterCRC16.getSlot(key);
        Map.Entry<Long, String> entry;
        entry = getSlotHostMap().lowerEntry(Long.valueOf(slot + 1));

        if(entry == null){
            logger.error("entry为空!!!!! key为：" + key + "，slot为：" + slot);
            return null;
        }
        return historyCtrJedisClusterBatchUtil.getNodeMap().get(entry.getValue());
    }



    /**
     * 返还到连接池
     *
     * @param jedis
     */
    public static void returnResource(Jedis jedis) {
        if (jedis != null) {
            jedis.close();
        }
    }

    /**
     * 节点映射关系
     */
    private Map<String, JedisPool> nodeMap;

    /**
     * slot和host之间的映射
     */
    private TreeMap<Long, String> slotHostMap;


    /**
     * 初始化JedisNodeMap
     */
    private void initJedisNodeMap() {
        try {
            nodeMap = historyCtrJedisClusterFactory.getClusterNodes();
            String anyHost = nodeMap.keySet().iterator().next();
            initSlotHostMap(anyHost);
        } catch (JedisClusterException e) {
            logger.error(e.getMessage());
        }
    }

    /**
     * 获取slot和host之间的对应关系
     *
     * @param anyHostAndPortStr
     * @return
     */
    private void initSlotHostMap(String anyHostAndPortStr) {
        TreeMap<Long, String> tree = new TreeMap<Long, String>();
        String parts[] = anyHostAndPortStr.split(":");
        HostAndPort anyHostAndPort = new HostAndPort(parts[0], Integer.parseInt(parts[1]));
        try {
            Jedis jedis = new Jedis(anyHostAndPort.getHost(), anyHostAndPort.getPort());
            List<Object> list = jedis.clusterSlots();
            for (Object object : list) {
                List<Object> list1 = (List<Object>) object;
                List<Object> master = (List<Object>) list1.get(2);
                String hostAndPort = new String((byte[]) master.get(0)) + ":" + master.get(1);
                tree.put((Long) list1.get(0), hostAndPort);
                tree.put((Long) list1.get(1), hostAndPort);
            }
            jedis.close();
        } catch (Exception e) {
            e.printStackTrace();
        }
        slotHostMap = tree;
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        initJedisNodeMap();
    }
```

## 2.2 利用JedisClusterConnectionHandler

将一个JedisCluster下的pipeline分解为每个单节点下独立的jedisPipeline操作，最后合并response返回。具体实现就是通过JedisClusterCRC16.getSlot(key)计算key的slot值，通过每个节点的slot分布，就知道了哪些key应该在哪些节点上。再获取这个节点的JedisPool就可以使用pipeline进行读写了。
实现上面的过程可以有很多种方式，本文将介绍一种也许是代码量最少的一种解决方案

其实在JedisClusterInfoCache对象中都已经帮助开发人员实现了，但是这个对象在JedisClusterConnectionHandler中为protected并没有对外开放，而且通过JedisCluster的API也无法拿到JedisClusterConnectionHandler对象。所以通过下面两个类将这些对象暴露出来，这样使用getJedisPoolFromSlot就可以知道每个key对应的JedisPool了。

JedisClusterPipeline


```java
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisCluster;

import java.util.Set;

public class JedisClusterPipeline extends JedisCluster {
    public JedisClusterPipeline(Set<HostAndPort> jedisClusterNode, int connectionTimeout, int soTimeout, int maxAttempts, String password, final GenericObjectPoolConfig poolConfig) {
        super(jedisClusterNode,connectionTimeout, soTimeout, maxAttempts, password, poolConfig);
        super.connectionHandler = new JedisSlotAdvancedConnectionHandler(jedisClusterNode, poolConfig,
                connectionTimeout, soTimeout ,password);
    }

    public JedisSlotAdvancedConnectionHandler getConnectionHandler() {
        return (JedisSlotAdvancedConnectionHandler)this.connectionHandler;
    }

    /**
     * 刷新集群信息，当集群信息发生变更时调用
     * @param
     * @return
     */
    public void refreshCluster() {
        connectionHandler.renewSlotCache();
    }
}
```

JedisSlotAdvancedConnectionHandler

```java
import org.apache.commons.pool2.impl.GenericObjectPoolConfig;
import redis.clients.jedis.HostAndPort;
import redis.clients.jedis.JedisPool;
import redis.clients.jedis.JedisSlotBasedConnectionHandler;
import redis.clients.jedis.exceptions.JedisNoReachableClusterNodeException;

import java.util.Set;

public class JedisSlotAdvancedConnectionHandler extends JedisSlotBasedConnectionHandler {

    public JedisSlotAdvancedConnectionHandler(Set<HostAndPort> nodes, GenericObjectPoolConfig poolConfig, int connectionTimeout, int soTimeout,String password) {
        super(nodes, poolConfig, connectionTimeout, soTimeout, password);
    }

    public JedisPool getJedisPoolFromSlot(int slot) {
        JedisPool connectionPool = cache.getSlotPool(slot);
        if (connectionPool != null) {
            // It can't guaranteed to get valid connection because of node
            // assignment
            return connectionPool;
        } else {
            renewSlotCache(); //It's abnormal situation for cluster mode, that we have just nothing for slot, try to rediscover state
            connectionPool = cache.getSlotPool(slot);
            if (connectionPool != null) {
                return connectionPool;
            } else {
                throw new JedisNoReachableClusterNodeException("No reachable node in cluster for slot " + slot);
            }
        }
    }
}
```

编写测试类，向redis集群写入10000条数据，分别测试调用普通JedisCluster模式和调用上面实现的JedisCluster Pipeline模式的性能对比，测试类如下：


```java
import redis.clients.jedis.*;
import redis.clients.util.JedisClusterCRC16;
import java.io.UnsupportedEncodingException;
import java.util.*;

public class PipelineTest {
    public static void main(String[] args) throws UnsupportedEncodingException {
        PipelineTest client = new PipelineTest();
        Set<HostAndPort> nodes = new HashSet<>();
        nodes.add(new HostAndPort("node1",20249));
        nodes.add(new HostAndPort("node2",20508));
        nodes.add(new HostAndPort("node3",20484));
        String redisPassword = "123456";
        //测试
        client.jedisCluster(nodes,redisPassword);
        client.clusterPipeline(nodes,redisPassword);
    }
    //普通JedisCluster 批量写入测试
    public void jedisCluster(Set<HostAndPort> nodes,String redisPassword) throws UnsupportedEncodingException {
        JedisCluster jc = new JedisCluster(nodes, 2000, 2000,100,redisPassword, new JedisPoolConfig());
        List<String> setKyes = new ArrayList<>();
        for (int i = 0; i < 10000; i++) {
            setKyes.add("single"+i);
        }
        long start = System.currentTimeMillis();
        for(int j = 0;j < setKyes.size();j++){
            jc.setex(setKyes.get(j),100,"value"+j);
        }
        System.out.println("JedisCluster total time:"+(System.currentTimeMillis() - start));
    }
    //JedisCluster Pipeline 批量写入测试
    public void clusterPipeline(Set<HostAndPort> nodes,String redisPassword) {
        JedisClusterPipeline jedisClusterPipeline = new JedisClusterPipeline(nodes, 2000, 2000,10,redisPassword, new JedisPoolConfig());
        JedisSlotAdvancedConnectionHandler jedisSlotAdvancedConnectionHandler = jedisClusterPipeline.getConnectionHandler();
        Map<JedisPool, List<String>> poolKeys = new HashMap<>();
        List<String> setKyes = new ArrayList<>();
        for (int i = 0; i < 10000; i++) {
                setKyes.add("pipeline"+i);
        }
        long start = System.currentTimeMillis();
        //查询出 key 所在slot ,通过 slot 获取 JedisPool ,将key 按 JedisPool 分组
        jedisClusterPipeline.refreshCluster();
        for(int j = 0;j < setKyes.size();j++){
            String key = setKyes.get(j);
            int slot = JedisClusterCRC16.getSlot(key);
            JedisPool jedisPool = jedisSlotAdvancedConnectionHandler.getJedisPoolFromSlot(slot);
            if (poolKeys.keySet().contains(jedisPool)){
                List<String> keys = poolKeys.get(jedisPool);
                keys.add(key);
            }else {
                List<String> keys = new ArrayList<>();
                keys.add(key);
                poolKeys.put(jedisPool, keys);
            }
        }
        //调用Jedis pipeline进行单点批量写入
        for (JedisPool jedisPool : poolKeys.keySet()) {
            Jedis jedis = jedisPool.getResource();
            Pipeline pipeline = jedis.pipelined();
            List<String> keys = poolKeys.get(jedisPool);
            for(int i=0;i<keys.size();i++){
                pipeline.setex(keys.get(i),100, "value" + i);
            }
            pipeline.sync();//同步提交
            jedis.close();
        }
        System.out.println("JedisCluster Pipeline total time:"+(System.currentTimeMillis() - start));
    }
}

```

# 3. 总结
本文旨在介绍一种在Redis集群模式下提供Pipeline批量操作的功能。基本思路就是根据redis cluster对数据哈希取模的算法，先计算数据存放的slot位置, 然后根据不同的节点将数据分成多批，对不同批的数据进行单点pipeline处理。

但是需要注意的是，由于集群模式存在节点的动态添加删除，且client不能实时感知（只有在执行命令时才可能知道集群发生变更），因此，该实现不保证一定成功，建议在批量操作之前调用 refreshCluster() 方法重新获取集群信息。应用需要保证不论成功还是失败都会调用close() 方法，否则可能会造成泄露。如果失败需要应用自己去重试，因此每个批次执行的命令数量需要控制，防止失败后重试的数量过多。

基于以上说明，建议在集群环境较稳定（增减节点不会过于频繁）的情况下使用，且允许失败或有对应的重试策略。