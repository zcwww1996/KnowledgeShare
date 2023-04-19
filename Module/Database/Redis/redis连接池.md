[TOC]
# redis普通连接池

```scala
import com.typesafe.config.ConfigFactory
import org.apache.commons.pool2.impl.GenericObjectPoolConfig
import redis.clients.jedis.{Jedis, JedisPool}

/**
  * jedil连接池
  */
object JPools {

    /**
      * 默认加载application.conf -> properties -> json
      */
    lazy val config = ConfigFactory.load()

    lazy val poolConfig = new GenericObjectPoolConfig()
    poolConfig.setMaxTotal(1000)

    lazy val jedisPool = new JedisPool(poolConfig, config.getString("redis.host"), config.getInt("redis.port"))


    def getJedis: Jedis = {
        jedisPool.getResource
    }


    /**
      * 程序定制之前销毁连接池
      */
    Runtime.getRuntime.addShutdownHook(new Thread(new Runnable {
        override def run(): Unit = jedisPool.destroy()
    }))

}
```

# redis切片连接池

```scala
import java.util

import redis.clients.jedis.{JedisPoolConfig, JedisShardInfo, ShardedJedis, ShardedJedisPool}


object JPools {

    //1.实例化Jedis分片对象，并放入到一个List当中
    val list = new util.ArrayList[JedisShardInfo]
    list.add(new JedisShardInfo("192.168.245.111", 6381))
    list.add(new JedisShardInfo("192.168.245.111", 6382))
    list.add(new JedisShardInfo("192.168.245.111", 6383))

    //2.实例化Jedis连池配置对象
    val config: JedisPoolConfig = new JedisPoolConfig
    config.setMaxTotal(10)
    config.setMaxWaitMillis(30)

    //3.实例化Jedis连接池
    val jedisPool: ShardedJedisPool = new ShardedJedisPool(config, list)

    def getJedis: ShardedJedis = {
        jedisPool.getResource
    }

}
```

