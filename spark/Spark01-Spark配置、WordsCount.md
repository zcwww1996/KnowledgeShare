[TOC]
# 1. spark安装配置
1、**进入到Spark安装目录**

```bash
cd /usr/local/spark-2.1.0-bin-hadoop2.6
```

2、**进入conf目录并重命名并修改spark-env.sh.template文件**

```bash
cd conf/
mv spark-env.sh.template spark-env.sh
vi spark-env.sh
```

在该配置文件中添加如下配置

```bash
export JAVA_HOME= /usr/local/jdk1.8.0_141
export SPARK_MASTER_IP=linux01
export SPARK_MASTER_PORT=7077
```

保存退出

3、**重命名并修改slaves.template文件**

```bash
mv slaves.template slaves
vi slaves
```

在该文件中添加子节点所在的位置（Worker节点）

```bash
linux02
linux03
linux04
```

保存退出

4、**将配置好的Spark拷贝到其他节点上**

5、**在linux01上启动Spark集群**

```bash
/sbin/start-all.sh
```


6、**查看集群**

启动后执行jps命令，主节点上有Master进程，其他子节点上有Work进行，登录Spark管理界面查看集群状态
```bash
（主节点）：http://192.168.111.134:8080/
```

# 2. spark调优
## 2.1 配置spark高可用集群
在spark-env.sh中添加一个配置选项即可 （去掉以前配置的SPARK_MASTER_IP=MASTER）

在node1上执行sbin/start-all.sh脚本，然后在node2上执行sbin/start-master.sh启动第二个Master

```bash
export SPARK_DAEMON_JAVA_OPTS="-Dspark.deploy.recoveryMode=ZOOKEEPER -Dspark.deploy.zookeeper.url=zk-1:2181,zk-2:2181,zk-3:2181 -Dspark.deploy.zookeeper.dir=/spark"
```

## 2.2 配置spark的Work的计算资源
- 配置了Worker的可用内存（要给操作系统预留一下内存）
- 配置Worker的可用核数（线程数量）

配置方法：在Worker所在的节点的spark-env.sh上添加两个选项

```bash
export SPARK_WORKER_CORES=8
export SPARK_WORKER_MEMORY=6g
```

重启spark，重新加载最新的配置

## 2.3 启动了一个spark-shell
（Spark Shell是一个连接到Spark集群中用于提交计算任务的命令行客户端）

```bash
/bin/spark-shell --master spark://linux01:7077,linux02:7077 --executor-memory 5g --total-executor-cores 20
```

## 2.4 写了一个wordcount
先启动hdfs，然后上传要计算的数据

在spark shell中编写spark程序

```bash
sc.textFile("hdfs://linux01:9000/test.txt").flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_).sortBy(_._2, false).collect
```

# 3. Spark的standalone模式和Hadoop的YARN进行对比

|Spark|Yarn|职责|
|---|---|---|
|Master|ResourceManger|管理子节点，并进行资源调度，接收客户的提交的任务|
|Worker|NodeManager|管理当前节点，并向老大回报状态，启动真正负责执行计算任务的进程|
|Executor|YarnChild|真正的计算任务的进程，里面运行着真正执行数据计算的Task|
|SparkSubmit|AppMaster + Client|管理真正用于计算的进程和任务执行进度，并提交任务|

**提交wordCount程序的jar包到集群**

[![QJXgO0.png](https://s2.ax1x.com/2019/12/06/QJXgO0.png "提交jar到集群")](https://telegraph-image-5x2.pages.dev/file/4400abb1e71bb266d9be8.png)

```bash
/bin/spark-submit --master spark://linux01:7077,linux02:7077 --class cn.edu360.spark.day1.ScalaWordCount --total-executor-cores 2 /home/xiaoniu/original-HelloSpark2-1.0-SNAPSHOT.jar hdfs:/linux01:9000/words hdfs://linux01:9000/out
```

# 4. WordCount
## 4.1 scala

```scala
import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object ScalaWordCount {

  def main(args: Array[String]): Unit = {

    val conf = new SparkConf().setAppName("ScalaWordCount").setMaster("local[4]")
    //sparkContext，是Spark程序执行的入口
    val sc = new SparkContext(conf)

    //通过sc指定以后从哪里读取数据
    //弹性分布式数据集，一个神奇的大集合
    val inputdata: RDD[String] = sc.textFile(args(0))
    
     //去除标点符号
    val lines = inputdata.map(_.replaceAll("[\\pP+~$`^=|<>～｀＄＾＋＝｜＜＞￥×]", ""))

    //将内容切分压平
    val words: RDD[String] = lines.flatMap(_.split(" "))

    //将单词和一组合在一起
    val wordAndOne: RDD[(String, Int)] = words.map((_, 1))

    //分组聚合
    val reduced: RDD[(String, Int)] = wordAndOne.reduceByKey(_+_)

    //排序，false：倒序
    val sorted: RDD[(String, Int)] = reduced.sortBy(_._2, false)

    //保存结果
    sorted.saveAsTextFile(args(1))

    //取前10条数据
    sorted.take(10).foreach(println)
    
    //释放资源
    sc.stop()
  }

}
```

> 补充：spark 正则过滤
> ```Scala
> 
> # 去除所有非数字字符
> str.replaceAll("\\D", "")
> 
> # spark配合正则表达式过滤
> val pattern = """[a-zA-Z]*""".r
> val filtered = rdd.filter(line => pattern.pattern.matcher(line).matches)
> 
> val regexStr = "^[6-9]\\d{9}$"
> val bool1 = Pattern.matches(regexStr, "612345678a")
> val bool2 = Pattern.compile(regexStr).matcher("6123456789").matches()
> ```


## 4.2 java

```java
import org.apache.spark.SparkConf;
import org.apache.spark.SparkContext;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.function.FlatMapFunction;
import org.apache.spark.api.java.function.Function2;
import org.apache.spark.api.java.function.PairFunction;
import scala.Tuple2;

import java.util.Arrays;
import java.util.Iterator;

public class JavaWordCount {

    public static void main(String[] args) {

        SparkConf conf = new SparkConf().setAppName("JavaWordCount");

        //创建SparkContext，是Spark程序执行的入口
        JavaSparkContext jsc = new JavaSparkContext(conf);
        
        //指定以后从哪里读取数据
        JavaRDD<String> lines = jsc.textFile(args[0]);

        //切分压平
        JavaRDD<String> words = lines.flatMap(new FlatMapFunction<String, String>() {
            @Override
            public Iterator<String> call(String line) throws Exception {
                return Arrays.asList(line.split(" ")).iterator();
            }
        });

        //将单词和一组合在一起
        JavaPairRDD<String, Integer> wordAndOne = words.mapToPair(new PairFunction<String, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(String word) throws Exception {
                return new Tuple2<>(word, 1);
            }
        });

        //分组聚合
        JavaPairRDD<String, Integer> reduced = wordAndOne.reduceByKey(new Function2<Integer, Integer, Integer>() {
            @Override
            public Integer call(Integer v1, Integer v2) throws Exception {
                return v1 + v2;
            }
        });

        //排序

        //将key和value顺序颠倒
        JavaPairRDD<Integer, String> swaped = reduced.mapToPair(new PairFunction<Tuple2<String, Integer>, Integer, String>() {
            @Override
            public Tuple2<Integer, String> call(Tuple2<String, Integer> tp) throws Exception {
                //return new Tuple2<>(tp._2(), tp._1());
                return tp.swap();
            }
        });

        //排序
        JavaPairRDD<Integer, String> sorted = swaped.sortByKey(false);

        //颠倒顺序
        JavaPairRDD<String, Integer> reslut = sorted.mapToPair(new PairFunction<Tuple2<Integer, String>, String, Integer>() {
            @Override
            public Tuple2<String, Integer> call(Tuple2<Integer, String> tp) throws Exception {
                return tp.swap();
            }
        });

        //保存
        reslut.saveAsTextFile(args[1]);

        //释放资源
        jsc.stop();

    }
}
```

## 4.3 Lambada
```java
import org.apache.spark.SparkConf;
import org.apache.spark.api.java.JavaPairRDD;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.api.java.JavaSparkContext;
import scala.Tuple2;

import java.util.Arrays;

public class JavaLambdaWC {

    public static void main(String[] args) {

        SparkConf conf = new SparkConf().setAppName("JavaLambdaWC");

        JavaSparkContext jsc = new JavaSparkContext(conf);

        //以后从哪来里读取数据
        JavaRDD<String> lines = jsc.textFile(args[0]);

        //切分压平
        JavaRDD<String> words = lines.flatMap(line -> Arrays.asList(line.split(" ")).iterator());

        //将单词和一组合
        JavaPairRDD<String, Integer> wordAndOne = words.mapToPair(w -> new Tuple2<>(w, 1));

        //分组聚合
        JavaPairRDD<String, Integer> reduced = wordAndOne.reduceByKey((x, y) -> x + y);

        //颠倒顺序
        JavaPairRDD<Integer, String> swaped = reduced.mapToPair(tp -> tp.swap());

        //排序
        JavaPairRDD<Integer, String> sorted = swaped.sortByKey(false);

        //颠倒回来
        JavaPairRDD<String, Integer> result = sorted.mapToPair(tp -> tp.swap());

        //保存结果
        result.saveAsTextFile(args[1]);

        jsc.stop();
    }
}
```
