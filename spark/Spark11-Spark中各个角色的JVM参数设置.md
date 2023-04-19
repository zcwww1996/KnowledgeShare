备注：https://www.cnblogs.com/Scott007/p/3889959.html

## (1)Driver的JVM参数：

<font color="green">-Xmx，-Xms，如果是yarn-client模式，则默认读取spark-env文件中的SPARK_DRIVER_MEMORY值，-Xmx，-Xms值一样大小；如果是yarn-cluster模式，则读取的是spark-default.conf文件中的spark.driver.extraJavaOptions对应的JVM参数值。
</font>

<font color="DarkTurquoise">PermSize，如果是yarn-client模式，则是默认读取spark-class文件中的JAVA_OPTS="-XX:MaxPermSize=256m $OUR_JAVA_OPTS"值；如果是yarn-cluster模式，读取的是spark-default.conf文件中的spark.driver.extraJavaOptions对应的JVM参数值。
</font>

<font color="purple">GC方式，如果是yarn-client模式，默认读取的是spark-class文件中的JAVA_OPTS；如果是yarn-cluster模式，则读取的是spark-default.conf文件中spark.driver.extraJavaOptions对应的参数值。
</font>

<font color="red">**以上值最后均可被spark-submit工具中的--driver-java-options参数覆盖。**
</font>

## (2)Executor的JVM参数：
<font color="green">-Xmx，-Xms，如果是yarn-client模式，则默认读取spark-env文件中的SPARK_EXECUTOR_MEMORY值，-Xmx，-Xms值一样大小；如果是yarn-cluster模式，则读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。
</font>

<font color="DarkTurquoise">PermSize，两种模式都是读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。
</font>

<font color="purple">GC方式，两种模式都是读取的是spark-default.conf文件中的spark.executor.extraJavaOptions对应的JVM参数值。
</font>

## (3)Executor数目及所占CPU个数
<font color="orange">如果是yarn-client模式，Executor数目由spark-env中的SPARK_EXECUTOR_INSTANCES指定，每个实例的数目由SPARK_EXECUTOR_CORES指定；</font><font color="blue">如果是yarn-cluster模式，Executor的数目由spark-submit工具的--num-executors参数指定，默认是2个实例，而每个Executor使用的CPU数目由--executor-cores指定，默认为1核。</font>

每个Executor运行时的信息可以通过yarn logs命令查看到，类似于如下：

```java
14/08/13 18:12:59 INFO org.apache.spark.Logging$class.logInfo(Logging.scala:58): Setting up executor with commands: List($JAVA_HOME/bin/java, -server, -XX:OnOutOfMemoryError='kill %p', -Xms1024m -Xmx1024m , -XX:PermSize=256M -XX:MaxPermSize=256M -verbose:gc -XX:+PrintGCDetails -XX:+PrintGCTimeStamps -XX:+PrintHeapAtGC -Xloggc:/tmp/spark_gc.log, -Djava.io.tmpdir=$PWD/tmp, -Dlog4j.configuration=log4j-spark-container.properties, org.apache.spark.executor.CoarseGrainedExecutorBackend, akka.tcp://spark@sparktest1:41606/user/CoarseGrainedScheduler, 1, sparktest2, 3, 1>, <LOG_DIR>/stdout, 2>, <LOG_DIR>/stderr)
```

其中，akka.tcp://spark@sparktest1:41606/user/CoarseGrainedScheduler表示当前的Executor进程所在节点，后面的1表示Executor编号，sparktest2表示ApplicationMaster的host，接着的3表示当前Executor所占用的CPU数目。