build.gradle 如下：

```gradle
group 'cn.com.adtec'
version '1.0-SNAPSHOT'

buildscript {
    repositories {
        maven { url 'https://maven.aliyun.com/repository/gradle-plugin/' }
        maven { url 'https://plugins.gradle.org/m2/' }
    }
    dependencies {
        classpath 'com.github.jengelman.gradle.plugins:shadow:4.0.1'
    }
}

apply plugin: 'com.github.johnrengelman.shadow'
apply plugin: 'idea'
apply plugin: 'java'
apply plugin: 'scala'


sourceCompatibility = 1.8
targetCompatibility = 1.8
//设置编译字符集
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8"
}

//1、compileScala.options.encoding = 'UTF-8'
// 或2、tasks.withType(ScalaCompile)
tasks.withType(ScalaCompile) {
    scalaCompileOptions.encoding = "UTF-8"
}
// 构建可执行jar包，运行依赖jar内容会直接打到jar里面
shadowJar {
    // 包的文件数量大于65535 或者文件大小大于4G，需要加zip64 true
    zip64 true
    
    // 生成文件名附带时间戳与版本号，例如： journeyPlus-1.0-SNAPSHOT-assemble-202112071734.jar
    
    // jar 包名称
    archiveBaseName.set('journeyPlus')
    // 时间戳
    archiveClassifier.set('assemble' + "-" + new Date().format('yyyyMMddHHmm'))
    // 版本号
    archiveVersion.set(project.getVersion().toString())

    // 排除配置文件
    exclude('system.properties')
    exclude('log4j.properties')

    // 打包时排除指定的 jar 包
    dependencies {
        exclude(dependency('com.alibaba:fastjson:1.2.47'))
    }

    // 将 build.gradle打入到jar中, 方便查看依赖包版本
    from("./") {
        include 'build.gradle'
    }
}


dependencies {

    testCompile group: 'junit', name: 'junit', version: '4.12'
    compile "org.scala-lang:scala-library:2.11.8"
    compile "org.scala-lang:scala-compiler:2.11.8"
    compile "org.scala-lang:scala-reflect:2.11.8"
    compile group: 'org.apache.spark', name: 'spark-core_2.11', version: '2.3.0'
    compile group: 'org.apache.spark', name: 'spark-sql_2.11', version: '2.3.0'
    compile group: 'org.apache.spark', name: 'spark-streaming_2.11', version: '2.3.0'
    compile group: 'com.alibaba', name: 'fastjson', version: '1.2.47'
}

// 阿里云镜像、spring镜像
repositories {
    mavenLocal()
    maven { url 'https://maven.aliyun.com/repository/public/' }
    maven { url 'https://repo.spring.io/libs-snapshot/' }
}
```
