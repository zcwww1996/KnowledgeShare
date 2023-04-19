[TOC]
# 1. scala编写UDF

## 1.1 需要的依赖

```
 implementation group: 'org.scala-lang', name: 'scala-library', version: '2.11.12'
 implementation group: 'org.apache.hive', name: 'hive-exec', version: '1.1.0'
 // 非必要，输入类型不使用Text，可以不引入
 implementation group: 'org.apache.hadoop', name: 'hadoop-common', version: '2.6.0'
```

## 1.2 代码

创建Scala类并扩展Hive的UDF类

```Scala
package com.smartstep.UDF.scala

import org.apache.hadoop.hive.ql.exec.{Description, UDF}
import org.apache.hadoop.io.Text

import java.security.MessageDigest

@Description(
  name = "md5",
  value = "_FUNC_(str) - returns the com.smartstep.UDF.scala.Str_MD5 hash of a string",
  extended = "Example:\n" +
    "  > SELECT md5('hello world') FROM mytable;\n" +
    "  '5eb63bbbe01eeed093cb22bb8f5acdc3'"
)
class Str_MD5 extends UDF {
  def evaluate(input: String): String = {
    if (input == null) return null

    val md5 = MessageDigest.getInstance("MD5")
    val encoded = md5.digest(input.getBytes("UTF-8"))
    encoded.map("%02x".format(_)).mkString
  }
}
```

在上面的代码中，我们使用Java的`MessageDigest`类来计算输入字符串的MD5散列值，并将其转换为十六进制字符串返回。

**注意事项**：

- 输入数据类型可以是scala中的`String`类型，也可以是Hadoop中的`org.apache.hadoop.io.Text`类型，后者需要导入`hadoop-common`包
- `Description`注释属于非必须代码

## 1.3 注册函数

```sql
# 注册永久函数
CREATE FUNCTION str_md5_udf AS 'com.smartstep.UDF.scala.Str_MD5' USING JAR 'hdfs://data/zhangchao/DecodeLine.jar';

SELECT str_md5_udf('hello world');

--- 5eb63bbbe01eeed093cb22bb8f5acdc3
```

- 默认注册在default库，如果需要其他库(myudf)，使用`myudf.str_md5_udf`
- 如果需要**更新函数**，可以重新打包JAR文件并使用以下命令重新加载函数

```sql
DROP FUNCTION str_md5_udf;
CREATE FUNCTION str_md5_udf AS 'com.smartstep.UDF.scala.Str_MD5' USING JAR 'hdfs://data/zhangchao/DecodeLine.jar';
```

## 1.4 自定义函数开发中异常处理

参考：[编写scala版hive的自定义函数](https://blog.csdn.net/md_2014/article/details/128647266)

1. 继承hive的UDF类后，必须用scala的class定义子类，而非object类
2. 方法内用到split方法时，默认会对结果做隐式转换，这会导致udf运行时，
3. UDFArgumentTypeException第一个参数注意从0开始，否则无法正常抛异常
4. running query: `java.lang.BootstrapMethodError: java.lang.NoClassDefFoundError: scala/runtime/java8/JFunction1m c V I mcVImcVIsp (state=,code=0)`，循环遍历数组只能用while
5. `java.lang.NoSuchMethodError: scala.Predef`，这个错误可能是由于scala版本不兼容导致的。需要检查您的UDF类使用的scala版本和hive或spark运行时使用的scala版本是否一致
6. 如果不能更改版本，可以通过下面方法回避：
    1) 触发报错running query: `java.lang.NoSuchMethodError: scala.Predef`，解决方案是`import scala.Predef.{refArrayOps => _}`，禁止做隐式转换
    2) 使用Scala中的map集合会出现 `java.lang.NoSuchMethodError: scala.Predef`，使用Java的hashmap集合替代即可，`new util.HashMap[Int, String]()`

## 1.5 hive兼容性


|Hive version |Hadoop version| Spark version |Scala version|
|---|---|---|---|
|1.1.0|2.6|1.2.0|2.10|
|1.2.x|2.6|1.3.1|2.10|
|2.0.x|2.7|1.5.0|2.10|
|2.1.x|2.7|1.6.0|2.10|
|2.2.x|2.7|1.6.0|2.11|
|2.3.x|2.7|2.0.0|2.11|
|3.0.x|3.0|2.3.0|2.11|

