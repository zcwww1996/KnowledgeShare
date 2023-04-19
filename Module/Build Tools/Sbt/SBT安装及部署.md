来源： https://blog.csdn.net/zrc199021/article/details/73543210

sbt参考链接: http://wiki.jikexueyuan.com/project/sbt-getting-started/

[TOC]

# 1. 在Windows中安装sbt
## 1.1 下载
官网:http://www.scala-sbt.org/

github:https://github.com/sbt/sbt/releases/download/v0.13.15/sbt-0.13.15.msi<br/>
(官网的地址好像下到一半就失败.)

## 1.2 安装
### 1.2.1 安装 sbt-0.13.15.msi

注意安装路径不要有中文或者空格, 最好放到根目录下如:`D:\soft\sbt`

![SBT_HOME.png](https://i0.wp.com/i.loli.net/2020/01/20/832SwIEQifa4YDF.png)
### 1.2.2 配置环境变量:

再在Path中添加:

![Path.png](https://i0.wp.com/i.loli.net/2020/01/20/do5cisFkwbLKWrT.png)

### 1.2.3 修改配置文件
修改 安装目录下conf/sbtconfig.txt, 我的内容如下:

```properties
-Dsbt.log.format=true 
-Dfile.encoding=UTF8 
-Dsbt.ivy.home=D:\Repository\Sbt\.ivy2
-Dsbt.global.base=D:\Repository\Sbt\.sbt
-Dsbt.repository.config=D:\Repository\Sbt\conf\repo.properties
-Xmx512M 
-Xss2M 
-XX:+CMSClassUnloadingEnabled
```

关于 几个-D开头的配置, 可参考下图(更多信息参考官网链接:Command Line Options):

[![sbt配置.png](https://i0.wp.com/i.loli.net/2020/01/20/7eoacqNy4BhnsbA.png)](https://pic.downk.cc/item/5e256ea92fb38b8c3c9b9ffd.png)

### 1.2.4 修改仓库地址为阿里云

(如果在上面配置中没有写-Dsbt.repository.config, 那么默认的位置就在用户目录/.sbt/下新建repositories文件), 文件内容:

```bash
[repositories]
  local
  aliyun nexus:https://maven.aliyun.com/nexus/content/groups/public/
```

### 1.2.5 下载依赖 
配置完后, 命令行输入: `sbt` , 会下载一些必须的依赖, 如果其中有失败就多试几次. 下完后就可以使用了

# 2. 在IDEA中安装sbt
为了更好的在idea中使用sbt, 最好安装下相关插件

![sbt插件.png](https://i0.wp.com/i.loli.net/2020/01/22/RV9Uj13Wwvakq4L.png)

安装好重启后,需要对其进行配置, 以免该插件下载的依赖又跑到其他路径上. 

复制sbtconfig.txt里的内容到如下输入框中:<br/> 
![sbt插件设置1.png](https://i0.wp.com/i.loli.net/2020/01/22/8ByoZtmTxjlQ1sF.png)

[![sbt插件设置2.png](https://i0.wp.com/i.loli.net/2020/01/22/U1oDAjMhKf5eQwN.png)](https://s2.ax1x.com/2020/01/22/1AMhTK.png)

这是default settings, settings里最好也设置一下

除了上述位置以外, 还有两处配置, 最好也相似的配置一下(): 
Settings 和 default Settings => other Settings => SBT 

[![sbt_vm设置.png](https://i0.wp.com/i.loli.net/2020/01/22/CbfuALmYqgXkUMD.png)](https://s2.ax1x.com/2020/01/22/1AQQXR.png)

其二：

![sbt项目设置1.png](https://i0.wp.com/i.loli.net/2020/01/22/pZTxSoFhNJKcGQ1.png)

[![sbt项目设置2.png](https://i0.wp.com/i.loli.net/2020/01/22/VXxGAjIt3O2hdDz.png)](https://s2.ax1x.com/2020/01/22/1AQ1n1.png)

# 3. 问题
## 3.1 sbt存在https的问题

![sbt-https.png](https://i0.wp.com/i.loli.net/2020/01/22/lDU3jLGSrwxs78F.png)

下载官方的 sbt-launch.jar，解压后修改配置文件，再重新打包
需要修改其中的 ./sbt/sbt.boot.properties 文件（`vim ./sbt/sbt.boot.properties`），将 [repositories] 处修改为如下内容（即增加了一条aliyun镜像，并且将原有的 https 镜像都改为相应的 http 版）：

```properties
[repositories]
  local
  aliyun nexus:https://maven.aliyun.com/repository/public/
  jcenter: http://jcenter.bintray.com/
  typesafe-ivy-releases: http://repo.typesafe.com/typesafe/ivy-releases/, [organization]/[module]/[revision]/[type]s/[artifact](-[classifier]).[ext], bootOnly
  maven-central: http://repo1.maven.org/maven2/
```

保存后，重新打包jar：

```bash
rm ./sbt-launch.jar         # 删除旧的
jar -cfM ./sbt-launch.jar .       # 重新打包
ls | grep -v "sbt-launch.jar" | xargs rm -r    # 解压后的文件已无用，删除
```

注意打包时，需要使用 -M 参数，否则 ./META-INF/MANIFEST.MF 会被修改，导致运行时会出现 “./sbt-launch.jar中没有主清单属性” 的错误。