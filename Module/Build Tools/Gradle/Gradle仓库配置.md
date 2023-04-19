[TOC]
# 1. Gradle支持的仓库类型
Gradle的使用非常灵活，其中可以设置使用多种类型的仓库，来获取应用中使用的库文件。 
支持的类型有如下几种：

|类型|说明|
|---|---|
|Maven central repository|这是Maven的中央仓库，无需配置，直接声明就可以使用。但不支持https协议访问|
|Maven JCenter repository|JCenter中央仓库，实际也是是用的maven搭建的，但相比Maven仓库更友好，通过CDN分发，并且支持https访问。|
|Maven local repository|Maven本地的仓库，可以通过本地配置文件进行配置|
|Maven repository|常规的第三方Maven仓库，可设置访问Url|
|Ivy repository|Ivy仓库，可以是本地仓库，也可以是远程仓库|
|Flat directory repository|使用本地文件夹作为仓库|

# 2. 几种仓库的使用方法

## 2.1 Maven central repository

在build.gradle中配置

```scala
repositories {
    mavenCentral()
}
```

就可以直接使用了。

## 2.2 Maven JCenter repository

最常用也是Android Studio默认配置：

```scala
repositories {
    jcenter()
}
```

这时使用jcenter仓库是通过https访问的，如果想切换成http协议访问，需要修改配置：

~~~scala
repositories {
    jcenter {
        url "http://jcenter.bintray.com"
    }
}
~~~

## 2.3 Local Maven repository

可以使用Maven本地的仓库。默认情况下，本地仓库位于`USER_HOME/.m2/repository`（例如windows环境中，在C:\Users\NAME.m2.repository），同时可以通过`USER_HOME/.m2/`下的settings.xml配置文件修改默认路径位置。 
若使用本地仓库在build.gradle中进行如下配置：

```scala
repositories {
    mavenLocal()
}
```

## 2.4 Maven repositories

第三方的配置也很简单，直接指明url即可：

```scala
repositories {
    maven {
        url "http://repo.mycompany.com/maven2"
    }
}
```

## 2.5 Ivy repositor

配置如下：

```scala
repositories {
    ivy {
        url "http://repo.mycompany.com/repo"
    }
}
```

## 2.6 Flat directory repository

使用本地文件夹，这个也比较常用。直接在build.gradle中声明文件夹路径：

```scala
repositories {
    flatDir {
        dirs 'lib'
    }
    flatDir {
        dirs 'lib1', 'lib2'
    }
}
```

使用本地文件夹时，就不支持配置元数据格式的信息了（POM文件）。并且Gradle会优先使用服务器仓库中的库文件：例如同时声明了jcenter和flatDir，当flatDir中的库文件同样在jcenter中存在，gradle会优先使用jcenter的。

来源： https://blog.csdn.net/ucxiii/article/details/51943848

# 3. Gradle仓库报错

## 3.1 gradle仓库地址不安全警告

如果有报以下警告：

> Using insecure protocols with repositories, without explicit opt-in, is unsupported.  
> Switch Maven repository 'maven7(http://maven.aliyun.com/repository/public)' to redirect to a secure protocol (like HTTPS) or allow insecure protocols.  
> See https://docs.gradle.org/7.1.1/dsl/org.gradle.api.artifacts.repositories.UrlArtifactRepository.html#org.gradle.api.artifacts.repositories.UrlArtifactRepository:allowInsecureProtocol for more details.

说明你配置了除 maven 中央仓库之外的其他不安全的仓库

修改方式有两种：

**第一种：在仓库前添加关键字**

`allowInsecureProtocol = true`


```Groovy
repositories {
    mavenLocal()
    maven {
        allowInsecureProtocol = true
        url 'http://maven.aliyun.com/repository/public'
    }
    mavenCentral()
}
```

**第二种：把http链接修改成https对应的仓库**

```Groovy
repositories {
    mavenLocal()
    maven {
        url 'https://maven.aliyun.com/repository/public'
    }
    mavenCentral()
}
```