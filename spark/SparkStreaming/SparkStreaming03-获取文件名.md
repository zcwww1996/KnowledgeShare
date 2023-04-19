spark Streaming获取输入文件名

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