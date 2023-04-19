[TOC]

gradle升级3.0之后，有了新的依赖方式，下面我来介绍一下他们的使用

先看看之前2.0的<br>
[![2.0.png](https://z3.ax1x.com/2021/06/02/2M3ggI.png)](https://imgtu.com/i/2M3ggI)

再看看现在3.0的<br>
[![3.0.png](https://z3.ax1x.com/2021/06/02/2M36Cd.png)](https://imgtu.com/i/2M36Cd)

在gradle3.0中，**compile依赖关系已被弃用，被implementation和api替代，provided被compile only替代，apk被runtime only替代**，剩下的看名字就知道了。

# 1. implementation和api的区别

- **api**：跟2.x版本的 compile完全相同
 
- **implementation**：只能在内部使用此模块，比如我在一个libiary中使用implementation依赖了gson库，然后我的主项目依赖了libiary，那么，我的主项目就无法访问gson库中的方法。这样的好处是编译速度会加快，推荐使用implementation的方式去依赖，如果你需要提供给外部访问，那么就使用api依赖即可

# 2. gradle 新旧版本对比
2.x版本依赖的可以看看下面的说明，括号里对应的是3.0版本的依赖方式

## 2.1 compile（implementation，api）
这种是我们最常用的方式，使用该方式依赖的库将会参与编译和打包。

implementation：该依赖方式所依赖的库不会传递，只会在当前module中生效。
api：该依赖方式会传递所依赖的库，当其他module依赖了该module时，可以使用该module下使用api依赖的库。

当我们依赖一些第三方的库时，可能会遇到com.android.support冲突的问题，就是因为开发者使用的compile或api依赖的com.android.support包与我们本地所依赖的com.android.support包版本不一样，所以就会报All com.android.support libraries must use the exact same version specification (mixing versions can lead to runtime crashes这个错误。

解决办法可以看这篇博客：[com.android.support冲突的解决办法](http://blog.csdn.net/yuzhiqiang_1993/article/details/78214812)

## 2.2 provided（compileOnly）
只在编译时有效，不会参与打包
可以在自己的moudle中使用该方式依赖一些比如com.android.support，gson这些使用者常用的库，避免冲突。

## 2.3 apk（runtimeOnly）
只在生成apk的时候参与打包，编译时不会参与，很少用。

## 2.4 testCompile（testImplementation）
testCompile 只在单元测试代码的编译以及最终打包测试apk时有效。

## 2.5 debugCompile（debugImplementation）
debugCompile 只在debug模式的编译和最终的debug apk打包时有效

## 2.6 releaseCompile（releaseImplementation）
Release compile 仅仅针对Release 模式的编译和最终的Release apk打包。

# 3. 样例

main中Spark、Scala 不进行打包，test中Spark、Scala正常测试

```groovy
dependencies {
    // spark scala 不进行打包
    compileOnly 'org.scala-lang:scala-library:2.12.15'
    testCompileOnly 'org.scala-lang:scala-library:2.12.15'
    compileOnly group: 'org.apache.spark', name: 'spark-core_2.12', version: '2.4.2'
    testCompileOnly group: 'org.apache.spark', name: 'spark-core_2.12', version: '2.4.2'
    compileOnly group: 'org.apache.spark', name: 'spark-sql_2.12', version: '2.4.2'
    testCompileOnly group: 'org.apache.spark', name: 'spark-core_2.12', version: '2.4.2'
    compileOnly group: 'org.apache.spark', name: 'spark-mllib_2.12', version: '2.4.2'
    testCompileOnly group: 'org.apache.spark', name: 'spark-mllib_2.12', version: '2.4.2'


    implementation group: 'org.yaml', name: 'snakeyaml', version: '1.28'
    implementation group: 'ch.hsr', name: 'geohash', version: '1.3.0'
    implementation group: 'com.typesafe.scala-logging', name: 'scala-logging_2.12', version: '3.9.3'
    implementation group: 'ch.qos.logback', name: 'logback-classic', version: '1.2.3'

    testImplementation group: 'org.scalatest', name: 'scalatest_2.12', version: '3.3.0-SNAP3'
    testImplementation 'org.junit.jupiter:junit-jupiter-api:5.7.0'
    testRuntimeOnly 'org.junit.jupiter:junit-jupiter-engine:5.7.0'
}
```

