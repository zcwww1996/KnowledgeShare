[TOC]
# （一） 使用Restful的API对document操作
创建一个名为chao的index，用于保存document的相关数据
设定项|设定值|说明
---|---|--
ElasticSearchURL|http://Linux01:9200| 	可以访问到的Elasticsearch的环境，此处为Linux01
|HTTP方法|PUT|使用PUT进行索引的创建|
|index名称 |/chao|指定创建的索引的名称

## 1.新建索引
### 1.1 方式一

[![创建索引.png](https://i0.wp.com/img.vim-cn.com/9f/8433f01257d5b5bf7fac1a9ad028cbba71afa4.png "样例图片")](https://i0.wp.com/i.loli.net/2019/12/15/Rwuqp56OTxrNagi.png)

点击创建索引

[![创建索引.png](https://i0.wp.com/img.vim-cn.com/f8/594065f2761db3c643ad5c5b12e0317c79c9ba.png "创建索引")](https://i0.wp.com/i.loli.net/2019/12/15/7IQzpg3kdF95rNc.png)

这里我们输入索引名称即可 当然默认分片数是5 副本数是1 

![Image 索引student.png](https://i0.wp.com/img.vim-cn.com/4b/60855bb545cf3cfc56e52c38facca6eb166e49.png)

假如单个机器部署的话 副本是没地方分配的 一般集群都是2台或者2台以上机器集群，副本都不存对应的分片所以机器的，这样能保证集群系统的可靠性。

我们点击"OK" 即可轻松建立索引 以及分片数和副本；

### 1.2 方式二
进入主页，选择 复合查询

![Image 索引student.png](https://i0.wp.com/img.vim-cn.com/95/e442fc5a3616a16ec0356f3cd51985b25ec013.png)
右侧返回索引添加成功信息；

我们返回 概要 首页 点击 刷新 也能看到新建的索引student

## 2.删除索引
### 2.1 概览删除
[![Image 删除索引.png](https://i0.wp.com/img.vim-cn.com/61/360d91523202165e8586b84318a212f7c63c9b.jpg "删除索引")](https://i0.wp.com/i.loli.net/2019/12/16/wY5veFl7aWZLnEA.jpg)

### 2.2 url命令删除  delete
![Image 删除索引.png](https://i0.wp.com/img.vim-cn.com/72/c2cbe9bf9119f28c8b819310fab8e8bc652398.png)


## 3.添加文档
### 3.1 用head插件来实现
这里我们给student索引添加文档 ==post方式==

```url
http://linux01:9200/student/first/1/
```
![Image 添加文档.png](https://i0.wp.com/img.vim-cn.com/06/88138ecf279352955e53b9de5dbcfdcc6f2168.png)


这里student是索引 first是类别 1是id

假如id没写的话 系统也会给我们自动生成一个

假如id本身已经存在 那就变成了修改操作；我们一般都要指定下id

我们输入Json数据，然后点击提交，右侧显示创建成功，当然我们可以验证下json

![Image 浏览数据.png](https://i0.wp.com/img.vim-cn.com/f0/4c4b229f1a7e7baf803263f46b843c9c88d74f.png)
我们可以看到新添加的索引文档

## 4.修改文档
方式和添加一样，只不过我们一定要指定已经存在的id
地址输入：
```url
http://192.168.1.110:9200/student/first/12/
```
![Image 修改文档.png](https://i0.wp.com/img.vim-cn.com/e8/6b5fc85a936fc09e1b07458c013da5b8573f71.png)


## 5.删除文档
### 5.1 通过请求  选择delete方式
![Image 删除文档.png](https://i0.wp.com/img.vim-cn.com/63/f3f9d519f274cfca444e5dc8067a4f9d6f57ba.png)

## 6.查询文档
### 6.1 通过请求  选择get方式
![Image 查询文档.png](https://i0.wp.com/img.vim-cn.com/fb/b47be1ecb2efc5b4431ec1b4ce20d1f689a5ee.png)

### 6.1浏览数据
![Image 浏览数据.png](https://i0.wp.com/img.vim-cn.com/f0/4c4b229f1a7e7baf803263f46b843c9c88d74f.png)



-- 参考网址：http://blog.java1234.com/blog/articles/357.html