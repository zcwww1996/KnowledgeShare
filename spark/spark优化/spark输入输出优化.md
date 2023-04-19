[TOC]

# spark输入
## 问题一：输入路径小文件过多导致Task数目过多

这样的场景直接导致HDFS中存储着数目众多但单个文件数据量很小的情况，间接影响着Spark App Task的数目。

**原始写法：**

```Scala
 val conf = new SparkConf().setAppName("ScalaWordCount").setMaster("local[2]")
    //sparkContext，是Spark程序执行的入口
    val sc = new SparkContext(conf)
// 读取e:\\wordconut目录下的所有文件
     val lines: RDD[String] = sc.textFile("e:\\wordconut")
    
     println("分区数：" + lines.partitions.size)
```

这样一个block会形成一个partition，对应一个task。

**优化写法：**

```Scala
  val conf = new SparkConf().setAppName("ScalaWordCount").setMaster("local[2]")
    //sparkContext，是Spark程序执行的入口
    val sc = new SparkContext(conf)

  // 读取e:\\wordconut目录下的所有文件
    val lines: RDD[String] = sc.newAPIHadoopFile(
      "E:\\wordconut",
      classOf[org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat],
      classOf[org.apache.hadoop.io.LongWritable],
      classOf[org.apache.hadoop.io.Text]).map(s => s._2.toString)
```

`org.apache.hadoop.mapreduce.lib.input.CombineTextInputFormat`可以将多个小文件合并生成一个Split，而一个Split会被一个Task处理，从而减少Task的数目。

## 问题二：spark获取输入文件名
### 方法1：newAPIHadoopFile

```Scala
  def fileNameTest1(sc: SparkContext): Unit = {
    val input = "E:\\english"
    val fileRDD: RDD[(LongWritable, Text)] = sc.newAPIHadoopFile[LongWritable, Text, TextInputFormat](input)
    val hadoopRDD = fileRDD.asInstanceOf[NewHadoopRDD[LongWritable, Text]]
    val fileAdnLine: RDD[String] = hadoopRDD.mapPartitionsWithInputSplit((inputSplit: InputSplit, iterator: Iterator[(LongWritable, Text)]) => {
      val file = inputSplit.asInstanceOf[FileSplit]
      iterator.map(x => {
        //file.getPath.toString   文件的全路径
        //file.getPath.getName  文件名
        file.getPath.getName().split("_")(0) + " ---> " + x._2.toString()
      })
    })
    fileAdnLine.collect().toList.foreach(println)
  }
```

### 方法2：wholeTextFiles

```Scala
def fileNameTest2(sc: SparkContext): Unit = {
    val input = "E:\\english"
    val rdd: RDD[(String, String)] = sc.wholeTextFiles(input)

    val rdd_2: RDD[(String, String)] = rdd.mapPartitions(iter => {
      var res = List[(String, String)]()
      while (iter.hasNext) {
        val file = iter.next
        val fileName = file._1
        val lines = file._2.split("\n").toList

        for (line <- lines) {
          res = res.::((fileName, line))
        }
      }
      res.iterator

    })

    rdd_2.collect().toList.foreach(println)
  }
```

### 方法3：spark Streaming获取文件名
```Scala
  def getFileNameStream(ssc: StreamingContext, path: String): DStream[(String, String)] = {

// directory：指定待分析文件的目录；
// filter：用户指定的文件过滤器，用于过滤directory中的文件；
// newFilesOnly：应用程序启动时，目录directory中可能已经存在一些文件，如果newFilesOnly值为true，表示忽略这些文件；如果newFilesOnly值为false，表示需要分析这些文件；
// conf：用户指定的Hadoop相关的配置属性；
    val source: DStream[(String, String)] = ssc.fileStream[LongWritable, Text, TextInputFormat](path + "/_bakGZ", pathFilter, true).transform(rdd => {

      val tmp: UnionRDD[String] = new UnionRDD[String](rdd.context, rdd.dependencies.map(dep => {
        dep.rdd.asInstanceOf[RDD[(LongWritable, Text)]].map(_._2.toString).setName(dep.rdd.name)
      }))


      new UnionRDD(tmp.context, tmp.dependencies.flatMap(dep => {

        if (dep.rdd.isEmpty()) {
          None
        } else {
          val fileName = new Path(dep.rdd.name).getName
          Some(dep.rdd.asInstanceOf[RDD[String]].map(f => {
            (fileName, f)
          }).setName(fileName))
        }

      }))

    })


    source
  }
  
  
  def pathFilter(path: Path): Boolean = {
      !path.getName.endsWith(".gz")
    }
```

## 问题三：spark多个输入路径


```Scala
    val fileList: Seq[String] = Seq("D:\\input\\parquet\\test.parquet", "D:\\input\\parquet\\test2.parquet", "D:\\input\\parquet\\test3.parquet")
    // 读取hdfs文件目录
    val parquetFile: DataFrame = session.read.parquet(fileList: _*)
```


# spark输出
## 问题一：saveAsTextFile方法出现很多空的文件
saveAsTextFile方法会将数据按分区写入，有时候数据被过滤后很小(200条)，但是rdd分区为500，saveAsTextFile会产生500个文件，其中大部分为空文件

针对这种情况，可以调用repartition(num)，或者coalesce(num,true)，利用shuffle，将分区合并，shuffle会将数据均匀打散到各个分区，避免了空文件

```Scala
优化前
supplementRdd.saveAsTextFile(output_str)

优化后
supplementRdd.coalesce(50,true).saveAsTextFile(output_str)
```

## 问题二：saveAsHadoopFile函数中 FileOutputCommitter版本问题

参考：
- [遇到AM报内存溢出问题,查证原因是该配置mapreduce.fileoutputcommitter.algorithm.version=1引起](https://blog.csdn.net/Little_White_Growth/article/details/98946042)
- [mapreduce.fileoutputcommitter.algorithm.version版本区别](https://blog.csdn.net/daoxu_hjl/article/details/108208327)

### 两个版本优劣
**==mapreduce.fileoutputcommitter.algorithm.version = 1==**<br>
**性能方面**：v1在task结束后只是将输出文件拷到临时目录，然后在job结束后才由Driver把这些文件再拷到输出目录。如果文件数量很多，Driver就需要不断的和NameNode做交互，而且这个过程是单线程的，因此势必会增加耗时。如果我们碰到有spark任务所有task结束了但是任务还没结束，很可能就是Driver还在不断的拷文件；

**数据一致性方面**：v1在Job结束后才批量拷文件，其实就是两阶段提交，它可以保证数据要么全部展示给用户，要么都没展示（当然，在拷贝过程中也无法保证完全的数据一致性，但是这个时间一般来说不会太长）。如果任务失败，也可以直接删了_temporary目录，可以较好的保证数据一致性。

**==mapreduce.fileoutputcommitter.algorithm.version = 2==**<br>
**性能方面**：v2在task结束后立马将输出文件拷贝到输出目录，后面Job结束后Driver就不用再去拷贝了

**数据一致性方面**： v2在task结束后就拷文件，就会造成spark任务还未完成就让用户看到一部分输出，这样就完全没办法保证数据一致性了。另外，如果任务在输出过程中失败，就会有一部分数据成功输出，一部分没输出的情况。

总结<br>
在 **==性能方面，v2完胜v1==** <br>
在 **==数据一致性方面，v1完胜v2==**

### 设置方法
- 直接在 conf/spark-defaults.conf 里面设置 `spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version 2`，这个是全局影响的。
- 在spark2-submit提交中可以使用`--conf spark.hadoop.mapreduce.fileoutputcommitter.algorithm.version=2` 这中形式设置这个参数.
- 直接在 Spark 程序里面设置，`spark.conf.set("mapreduce.fileoutputcommitter.algorithm.version", "2")`，这个是作业级别的。
- 如果你是使用 Dataset API 写数据到 HDFS，那么你可以这么设置 `dataset.write.option("mapreduce.fileoutputcommitter.algorithm.version", "2")`。
- 不过如果你的 Hadoop 版本为 3.x，`mapreduce.fileoutputcommitter.algorithm.version` 参数的默认值已经设置为2了，具体参见 MAPREDUCE-6336 和 MAPREDUCE-6406。

备注：在 Hadoop 2.7.0 之前版本，我们可以将 ·mapreduce.fileoutputcommitter.algorithm.version· 参数设置为非1的值就可以实现这个目的，因为程序里面并没有限制这个值一定为2,。不过到了 Hadoop 2.7.0，·mapreduce.fileoutputcommitter.algorithm.version· 参数的值必须为1或2

### 原理解析
Spark 2.x 用到了 Hadoop 2.x，其将生成的文件保存到 HDFS 的时候，最后会调用了 saveAsHadoopFile，而这个函数在里面用到了 FileOutputCommitter，如下：

```scala
def saveAsHadoopFile(

      path: String,

      keyClass: Class[_],

      valueClass: Class[_],

      outputFormatClass: Class[_ <: OutputFormat[_, _]],

      conf: JobConf = new JobConf(self.context.hadoopConfiguration),

      codec: Option[Class[_ <: CompressionCodec]] = None): Unit = self.withScope {

    ........

     // Use configured output committer if already set

    if (conf.getOutputCommitter == null) {

      hadoopConf.setOutputCommitter(classOf[FileOutputCommitter])

    }

    ........

}
```

Hadoop 2.x 的 FileOutputCommitter 实现，FileOutputCommitter 里面有两个值得注意的方法：commitTask 和 commitJob。在 Hadoop 2.x 的FileOutputCommitter 实现里面，**`mapreduce.fileoutputcommitter.algorithm.version` 参数控制着 commitTask 和 commitJob的工作方式**。

```java
public void commitTask(TaskAttemptContext context, Path taskAttemptPath) 
    throws IOException {
 
     ........
 
    if (taskAttemptDirStatus != null) {
      if (algorithmVersion == 1) {
        Path committedTaskPath = getCommittedTaskPath(context);
        if (fs.exists(committedTaskPath)) {
           if (!fs.delete(committedTaskPath, true)) {
             throw new IOException("Could not delete " + committedTaskPath);
           }
        }
        if (!fs.rename(taskAttemptPath, committedTaskPath)) {
          throw new IOException("Could not rename " + taskAttemptPath + " to "
              + committedTaskPath);
        }
        LOG.info("Saved output of task '" + attemptId + "' to " +
            committedTaskPath);
      } else {
        // directly merge everything from taskAttemptPath to output directory
        mergePaths(fs, taskAttemptDirStatus, outputPath);
        LOG.info("Saved output of task '" + attemptId + "' to " +
            outputPath);
      }
    } else {
      LOG.warn("No Output found for " + attemptId);
    }
  } else {
    LOG.warn("Output Path is null in commitTask()");
  }
}
 
public void commitJob(JobContext context) throws IOException {
      ........
      jobCommitNotFinished = false;
     ........
}
 
protected void commitJobInternal(JobContext context) throws IOException {
    ........
    if (algorithmVersion == 1) {
      for (FileStatus stat: getAllCommittedTaskPaths(context)) {
        mergePaths(fs, stat, finalOutput);
      }
    }
   ........
}

```

设置成v1时，task完成commitTask时会将临时生成的数据移动到task对应目录下，当所有task都执行完成进行commitJob的时候，会由driver按照单线程模式将所有task目录下的数据文件移动到最终的作业输出目录

> v1：commitTask的操作是将 `${output.dir}/_temporary/${appAttemptId}/_temporary/${taskAttemptId}` 重命名为 `${output.dir}/_temporary/${appAttemptId}/${taskId}`

设置成v2时，task完成就会直接将临时生成的数据移动到Job最终输出目录，commitJob的时候就不需要再次移动数据

> v2: commitTask的操作是将 `${output.dir}/_temporary/${appAttemptId}/_temporary/${taskAttemptId}` 下的文件移动到 `${output.dir}` 目录下 （也就是最终的输出目录）


