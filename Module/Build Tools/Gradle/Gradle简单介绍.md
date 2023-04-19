[TOC]
# 1. 简单介绍
Gradle是一个好用的构建工具
使用它的原因是
1. 配置相关依赖代码量少，不会像maven一样xml过多
2. 打包编译测试发布都有，而且使用起来方便
3. 利用自定义的任务可以完成自己想要的功能
 
# 2. 安装使用
## 2.1 软件安装
1. 下载地址http://services.gradle.org/distributions/
2. 下载你所需要对应的版本，gradle-4.3.1-bin.zip
3. 下载后解压到你想要的目录
4. 设置环境变量GRADLE_HOME

[![GRADLE_HOME](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_vk8lr.thumb.700_0.png "GRADLE_HOME")](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_vk8lr.thumb.700_0.png)

[![Path](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_uNCtr.thumb.700_0.png "Path")](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_uNCtr.thumb.700_0.png)
5. 在cmd模式下查看，出现以下信息证明安装成功

[![安装成功](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_uVeQF.thumb.700_0.png "安装成功")](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_uVeQF.thumb.700_0.png)

## 2.2 idea使用
### 2.2.1 创建一个web的Gradle项目
1. 使用gradle模板

[![](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_CwWdA.png)](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_CwWdA.png)
2. 填写包名、域名

[![](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_ywvH3.png)](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_ywvH3.png)
3. 关联本地gradle

[![](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_CwWdA.png)](https://c-ssl.duitang.com/uploads/item/201912/23/2019122390332_CwWdA.png)

### 2.2.2 如何进行打war包
1. View -> Tool Windows -> Gradle

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_AuQKs.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_AuQKs.thumb.700_0.png)
2. 双击war

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_zQGNk.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_zQGNk.thumb.700_0.png)
3. 打包完成之后的war文件位置

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_d4nv3.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_d4nv3.thumb.700_0.png)

4. 然后把war放入对应的tomcat目录即可，这里就不多解释了

## 2.3 Gradle生成jar包
### 2.3.1 利用打入shadow插件打入第三方依赖

```gradle
group 'cn.com.ctsi'
version '1.0-SNAPSHOT'

buildscript {
    repositories {
        maven {
            url 'https://maven.aliyun.com/repository/public/'
        }
    }
    dependencies {
        classpath 'com.github.jengelman.gradle.plugins:shadow:4.0.1'
    }
}

apply plugin: 'java'
apply plugin: 'idea'
apply plugin: 'com.github.johnrengelman.shadow'
```


### 2.3.2 java和scala混合jar包

```gradle
 // 这里是关键（把java与scala的源代码目录全映射到scala上,
 // 这样gradle compile Scala时就能同时编译java与scala的源代码）
sourceSets {
    main {
        scala {
            srcDirs = ['src/main/scala', 'src/main/java']
        }
        java {
            srcDirs = []
        }
    }

    test {
        scala {
            srcDirs = ['src/test/scala', 'src/test/java']
        }
        java {
            srcDirs = []
        }
    }
}
```
## 2.4 解释build.gradle和settings.gradle
首先是一个项目包含group、name、version
settings.gradle是用来管理多项目的，里面包含了项目的name

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_GPuNE.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_GPuNE.thumb.700_0.png)
 
在build.gradle中，apply是应用的插件，如：

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_ePVYK.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_ePVYK.thumb.700_0.png)

这里我们用了java和war的插件
 
dependencies是用于声明这个项目依赖于哪些jar

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_ZLVrV.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_ZLVrV.thumb.700_0.png)
 
这里说明的是，测试编译阶段我们依赖junit的jar
其中包括complile（编译时）runtime（运行时）testCompile（测试编译时）testRuntime（测试运行时）
 
repositories是一个仓库gradle会根据从上到下的顺序依次去仓库中寻找jar

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_MfLhN.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_MfLhN.thumb.700_0.png)

这里我们默认的是一个maven的中心仓库
从gradle源代码中我们看到地址是这样的<br>
[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_VLjNU.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_VLjNU.thumb.700_0.png)
 
这里可以配置

```gradle
mavenLocal()    使用本地maven仓库
mavenCentral()  使用maven中心仓库
maven{
url '你的地址'
}
```

使用固定的地址，这里可以使用阿里云的镜像下载速度会快一些，然后也可以使用公司内部的私服地址
`maven {url 'http://maven.aliyun.com/nexus/content/groups/public/'}`
 
## 2.5 有关gradle的jar冲突
默认情况下，如果有jar包冲突，gradle会自动依赖两个冲突jar包最新的一个版本，所以默认不需要进行管理。
如果真的出现无法解决的冲突，gradle也会出现明显的冲突提示，所以不需要担心

在Hadoop依赖中排除jersey-core、servlet-api
```groovy
implementation(group: 'org.apache.hadoop', name: 'hadoop-common', version: '2.6.5') {
        exclude group: 'com.sun.jersey', module: "jersey-core"
        exclude group: 'javax.servlet', module: "servlet-api"
    }
```


## 2.6 仓库配置
### 2.6.1 本地仓库配置
gradle会下载相关需要依赖的jar包，默认的本地存放地址是：C:/Users/(用户名)/.gradle/caches/modules-2/files-2.1

很多人和我一样不愿意放在C盘，所以需要修改位置，网上说很简单，只需要添加一个环境变量就可以了，如下

配置环境变量GRADLE_USER_HOME，并指向你的一个本地目录，用来保存Gradle下载的依赖包。

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_A3V8j.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220170608_A3V8j.thumb.700_0.png)

但是对于IDEA来说木有用（当然上面的环境变量还是要添加的），在IDEA中使用gradle需要修改下面的路径

[![](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_YnMMc.thumb.700_0.png)](https://c-ssl.duitang.com/uploads/item/201912/20/20191220164338_YnMMc.thumb.700_0.png)

##############</br>
ps：IDEA2019中 gradle发生如下变化，在gradle通用配置页面可以配置gradle user home（gradle的仓库位置），use gradle home（gradle工具的路径）

[![idea2019_gradle配置](https://s2.ax1x.com/2020/02/18/3Fdz7t.png)](https://images.weserv.nl/?url=https://files.catbox.moe/prvkti.png)
这样修改之后你就可以发现已经在自定义路径下载jar了

### 2.6.2 远程仓库配置
一般Gradle、maven从中央仓库mavenCentral() `http://repo1.maven.org/maven2/`下载依赖包，但是在国内下载速度巨慢，我们只能使用国内的镜像。 
所以每个Gradle构建的项目中，我们可以在build.gradle做如下配置

```gradle
repositories {
    maven {
        url 'https://maven.aliyun.com/repository/public/'
    }
    mavenCentral()
```

每个项目都如此配置难免麻烦些，我们可以配置一个全局配置文件。

### 2.6.3 配置文件加载优先级
根据gradle官方的userguide, 大概意思是说使用mavenLocal()配置maven的本地仓库后，gradle默认会按以下顺序去查找本地的仓库：`USER_HOME/.m2/settings.xml` >> `M2_HOME/conf/settings.xml` >> `USER_HOME/.m2/repository`。所以，**先配置mavenLocal()，gradle会优先从本地加载依赖**。

**编辑安装目录 init.d\init.gradle**


```gradle
def repoConfig = {
    all { ArtifactRepository repo ->
        if (repo instanceof MavenArtifactRepository) {
            def url = repo.url.toString()
            if (url.contains('repo1.maven.org/maven2') || url.contains('jcenter.bintray.com')) {
                println "gradle init: (${repo.name}: ${repo.url}) removed"
                remove repo
            }
        }
    }
    maven { url 'https://maven.aliyun.com/repository/public' }
    maven { url 'https://repo.huaweicloud.com/repository/maven' }
    maven { url 'https://maven.aliyun.com/repository/apache-snapshots' }
    maven { url 'https://maven.aliyun.com/repository/gradle-plugin' }
	maven { url 'https://repository.cloudera.com/artifactory/cloudera-repos' }
	maven { url 'https://repo.hortonworks.com/content/repositories/releases' }
}

allprojects {
    buildscript {
        repositories repoConfig
    }

    repositories repoConfig
}
```


# 3. Gradle项目中文乱码的解决办法
## 3.1 解决办法一：
在build.gradle文件中添加如下一段，rhGradle将文件识别为UTF-8编码。当然，这需要你的项目文件本来就是UTF-8编码的，如果默认是GBK编码，就不需要更改。


```gradle
tasks.withType(JavaCompile) {
    options.encoding = "UTF-8" 
}

tasks.withType(ScalaCompile) {
    options.encoding = "UTF-8"
}
```

看了一下官方文档的说明，对于Gradle来说编码这个属性默认情况下是null，也就是Gradle会根据你操作系统的版本来选择编码。Windows中文操作系统的编码正是GBK。所以才会出现这个错误。

## 3.2 解决方法二：
使用环境变量。在Windows下，新建GRADLE_OPTS环境变量，值为-Dfile.encoding=utf-8。然后新开一个终端窗口再次使用gradle命令，就会发现这下Gradle已经可以正确识别编码了。

如果使用IDE进行Gradle操作，那么还需要设置IDE的参数。例如在IDEA中，需要打开**File->Other Settings->Default Settings->Gradle**，在Gradle Vm Options中设置-Dfile.encoding=utf-8。这样IDEA中的Gradle也可以正确执行Gradle命令了。

新版IDEA2019，需要打开**Help->Edit Custom VM options**,在配置文件中追加`-Dfile.encoding=UTF-8`

# 4. init.gradle简介
init.gradle文件在build开始之前执行，所以你可以在这个文件配置一些你想预先加载的操作

例如配置build日志输出、配置你的机器信息，比如jdk安装目录，配置在build时必须个人信息，比如仓库或者数据库的认证信息，and so on.

启用init.gradle文件的方法： 
1. 在命令行指定文件，例如：gradle –init-script yourdir/init.gradle -q taskName.你可以多次输入此命令来指定多个init文件 
2. 把init.gradle文件放到USER_HOME/.gradle/ 目录下. 
3. 把以.gradle结尾的文件放到USER_HOME/.gradle/init.d/ 目录下. 
4. 把以.gradle结尾的文件放到GRADLE_HOME/init.d/ 目录下.

如果存在上面的4种方式的2种以上，gradle会按上面的1-4序号依次执行这些文件，如果给定目录下存在多个init脚本，会按拼音a-z顺序执行这些脚本

类似于build.gradle脚本，init脚本有时groovy语言脚本。每个init脚本都存在一个对应的gradle实例，你在这个文件中调用的所有方法和属性，都会委托给这个gradle实例，每个init脚本都实现了Script接口

下面的例子是在build执行之前给所有的项目制定maven本地库，这个例子同时在 build.gradle文件指定了maven的仓库中心，注意它们之间异同

**build.gradle**

```gradle
repositories {
    mavenCentral()
}

task showRepos << {
    println "All repos:"
    println repositories.collect { it.name }
}
```
**init.gradle**

```gradle
allprojects {
    repositories {
        mavenLocal()
    }
}
```

在命令行输入命令:`gradle –init-script init.gradle -q showRepos`

```gradle
gradle --init-script init.gradle -q showRepos
All repos:
[MavenLocal, MavenRepo]
```

