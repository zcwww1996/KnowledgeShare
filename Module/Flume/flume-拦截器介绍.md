- [flume学习（四）：Flume Interceptors的使用](https://blog.csdn.net/xiao_jun_0820/article/details/38111305)
- [Flume中的拦截器（Interceptor）介绍与使用（一）](https://blog.csdn.net/jek123456/article/details/65633958)
- [Interceptors - 拦截器](https://www.cnblogs.com/zpb2016/p/5766939.html)

官方上提供的已有的拦截器有：
flume的拦截器也是chain形式的，可以对一个source指定多个拦截器，按先后顺序依次处理。
- **Timestamp Interceptor** :在event的header中添加一个key叫：timestamp,value为当前的时间戳（毫秒）。这个拦截器在sink为hdfs 时很有用

```java
hdfs.path = hdfs://cdh5/tmp/dap/%Y%m%d
hdfs.filePrefix = log_%Y%m%d_%H
```
会根据时间戳将数据写入相应的文件中。

但可以用其他方式代替（设置useLocalTimeStamp = true）。
- **Host Interceptor**：在event的header中添加一个key叫：host,value为当前机器的hostname或者ip。

```properties
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = host
a1.sources.r1.interceptors.i1.useIP = false
a1.sources.r1.interceptors.i1.hostHeader = agentHost
```
- **Static Interceptor**:可以在event的header中添加自定义的key和value。

```properties
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type = static
a1.sources.r1.interceptors.i1.preserveExisting = true
a1.sources.r1.interceptors.i1.key = static_key
a1.sources.r1.interceptors.i1.value = static_value
```

- **UUID Interceptor**:用于在每个events header中生成一个UUID字符串，例如：b5755073-77a9-43c1-8fad-b7a586fc1b97。生成的UUID可以在sink中读取并使用。

```properties
# 使flume的log均匀的下沉到各个分区
a1.sources.r1.interceptors = i1
a1.sources.r1.interceptors.i1.type=org.apache.flume.sink.solr.morphline.UUIDInterceptor$Builder
a1.sources.r1.interceptors.i1.headerName=key
a1.sources.r1.interceptors.i1.preserveExisting=false
```

- **Regex Filtering Interceptor**:通过正则来清洗或包含匹配的events。
- **Regex Extractor Interceptor**：通过正则表达式来在header中添加指定的key,value则为正则匹配的部分