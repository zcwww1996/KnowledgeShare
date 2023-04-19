近开发spark项目使用到scala语言，这里介绍如何在idea上使用sbt来编译项目。

开发环境：windows

# 1. 下载sbt

http://www.scala-sbt.org/download.html

我使用的是zip包，下载后解压到d:\tool\目录

# 2. 添加配置

## 2.1 修改sbtconfig.txt
打开D:\tool\sbt\conf\sbtconfig.txt，在最后添加下面几行配置，注意指定的目录和文件

```properties
-Dsbt.log.format=true 
-Dfile.encoding=UTF8 
-Dsbt.ivy.home=D:\Repository\Sbt\.ivy2
-Dsbt.global.base=D:\Repository\Sbt\.sbt
-Dsbt.repository.config=D:\Repository\Sbt\conf\repo.properties
-Dsbt.coursier.home=D:\Repository\Sbt\Coursier
-XX:+CMSClassUnloadingEnabled
```

 第三行sbt.ivy.home指定了本地自定义的repository路径（如果不设置就是默认的用户目录C:\Users\Administrator\.ivy2）

## 2.2 添加镜像网站
在D:/tool/sbt/conf/目录下新建repo.properties文件，填写下面内容，指定镜像站的地址：

```properties
[repositories]
[repositories]
  local
  aliyun:https://maven.aliyun.com/repository/public
```
## 2.3 添加环境变量
在环境变量PATH中添加`SBT_HOME=D:\Program Files (x86)\Sbt\sbt-1.6.2\bin`

# 3. Idea中设置

## 3.1 在idea中确保正确安装了scala插件

## 3.2 添加sbt jvm参数
文件 -> 其他设置 -> 默认设置中如下设置

[![1AlRIK.png](https://www.helloimg.com/images/2022/09/27/ZKGsEb.jpg)](https://gd-hbimg.huaban.com/cef2ffeaad2693c3ef98a501e6d194938e77b90a6768-J1haiy)

VM parameters:

```properties
-Dsbt.log.format=true 
-Dfile.encoding=UTF8 
-Dsbt.ivy.home=D:\Repository\Sbt\.ivy2
-Dsbt.global.base=D:\Repository\Sbt\.sbt
-Dsbt.repository.config=D:\Repository\Sbt\conf\repo.properties
-Dsbt.coursier.home=D:\Repository\Sbt\Coursier
-XX:+CMSClassUnloadingEnabled
```

来源： https://www.cnblogs.com/30go/p/7909630.html