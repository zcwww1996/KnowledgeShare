[TOC]


> 先入为主：本文采用的是gradle7.5版本，gradle各版本之间差异较大，若有不同，请检查是否为版本问题。

# 一、安装

## 1.1 下载安装

网上非常多教程，自行下载安装看，配置环境变量即可，gradle是一个非常新潮的项目，每隔几个月就会发布一个新版本，这种方式可能跟不上gradle的更新速度。

## 1.2 scoop 安装

scoop 安装，对scoop不了解的可以参考我的这篇文章（埋个坑，不定期填），scoop安装的好处就是较为容易地切换gradle版本，缺点就在于国内环境不太适合使用scoop。

## 1.3 Gradle Wrapper（建议）

在IDEA中创建Gradle项目时，会自动生成gradle文件夹，其中就包括`gradle-wrapper.properties`，IDEA默认使用gradle wrapper来创建项目，所以无需安装gradle也可以正常运行。gradle wrapper的优点之一就是可以自定义下载的gradle的版本，如果是团队协作的话，这个功能就非常方便，简单设置即可统一团队的构建工具版本。

**当然，如果你想使用gradle的全局命令的话，还需要你自行修改环境变量配置，需要你到配置的gradle文件夹中找到wrapper文件夹一步步找到对应版本的bin文件夹，并添加到环境变量**

> gradle-wrapper.properties文件内容

    distributionBase=GRADLE_USER_HOME
    distributionPath=wrapper/dists
    distributionUrl=https\://services.gradle.org/distributions/gradle-7.5-bin.zip
    zipStoreBase=GRADLE_USER_HOME
    zipStorePath=wrapper/dists

关于每个字段的意思如下：

1.  **distributionBase：** Gradle 解包后存储的父目录;
2.  **distributionPath：** `distributionBase`指定目录的子目录。`distributionBase+distributionPath`就是 Gradle 解包后的存放的具体目录;
3.  **distributionUrl：** Gradle 指定版本的压缩包下载地址;
4.  **zipStoreBase：** Gradle 压缩包下载后存储父目录;
5.  **zipStorePath：** `zipStoreBase`指定目录的子目录。`zipStoreBase+zipStorePath`就是 Gradle 压缩包的存放位置。

# 二、groovy语言基础

> 内容较多，不便赘述。  
> 参考[https://www.imooc.com/wiki/gradlebase/basis.html](https://link.zhihu.com/?target=https%3A//www.imooc.com/wiki/gradlebase/basis.html)

# 三、gradle构建脚本与任务

> 参考： [https://www.bilibili.com/video/BV1dK411K7Pg/](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1dK411K7Pg/)

注意gradle的build的生命周期

**gradle 构建的生命周期主要分为三个阶段，Initialization，Configuration，Execution。**

1.  **Initialization**：Gradle支持单个或多个工程的构建。在Initialization阶段，Gradle决定哪些工程将参与到当前构建过程，并为每一个这样的工程创建一个Project实例。一般情况下，参与构建的工程信息将在settings.gradle中定义。  
    
2.  **Configuration**：在这一阶段，配置project的实例。所有工程的构建脚本都将被执行。Task，configuration和许多其他的对象将被创建和配置。  
    
3.  **Execution**：在之前的configuration阶段，task的一个子集被创建并配置。这些子集来自于作为参数传入gradle命令的task名字，在execution阶段，这一子集将被依次执行。  
    

# 四、IDEA创建gradle项目

## 4.1 IDEA设置

![](https://pic1.zhimg.com/v2-c4b4c7dbf047606d69f2e2f1edc675d8_r.jpg)


尤其注意两个地方：

\==**Gradle user home**\==：就是gradle-warpper中GRADLE\_USER\_HOME的位置，建议在环境变量中新建GRADLE\_USER\_HOME变量，以后gradle-wrapper下载的gradle以及下载的依赖都会存放在此处。

![](https://pic4.zhimg.com/v2-c5bfad418e7eb459b19be7a1324d4463_r.jpg)


\==**Gradle JVM**\==一定要选择与自己项目一致的版本。

**Use Gradle from**： 可以选择使用自己下载的gradle还是使用gradle wrapper 来管理gradle

## 4.2 新建项目

![](https://pic1.zhimg.com/v2-b5590bf07a5bb2c16d53824d704f7b9c_r.jpg)


![](https://pic4.zhimg.com/v2-cd6d8fe4fb534f4d6106237241faa4ff_r.jpg)


这步很简单，只需要选择gradle作为构建工具即可，下面是基于gradle的spring项目结构。

![](https://pic2.zhimg.com/80/v2-cd06205f81ea46e951ac66d250eccd35_720w.webp)

1.  **gradle**：gradle-wrapper存放位置
2.  **src**:与maven目录一致
3.  **build.gradle**： gradle项目构建文件
4.  **gradlew**：gradle命令行工具
5.  **settings.gradle**: 多模块项目配置文件

> \==注== : 如果在构建项目过程中出现中文乱码的情况，打开`Help->Eidt Custom VM Options`，在Gradle Vm Options中添加设置  
> `-Dfile.encoding=utf-8`，如果还是不行，那就在build.gradle中添加 `gradle tasks.withType(JavaCompile) { options.encoding = "UTF-8" }`

# 五、依赖管理

在新建的gradle项目中，大致有如下内容

![](https://pic1.zhimg.com/v2-fbdfcd30c1386f2b5387c4d47cff4344_r.jpg)


其中：group，version，sourceCompatibility对应maven很好理解。

## 5.1 插件引入方式

plugins闭包出现在比较高版本的gradle中，gradle插件类似于maven插件，`plugs{ id 'java'}`与`apply plugin: 'java'`效果是一样的，plugins闭包里的插件必须是gradle官方插件库（[https://plugins.gradle.org/](https://link.zhihu.com/?target=https%3A//plugins.gradle.org/)）里的，另外plugins块不能放在多项目配置块（allProjects, subProjects）里。因为java是**核心插件**，所以不用指定版本，而org.springframework.boot是**社区插件**，所以必须指定版本

而`apply plguin: 'java'`就比较灵活,以下是apply方法应用插件的方式，

    buildscript {
      repositories {
        maven {
          url "https://plugins.gradle.org/m2/"
        }
      }
      dependencies {
        classpath "org.springframework.boot:spring-boot-gradle-plugin:2.7.4"
      }
    }
    
    apply plugin: "org.springframework.boot"

另外，有时还可以看见下面这种方式

    buildscript {
        ext {
            springBootVersion = "2.3.3.RELEASE"
        }
        repositories {
            mavenLocal()
            maven { url 'http://maven.aliyun.com/nexus/content/groups/public' }
            jcenter()
        }
        dependencies {
            classpath("org.springframework.boot:spring-boot-gradle-plugin:${springBootVersion}")
        }
    }
    apply plugin: 'java'
    apply plugin: 'org.springframework.boot'

这种当时下边就多了个**ext代码块**，ext代码块是提供给用户自定义属性用的，一般用来定义版本，实现版本的集中管理。

gradle官方插件库中有插件的两种引入方式：

![](https://pic1.zhimg.com/v2-a82a63f79cb87ebc68811a0a9cc37600_r.jpg)


## 5.2 buildscript

buildscript中的声明是gradle脚本自身需要使用的资源。可以声明的资源包括**依赖项**、**第三方插件**、**maven仓库**地址等。而在build.gradle文件中直接声明的依赖项、仓库地址等信息是项目自身需要的资源。gradle在执行脚本时，会**优先执行**buildscript代码块中的内容，然后才会执行剩余的build脚本。

buildscript代码块中的repositories和dependencies的使用方式与直接在build.gradle文件中的使用方式几乎完全一样。唯一不同之处是在buildscript代码块中你可以对dependencies使用**classpath**声明。该classpath声明说明了在执行其余的build脚本时，class loader可以使用这些你提供的依赖项。这也正是我们使用buildscript代码块的目的。某种意义上来说，classpath 声明的依赖，不会编译到最终的 jar包里面

另外，buildscript必须位于plugins块和apply plugin之前

## 5.3 依赖引入

gradle依赖写在dependcies代码块中，gradle中引入依赖只需要一行，遵循`scope 'gropId:artifactId:version'`的格式，也可以是`scope (gropId:artifactId:version)`

maven只有compile、provided、test、runtime，而gradle有以下几种scope:

1.  **implementation**：会将指定的依赖添加到编译路径，并且会将该依赖打包到输出，但是这个依赖在编译时不能暴露给其他模块，例如依赖此模块的其他模块。这种方式指定的依赖在编译时**只能在当前模块中访问**。
2.  **api**：==注意：api关键字是由java-library提供，若要使用，请在plugins中添加：`id 'java-library'`\== （在旧版本中作用与compile相同，新版本移除了compile）使用api配置的依赖会将对应的依赖添加到编译路径，并将依赖打包输出，但是这个依赖是可以**传递**的，比如模块A依赖模块B，B依赖库C，模块B在编译时能够访问到库C，但是与implemetation不同的是，在模块A中库C也是可以访问的。
3.  **compileOnly**：compileOnly修饰的依赖会添加到编译路径中，但是**不会被打包**，因此只能在编译时访问，且compileOnly修饰的依赖**不会传递**。
4.  **runtimeOnly**：这个与compileOnly相反，它修饰的依赖**不会添加到编译路径中**，但是能被打包，在运行时使用。和Maven的provided比较接近。
5.  **annotationProcessor**：用于注解处理器的依赖配置。
6.  **testImplementation**：这种依赖在测试编译时和运行时可见，类似于Maven的test作用域。
7.  **testCompileOnly和testRuntimeOnly**：这两种类似于compileOnly和runtimeOnly，但是作用于测试编译时和运行时。
8.  **classpath**：见上一段，classpath并不能在buildscript外的dependcies中使用

**排除依赖**：

    dependencies {
        implementation 'org.springframework.boot:spring-boot-starter-web'
        testImplementation('org.springframework.boot:spring-boot-starter-test') {
            exclude group: 'org.junit.vintage', module: 'junit-vintage-engine'
        }
    }

**全局排除指定环境的依赖**

    configurations {
        runtime.exclude group: "org.slf4j", module: "slf4j-log4j12"
        compile.exclude group: "org.slf4j", module: "slf4j-log4j12"
    }

**全局排除所有环境下的依赖**

    configurations.all {
        exclude group: 'org.springframework.boot', module: 'spring-boot-starter-logging'
    }

## 5.4 仓库管理

gradle仓库可以直接使用maven的仓库，但是gradle下载的jar包文件格式与maven不一样，所以不能和maven本地仓库共用，仓库的配置，是在repository中的

    repositories {
         mavenLocal() //本地仓库
         maven { url 'http://maven.aliyun.com/nexus/content/groups/public' } //外部仓库（阿里云）
         //若这里解析报错，则将http替换为https，或者在url前边添加 allowInsecureProtocol = true
         mavenCentral() // maven 中心仓库
     }

# 六、多模块spring项目搭建

## 6.1 新建空项目

清空build.gradle文件，很多同学新建模块之后老是出现 src目录，就是因为安装了java插件，将java插件删了即可。

![](https://pic1.zhimg.com/v2-f7a4e21a154055e10ff012ed4b83eea4_r.jpg)


## 6.2 新建子模块

![](https://pic2.zhimg.com/v2-f09b852ce9a9066b0746caff4f0dd2a5_r.jpg)


![](https://pic3.zhimg.com/v2-3edee05468112a74a583c84088f97982_r.jpg)


当然，也可以创建springboot项目，但是得修改部分配置

![](https://pic2.zhimg.com/v2-712a1b2530909ef9d89988cd3d766911_r.jpg)


将groupId和package name改成和父工程一致即可，接下来选择你需要的依赖进行创建

![](https://pic2.zhimg.com/80/v2-d81a3c52ad8c7653510f606e105b965d_720w.webp)


创建好之后需要将多余的文件删掉，只留下src目录与build.gradle

![](https://pic3.zhimg.com/80/v2-691261567f513a3e34b916069ab06072_720w.webp)


接着在settings.gradle中添加home项目：

![](https://pic2.zhimg.com/80/v2-7163bef8f922a537461e306660d8cb19_720w.webp)


最后再到gradle窗口中将home删掉，刷新gradle即可

![](https://pic3.zhimg.com/v2-aa7c64e90cba59df7bd862f8ba6f7a7a_r.jpg)


大体框架便搭建好了

![](https://pic1.zhimg.com/80/v2-8bba8da10c0f09335b00e8716f50a058_720w.webp)


## 6.3 父工程build.gradle配置

    allprojects{ // 所有模块/项目的配置
        group = 'org.example'
        version = '0.0.1'
        repositories {
            mavenLocal()
            maven { url 'https://maven.aliyun.com/repository/public/' }
            mavenCentral()
        }
        //指定编码格式
        tasks.withType(JavaCompile) {
            options.encoding = "UTF-8"
        }
    }
    
    subprojects { //子模块配置
        apply plugin: "java"
        apply plugin: 'java-library'
        apply plugin: "idea" //指定编辑器
        //java版本
        sourceCompatibility = 1.8
        targetCompatibility = 1.8
    
        //公共依赖
        dependencies {
            compileOnly 'org.projectlombok:lombok:1.18.20'
            annotationProcessor 'org.projectlombok:lombok:1.18.20'
        }
    }

## 6.4 多模块依赖管理

低版本的用户可以参考[https://blog.csdn.net/sjs\_caomei/article/details/120711701](https://link.zhihu.com/?target=https%3A//blog.csdn.net/sjs_caomei/article/details/120711701)

在7.0+版本，gradle提供了dependencyResolutionManagement来进行多模块之间的依赖共享，类似于maven的dependencyManagement，在7.4以下版本启用该功能，得在`settings.gradle`中添加`enableFeaturePreview('VERSION_CATALOGS')`

用法：

在`settings.gradle`中添加：

    dependencyResolutionManagement{
        versionCatalogs{
            libs{  //libs名字可以任意取，最好为libs，到8.0可能会强制使用libs
               library('hutool-core','cn.hutool:hutool-core:5.8.6') 
               //library在7.4以下版本为alias在7.4+版本被弃用,并且alias用法与library略有区别，详情参考文末链接
            }
        }
    }

library第一个字符串为依赖的别名，注意别名必须由一系列以破折号（`-`，推荐）、下划线 (`_`) 或点 (`.`) 分隔的标识符组成（当然，不写分隔符也可以）。标识符本身必须由`ascii`字符组成，最好是小写，最后是数字。

之后，便可以在子模块`build.gradle`中使用：

    dependencies {
        implementation(libs.hutool.core)
    }

如果几个依赖共用一个版本，比如：

    dependencyResolutionManagement{
        versionCatalogs{
            libs{ 
               library('hutool-core','cn.hutool:hutool-core:5.8.6') 
               library('hutool-json','cn.hutool:hutool-json:5.8.6') 
               library('hutool-http','cn.hutool:hutool-http:5.8.6') 
            }
        }
    }

可以使用`version`来同意管理版本（注意，library中groupid与artifactid用‘，’隔开了），dependencies中用法一致

    dependencyResolutionManagement{
        versionCatalogs{
            libs{
                version('hutool','5.8.6')
                library('hutool-core','cn.hutool','hutool-core').versionRef('hutool')
                library('hutool-http','cn.hutool','hutool-http').versionRef('hutool')
                library('hutool-json','cn.hutool','hutool-json').versionRef('hutool')
            }
        }
    }

要是不想一个个引用这几依赖的话，还可以使用bundles将几个依赖绑定到一起，一次性引入多个依赖：

    dependencyResolutionManagement{
        versionCatalogs{
            libs{
                version('hutool','5.8.6')
                library('hutool-core','cn.hutool','hutool-core').versionRef('hutool')
                library('hutool-http','cn.hutool','hutool-http').versionRef('hutool')
                library('hutool-json','cn.hutool','hutool-json').versionRef('hutool')
                bundle('hutool',['hutool-core','hutool-http','hutool-json'])
            }
        }
    }

在dependencies中：

    dependencies {
        implementation(libs.bundles.hutool)
    }

![](https://pic2.zhimg.com/v2-86f525cc7e143f00481fb83620034cf1_r.jpg)


同时，插件的版本也可以管理：

    dependencyResolutionManagement{
        versionCatalogs{
            libs{
                 plugin('spring-dependency','io.spring.dependency-management').version('1.0.14.RELEASE')
            }
        }
    }

![](https://pic2.zhimg.com/v2-c14de957f9d84d3ccb247395ae1963ad_r.jpg)


## 6.5 模块之间的互相依赖

只需要在dependencies中按照如下格式依赖即可，并且也是遵循依赖引入的几种scope规则的

![](https://pic3.zhimg.com/v2-35c289caa776332643c5fa90ceabed1a_r.jpg)


要注意的是，被引用项目的类必须在软件包下，才可以被找到，不过，做项目过程中应该也不会出现没有软件包的情况吧

## 6.5 项目发布到maven 私服

`build.gradle`:

    plugins {
        id 'java-library'
        id 'maven-publish'
    }
    
    publishing {
        publications {
            myLibrary(MavenPublication) {
                from components.java
            }
        }
        repositories {
            maven {
    //            default credentials for a nexus repository manager
                credentials {
                    username 'admin'
                    password 'xxxxx'
                } 
                allowInsecureProtocol = true //允许http
                // 发布maven存储库的url
                url "http://localhost:8080/nexus/content/repositories/releases/"
            }
        }
    }

最后只需要在右侧的gradle工具窗口找到publish就可以发布到远程仓库了

![](https://pic1.zhimg.com/v2-15fced7ba7e44ae2a8521c996bf78d94_r.jpg)


> 对于publications的相关内容的翻译，自行理解吧，实在肝不动了  
> 它定义了一个名为“myLibrary”的发布，可以根据其类型(MavenPublication)将其发布到Maven存储库。该发布仅由生产JAR工件及其元数据组成，它们结合在一起由项目的java组件表示。  
> 组件是定义发布的标准方法。它们是由插件提供的，通常是同一语言或平台的插件。例如，Java插件定义了components.java SoftwareComponent，而War插件定义了components.web。

更加详细的内容参考官方文档：[https://docs.gradle.org/7.5.1/userguide/publishing\_maven.html](https://link.zhihu.com/?target=https%3A//docs.gradle.org/7.5.1/userguide/publishing_maven.html)

然后，本文到此结束！

# 参考：

> Gradle使用教程：[https://zhuanlan.zhihu.com/p/440595132](https://zhuanlan.zhihu.com/p/440595132)  
> Gradle 入门教程：[https://www.imooc.com/wiki/gradlebase/intro.html](https://link.zhihu.com/?target=https%3A//www.imooc.com/wiki/gradlebase/intro.html)  
> gradle快速入门（最通俗易懂教程）：[https://www.bilibili.com/video/BV1dK411K7Pg/](https://link.zhihu.com/?target=https%3A//www.bilibili.com/video/BV1dK411K7Pg/)  
> Gradle中的buildScript代码块：[https://www.jianshu.com/p/322f1427401b](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/322f1427401b)  
> Gradle依赖之‘五种依赖配置’ ：[https://zhuanlan.zhihu.com/p/110215979](https://zhuanlan.zhihu.com/p/110215979)  
> Gradle 排除依赖：[https://www.jianshu.com/p/c23bda40f487](https://link.zhihu.com/?target=https%3A//www.jianshu.com/p/c23bda40f487)  
> 【Gradle7.0】依赖统一管理的全新方式，了解一下~：[https://juejin.cn/post/6997396071055900680](https://link.zhihu.com/?target=https%3A//juejin.cn/post/6997396071055900680)  
> gradle官方文档： [https://docs.gradle.org/](https://link.zhihu.com/?target=https%3A//docs.gradle.org/)