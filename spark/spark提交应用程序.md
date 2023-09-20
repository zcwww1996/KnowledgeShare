[TOC]

spark任务提交有三种方式

1：通过local方式提交

2：通过spark-submit脚本提交到集群

3：通过spark提交的API SparkLauncher提交到集群，这种方式可以将提交过程集成到我们的spring工程中，更加灵活

# 1. spark-submit提交

## 1.1 spark-submit提交程序包含第三方jar文件

### 1.1.1 第一种方式：打包到jar应用程序

**操作：** 将第三方jar文件打包到最终形成的spark应用程序jar文件中

**应用场景：** 第三方jar文件比较小，应用的地方比较少

### 1.1.2 第二种方式：spark-submit 参数 --jars

**操作：** 使用spark-submit提交命令的参数: --jars

要求：

1、使用spark-submit命令的机器上存在对应的jar文件

2、至于集群中其他机器上的服务需要该jar文件的时候，通过driver提供的一个http接口来获取该jar文件的(例如：`http://192.168.187.146:50206/jars/mysql-connector-java-5.1.27-bin.jar` Added By User)

```bash
# 配置参数：--jars JARS
如下示例：
$ bin/spark-shell --jars /opt/cdh-5.3.6/hive/lib/mysql-connector-java-5.1.27-bin.jar
```

**应用场景：** 要求本地必须要有对应的jar文件

### 1.1.3 第三种方式：spark-submit 参数 --packages

**操作：** 使用spark-submit提交命令的参数: --packages

```bash
# 配置参数：--packages  jar包的maven地址
如下示例：
$ bin/spark-shell --packages  mysql:mysql-connector-java:5.1.27 --repositories http://maven.aliyun.com/nexus/content/groups/public/
```

- `--repositories` 为mysql-connector-java包的maven地址，若不给定，则会使用该机器安装的maven默认源中下载
- 若依赖多个包，则重复上述jar包写法，中间以逗号分隔
- 默认下载的包位于当前用户根目录下的.ivy/jars文件夹中
- 如果没有连接外网，可以把jar包上传到当前用户根目录下的`.ivy/jars`或`.m2/jars`目录

**应用场景：** 本地可以没有，集群中服务需要该包的的时候，都是从给定的maven地址，直接下载

### 1.1.4 第四种方式：添加到spark的环境变量

**操作：** 更改Spark的配置信息:SPARK\_CLASSPATH, 将第三方的jar文件添加到SPARK\_CLASSPATH环境变量中

**注意事项**：要求Spark应用运行的所有机器上必须存在被添加的第三方jar文件

```bash
# 1.创建一个保存第三方jar文件的文件夹:
$ mkdir external_jars

# 2.修改Spark配置信息
$ vim conf/spark-env.sh
修改内容：SPARK_CLASSPATH=$SPARK_CLASSPATH:/opt/cdh-5.3.6/spark/external_jars/*

3.将依赖的jar文件copy到新建的文件夹中
$ cp /opt/cdh-5.3.6/hive/lib/mysql-connector-java-5.1.27-bin.jar ./external_jars/
```

**应用场景：** 依赖的jar包特别多，写命令方式比较繁琐，被依赖包应用的场景也多的情况下

备注：（只针对spark on yarn(cluster)模式）

### 1.1.5 spark on yarn(cluster)，依赖第三方jar文件

**最终解决方案：**<br> 将第三方的jar文件copy到`${HADOOP_HOME}/share/hadoop/common/lib`文件夹中(Hadoop集群中所有机器均要求copy)

## 1.2 spark-submit提交程序携带外部配置文件

spark开发时，有些时候需要动态加载jar外部资源⽂件，需要在driver访问。就需要通过`--files`把外部资源⽂件加载到classpath路径，如下所示


```scala
/opt/ubd/core/spark-2.4.3/bin/spark-submit  \
        --class cn.smartstep.extract.tables.ng.UserAppNG \
        --master yarn               \
        --deploy-mode cluster \
        --queue ss_deploy                     \
        --driver-memory 20g                  \
        --num-executors 100                  \
        --executor-cores 4          \
        --executor-memory 20g                \
        --conf spark.default.parallelism=800  \
        --conf spark.sql.shuffle.partitions=800 \
        --files /home/zhangchao/testData/daas/cityList/daasCityList_user_app.conf \
        ../daasNG-1.1-SNAPSHOT.jar
```
### 1.2.1 读取hdfs临时目录
`--files`会把本地文件`/home/zhangchao/testData/daas/cityList/daasCityList_user_app.conf`上传到hdfs临时目录` hdfs://10.244.12.215:8020/user/zhangchao/.sparkStaging/application_1578647131225_1066324/daasCityList_user_app.conf`，要想读取"daasCityList_user_app.conf"文件，关键在于获取hdfs对应路径

```scala
// client模式
    val applicationId = sc.getConf.getAppId
    val tmpDir = s".sparkStaging/$applicationId/daasCityList_user_app.conf"
// cluster模式
   System.getenv("SPARK_YARN_STAGING_DIR").toString + "/daasCityList_user_app.conf"
```

**综述，client、cluster模式通用获取`--files`文件的方式**

方法1：
```scala
    val applicationId = sc.getConf.getAppId
    val tmpDir = s".sparkStaging/$applicationId/daasCityList_user_app.conf"
    val lines = sc.textFile(tmpDir).collect()
```

方法2：
```scala
import scala.util.Try

    val applicationId = sc.getConf.getAppId
    val tmpDir = s".sparkStaging/$applicationId"
    val tmpPath = Try(System.getenv("SPARK_YARN_STAGING_DIR").toString).getOrElse(tmpDir)+"/daasCityList_user_app.conf"
    val lines = sc.textFile(tmpPath).collect()
```


### 1.2.2 读取ApplicationMaster本地路径
在cluster模式下直接读取ApplicationMaster的工作路径（本地路径），`--file`里面的文件会复制到该路径，需要如下代码：

```Scala
// 读取ApplicationMaster的工作路径（本地路径）
val filePath = System.getProperty("user.dir")
val deployMode = System.getProperty("spark.submit.deployMode")

// 判断是cluster还是client
if("cluster"==deployMode){
      val source = Source.fromFile(filePath + "/" + "daasCityList_user_app.conf")
      val lineIterator = source.getLines()
      lineIterator.foreach(println)
    }
```

> 废弃，代码不可用
> ```scala
> val is: InputStream = this.getClass.getResourceAsStream(“./xxx.sql”)
> val bufferSource = Source.fromInputStream(is)
> ```

**client下ApplicationMaster的工作路径（本地路径）为submit提交路径，和`--file` 无关**

可以将`--files`的文件放在submit提交路径，与jar同一级目录，达到`"user.dir"`在cluster \ client下一致

**综述，client、cluster模式通用获取`--files`文件的方式**

submit命令
```scala
/opt/ubd/core/spark-2.4.3/bin/spark-submit  \
        --class cn.smartstep.extract.tables.ng.UserAppNG \
        --master yarn               \
        --deploy-mode cluster \
        --queue ss_deploy                     \
        --driver-memory 20g                  \
        --num-executors 100                  \
        --executor-cores 4          \
        --executor-memory 20g                \
        --conf spark.default.parallelism=800  \
        --conf spark.sql.shuffle.partitions=800 \
        --files daasCityList_user_app.conf \
        daasNG-1.1-SNAPSHOT.jar
```


读取文件代码
```Scala
// 取ApplicationMaster的工作路径（本地路径）
val filePath = System.getProperty("user.dir")

// cluster还是client，全在submit提交目录
      val source = Source.fromFile(filePath + "/" + "daasCityList_user_app.conf")
      val lineIterator = source.getLines()
      lineIterator.foreach(println)
```



> **==【拓展】==**<br>
> spark环境变量`System.getProperties`遍历，方法详见【Scala01-方法和函数、shell、wordscount】3.2 章节
> ```Scala
> (k,v)：java.vendor===Oracle Corporation
> (k,v)：sun.java.launcher===SUN_STANDARD
> (k,v)：sun.nio.ch.bugLevel===
> (k,v)：sun.management.compiler===HotSpot 64-Bit Tiered Compilers
> (k,v)：os.name===Linux
> (k,v)：sun.boot.class.path===/usr/java/jdk1.8.0_161/jre/lib/resources.jar:/usr/java/jdk1.8.0_161/jre/lib/rt.jar:/usr/java/jdk1.8.0_161/jre/lib/sunrsasign.jar:/usr/java/jdk1.8.0_161/jre/lib/jsse.jar:/usr/java/jdk1.8.0_161/jre/lib/jce.jar:/usr/java/jdk1.8.0_161/jre/lib/charsets.jar:/usr/java/jdk1.8.0_161/jre/lib/jfr.jar:/usr/java/jdk1.8.0_161/jre/classes
> (k,v)：spark.submit.deployMode===cluster
> (k,v)：java.vm.specification.vendor===Oracle Corporation
> (k,v)：java.runtime.version===1.8.0_161-b12
> (k,v)：spark.org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter.param.PROXY_HOSTS===lf319-rh2288s-017,lf319-rh2288s-018
> (k,v)：spark.yarn.queue===ss_deploy
> (k,v)：user.name===yarn
> (k,v)：spark.executor.cores===4
> (k,v)：user.language===zh
> (k,v)：sun.boot.library.path===/usr/java/jdk1.8.0_161/jre/lib/amd64
> (k,v)：spark.network.timeout===1200
> (k,v)：java.version===1.8.0_161
> (k,v)：user.timezone===Asia/Shanghai
> (k,v)：spark.master===yarn
> (k,v)：sun.arch.data.model===64
> (k,v)：spark.org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter.param.RM_HA_URLS===lf319-rh2288s-017:8088,lf319-rh2288s-018:8088
> (k,v)：java.endorsed.dirs===/usr/java/jdk1.8.0_161/jre/lib/endorsed
> (k,v)：sun.cpu.isalist===
> (k,v)：sun.jnu.encoding===UTF-8
> (k,v)：file.encoding.pkg===sun.io
> (k,v)：file.separator===/
> (k,v)：java.specification.name===Java Platform API Specification
> (k,v)：spark.org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter.param.PROXY_URI_BASES===http://lf319-rh2288s-017:8088/proxy/application_1578647131225_1098755,http://lf319-rh2288s-018:8088/proxy/application_1578647131225_1098755
> (k,v)：java.class.version===52.0
> (k,v)：jetty.git.hash===unknown
> (k,v)：user.country===CN
> (k,v)：java.home===/usr/java/jdk1.8.0_161/jre
> (k,v)：java.vm.info===mixed mode
> (k,v)：os.version===3.10.0-693.el7.x86_64
> (k,v)：path.separator===:
> (k,v)：java.vm.version===25.161-b12
> (k,v)：spark.eventLog.compress===true
> (k,v)：java.awt.printerjob===sun.print.PSPrinterJob
> (k,v)：sun.io.unicode.encoding===UnicodeLittle
> (k,v)：awt.toolkit===sun.awt.X11.XToolkit
> (k,v)：spark.default.parallelism===800
> (k,v)：spark.yarn.historyServer.address===http://lf319-rh2288s-019:18089
> (k,v)：user.home===/var/lib/hadoop-yarn
> (k,v)：java.specification.vendor===Oracle Corporation
> (k,v)：spark.yarn.app.container.log.dir===/var/log/hadoop-yarn/container/application_1578647131225_1098755/container_e94_1578647131225_1098755_01_000001
> (k,v)：java.library.path===:/opt/cloudera/parcels/CDH-5.13.1-1.cdh5.13.1.p0.2/lib/hadoop/lib/native:/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
> (k,v)：java.vendor.url===http://java.oracle.com/
> (k,v)：spark.yarn.app.id===application_1578647131225_1098755
> (k,v)：java.vm.vendor===Oracle Corporation
> (k,v)：spark.executor.memory===20g
> (k,v)：java.runtime.name===Java(TM) SE Runtime Environment
> (k,v)：sun.java.command===org.apache.spark.deploy.yarn.ApplicationMaster --class cn.smartstep.extract.tables.ng.configTest --jar file:/home/zhangchao/testData/daas/once/../daasNG-1.1-SNAPSHOT.jar --arg 5 --properties-file /mnt/sd10/yarn/nm/usercache/zhangchao/appcache/application_1578647131225_1098755/container_e94_1578647131225_1098755_01_000001/__spark_conf__/__spark_conf__.properties
> (k,v)：java.class.path===/mnt/sd10/yarn/nm/usercache/zhangchao/appcache/application_1578647131225_1098755/container_e94_1578647131225_1098755_01_000001:……[太长省略]……/__spark_conf__/__hadoop_conf__
> (k,v)：spark.shuffle.manager===sort
> (k,v)：java.vm.specification.name===Java Virtual Machine Specification
> (k,v)：java.vm.specification.version===1.8
> (k,v)：spark.ui.port===0
> (k,v)：sun.cpu.endian===little
> (k,v)：sun.os.patch.level===unknown
> (k,v)：java.io.tmpdir===/mnt/sd10/yarn/nm/usercache/zhangchao/appcache/application_1578647131225_1098755/container_e94_1578647131225_1098755_01_000001/tmp
> (k,v)：java.vendor.url.bug===http://bugreport.sun.com/bugreport/
> (k,v)：spark.yarn.dist.files===file:///home/zhangchao/testData/daas/cityList/daasCityList_user_app.conf
> (k,v)：os.arch===amd64
> (k,v)：java.awt.graphicsenv===sun.awt.X11GraphicsEnvironment
> (k,v)：java.ext.dirs===/usr/java/jdk1.8.0_161/jre/lib/ext:/usr/java/packages/lib/ext
> (k,v)：user.dir===/mnt/sd10/yarn/nm/usercache/zhangchao/appcache/application_1578647131225_1098755/container_e94_1578647131225_1098755_01_000001
> (k,v)：spark.port.maxRetries===100
> (k,v)：spark.driver.memory===20g
> (k,v)：spark.eventLog.dir===hdfs://nameservice1/user/spark/spark2ApplicationHistory
> (k,v)：line.separator===
> 
> (k,v)：java.vm.name===Java HotSpot(TM) 64-Bit Server VM
> (k,v)：spark.eventLog.enabled===true
> (k,v)：file.encoding===UTF-8
> (k,v)：spark.ui.filters===org.apache.hadoop.yarn.server.webproxy.amfilter.AmIpFilter
> (k,v)：spark.app.name===cn.smartstep.extract.tables.ng.configTest
> (k,v)：java.specification.version===1.8
> (k,v)：spark.executor.instances===100
> ```



## 1.3 spark作业配置与submit参数

### 1.3.1 spark作业配置的三种方式
1. 读取指定配置文件，默认为conf/spark-defaults.conf。
2. 在程序中的SparkConf中指定，如conf.setAppName(“myspark”)。
3. spark-submit中使用参数。

这三种方式的优先级为 SparkConf > spark-submit > 配置文件。可以在spark-submit中使用–verbos参数查看起作用的配置来自上述哪种方式。

### 1.3.2 spark-submit参数说明
使用spark-submit提交spark作业的时候有许多参数可供我们选择，这些参数有的用于作业优化，有的用于实现某些功能，所有的参数列举如下：



| 参数 | 说明 |
| --- | --- |
| --master | 集群的master地址。如：spark://host:port，mesos://host:port，yarn-client，yarn-cluster。<br>local\[k\]本地以k个worker线程执行，k一般为cpu的内核数，local\[\*\]以尽可能多的线程数执行。 |
| --deploy-mode | driver运行的模式，client或者cluster模式，默认为client |
| --class | 应用程序的主类（用于Java或者Scala应用） |
| --name | 应用程序的名称 |
| --jars | 作业执行过程中使用到的其他jar，可以使用逗号分隔添加多个。可以使用如下方式添加:<br>file：指定http文件服务器的地址，每个executor都从这个地址下载。<br>  hdfs,http,https,ftp:从以上协议指定的路径下载。<br>local:直接从当前的worker节点下载。 |
| --packages | 从maven添加作业执行过程中使用到的包，查找顺序先本地仓库再远程仓库。<br>可以添加多个，每个的格式为：groupId:artifactId:version |
| --exclude-packages | 需要排除的包，可以为多个，使用逗号分隔。 |
| --repositories | 远程仓库。可以添加多个，逗号分隔。 |
| --py-files | 逗号分隔的”.zip”,”.egg”或者“.py”文件，这些文件放在python app的PYTHONPATH下面 |
| --files | 逗号分隔的文件列表，这些文件放在每个executor的工作目录下。 |
| --conf | 其他额外的spark配置属性。 |
| --properties-file | 指向一个配置文件，通过这个文件可以加载额外的配置。<br>如果没有则会查找conf/spark-defaults.conf |
| --driver-memory | driver节点的内存大小。如2G，默认为1024M。 |
| --driver-java-options | 作用于driver的额外java配置项。 |
| --driver-library-path | 作用于driver的外部lib包。 |
| --driver-class-path | 作用于driver的额外类路径，使用--jar时会自动添加路径。 |
| --executor-memory | 每个excutor的执行内存。 |
| --proxy-user | 提交作业的模拟用户。是hadoop中的一种安全机制，具体可以参考:<br>http://dongxicheng.org/mapreduce-nextgen/hadoop-secure-impersonation/ |
| --verbose | 打印debug信息。 |
| --version | 打印当前spark的版本。 |
| --driver-cores | driver的内核数，默认为1。**（仅用于spark standalone集群中）** |
| --superivse | driver失败时重启 **（仅用于spark standalone或者mesos集群中）** |
| --kill | kill指定的driver **（仅用于spark standalone或者mesos集群中）** |
| --total-executor-cores | 给所有executor的所有内核数。**（仅用于spark standalone或者mesos集群中）** |
| --executor-cores | 分配给每个executor的内核数。**（仅用于spark standalone或者yarn集群中）** |
| --driver-cores | driver的内核数。**（仅yarn）** |
| --queue | 作业执行的队列。**（仅yarn）** |
| --num-executors | executor的数量。**（仅yarn）** |
| --archives | 需要添加到executor执行目录下的归档文件列表，逗号分隔。**（仅yarn）** |
| --principal | 运行于secure hdfs时用于登录到KDC的principal。**（仅yarn）** |
| --keytab | 包含keytab文件的全路径。**（仅yarn）** |

# 2. SparkLauncher提交

SparkLauncher也提供了两种方式提交任务

## 2.1 launch方式

SparkLauncher实际上是根据JDK自带的ProcessBuilder构造了一个UNIXProcess子进程提交任务，提交的形式跟spark-submit一样。这个子进程会以阻塞的方式等待程序的运行结果。简单来看就是拼接spark-submit命令，并以子进程的方式启动。

`process.getInputStream()实际上对应linux进程的标准输出stdout`<br> `process.getErrorStream()实际上对应linux进程的错误信息stderr`<br> `process.getOutputStream()实际上对应linux进程的输入信息stdin`<br>

```java

    public void sparkRun() {

        try {
            HashMap env = new HashMap();
            
            env.put("HADOOP_CONF_DIR", CommonConfig.HADOOP_CONF_DIR);
            env.put("JAVA_HOME", CommonConfig.JAVA_HOME);

            SparkLauncher handle = new SparkLauncher(env)
                    .setSparkHome(SparkConfig.SPARK_HOME)
                    .setAppResource(CommonConfig.ALARM_JAR_PATH)
                    .setMainClass(CommonConfig.ALARM_JAR_MAIN_CLASS)
                    .setMaster("spark://" + SparkConfig.SPARK_MASTER_HOST + ":" + SparkConfig.SPARK_MASTER_PORT)
                    .setDeployMode(SparkConfig.SPARK_DEPLOY_MODE)
                    .setVerbose(SparkConfig.SPARK_VERBOSE)
                    .setConf("spark.app.id", CommonConfig.ALARM_APP_ID)
                    .setConf("spark.driver.memory", SparkConfig.SPARK_DRIVER_MEMORY)
                    .setConf("spark.rpc.message.maxSize", SparkConfig.SPARK_RPC_MESSAGE_MAXSIZE)
                    .setConf("spark.executor.memory", SparkConfig.SPARK_EXECUTOR_MEMORY)
                    .setConf("spark.executor.instances", SparkConfig.SPARK_EXECUTOR_INSTANCES)
                    .setConf("spark.executor.cores", SparkConfig.SPARK_EXECUTOR_CORES)
                    .setConf("spark.default.parallelism", SparkConfig.SPARK_DEFAULT_PARALLELISM)
                    .setConf("spark.driver.allowMultipleContexts", SparkConfig.SPARK_DRIVER_ALLOWMULTIPLECONTEXTS)
                    .setVerbose(true);

            Process process = handle.launch();
            InputStreamRunnable inputStream = new InputStreamRunnable(process.getInputStream(), "alarm task input");
            ExecutorUtils.getExecutorService().submit(inputStream);
            InputStreamRunnable errorStream = new InputStreamRunnable(process.getErrorStream(), "alarm task error");
            ExecutorUtils.getExecutorService().submit(errorStream);

            logger.info("Waiting for finish...");
            int exitCode = process.waitFor();
            logger.info("Finished! Exit code:" + exitCode);
        } catch (Exception e) {
            logger.error("submit spark task error", e);
        }
    }

```

### 2.1.1 运行过程示意图

[![SparkLauncher launch启动示意图](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda8361ddd4796~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://pic.imgdb.cn/item/64dc8190661c6c8e54e2c0f3.webp)

1. 用户程序启动（SparkLauncher，非驱动程序）时会在当前节点上启动一个SparkSubmit进程，并将驱动程序（即spark任务）发送到任意一个工作节点上，在工作节点上启动DriverWrapper进程
2. 驱动程序会从集群管理器（standalone模式下是master服务器）申请执行器资源
3. 集群管理器反馈执行器资源给驱动器
4. 驱动器Driver将任务发送到执行器节点执行

### 2.1.2 spark首页监控

可以看到启动的Driver

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda8361487508d~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://pic.imgdb.cn/item/64dc8190661c6c8e54e2c14f.webp)

进一步可以查看到执行器情况

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda8361496749e~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://pic.imgdb.cn/item/64dc8190661c6c8e54e2c18c.webp)

### 2.1.3 通过服务器进程查看各进程之间的关系

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda8361e1757e9~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://pic.imgdb.cn/item/64dc8190661c6c8e54e2c1c9.webp)

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda8361dd82601~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://pic.imgdb.cn/item/64dc8190661c6c8e54e2c214.webp)

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda83653126f05~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://s1.ax1x.com/2023/08/16/pPlAVRs.jpg)

[![](https://p1-jj.byteimg.com/tos-cn-i-t2oaga2asx/gold-user-assets/2019/7/10/16bda83652f7d6ce~tplv-t2oaga2asx-jj-mark:3024:0:0:0:q75.awebp)](https://s1.ax1x.com/2023/08/16/pPlAEGj.jpg)

## 2.2 startApplication方式

```java
    public void sparkRun() {

        try {
            HashMap env = new HashMap();
            
            env.put("HADOOP_CONF_DIR", CommonConfig.HADOOP_CONF_DIR);
            env.put("JAVA_HOME", CommonConfig.JAVA_HOME);

            CountDownLatch countDownLatch = new CountDownLatch(1);
            SparkAppHandle handle = new SparkLauncher(env)
                    .setSparkHome(SparkConfig.SPARK_HOME)
                    .setAppResource(CommonConfig.ALARM_JAR_PATH)
                    .setMainClass(CommonConfig.ALARM_JAR_MAIN_CLASS)
                    .setMaster("spark://" + SparkConfig.SPARK_MASTER_HOST + ":" + SparkConfig.SPARK_MASTER_PORT)

                    .setDeployMode(SparkConfig.SPARK_DEPLOY_MODE)
                    .setVerbose(SparkConfig.SPARK_VERBOSE)
                    .setConf("spark.app.id", CommonConfig.ALARM_APP_ID)
                    .setConf("spark.driver.memory", SparkConfig.SPARK_DRIVER_MEMORY)
                    .setConf("spark.rpc.message.maxSize", SparkConfig.SPARK_RPC_MESSAGE_MAXSIZE)
                    .setConf("spark.executor.memory", SparkConfig.SPARK_EXECUTOR_MEMORY)
                    .setConf("spark.executor.instances", SparkConfig.SPARK_EXECUTOR_INSTANCES)
                    .setConf("spark.executor.cores", SparkConfig.SPARK_EXECUTOR_CORES)
                    .setConf("spark.default.parallelism", SparkConfig.SPARK_DEFAULT_PARALLELISM)
                    .setConf("spark.driver.allowMultipleContexts", SparkConfig.SPARK_DRIVER_ALLOWMULTIPLECONTEXTS)
                    .setVerbose(true).startApplication(new SparkAppHandle.Listener() {
                        
                        @Override
                        public void stateChanged(SparkAppHandle sparkAppHandle) {
                            if (sparkAppHandle.getState().isFinal()) {
                                countDownLatch.countDown();
                            }
                            System.out.println("state:" + sparkAppHandle.getState().toString());
                        }

                        @Override
                        public void infoChanged(SparkAppHandle sparkAppHandle) {
                            System.out.println("Info:" + sparkAppHandle.getState().toString());
                        }
                    });
            logger.info("The task is executing, please wait ....");
            
            countDownLatch.await();
            logger.info("The task is finished!");
        } catch (Exception e) {
            logger.error("submit spark task error", e);
        }
    }

```

这种模式下，yarn集群和local实测可以提交成功，在standalone模式下据说提交失败，未测试