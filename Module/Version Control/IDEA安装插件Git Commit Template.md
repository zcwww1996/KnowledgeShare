[TOC]

最近 项目代码版本管理迁移到了git，所以我们都使用git提交代码。但是提交的massage优点乱，如果统一化标准化的就很容易让人理解。我发现其实idea对此已经有一个很好的插件来支持了。那就是Git Commit Template

# 1. Git Commit message规范

常用的Git Commit message规范采用的是[Angular 规范](https://docs.google.com/document/d/1QrDFcIiPjSLDn3EL15IJygNPiHORgU1_OOAqWjiDU5Y/edit#heading=h.greljkmo14y0)。

本文参考阮老师的文章介绍：

http://www.ruanyifeng.com/blog/2016/01/commit\_message\_change\_log.html.

Angular规范中定义的格式有3个内容：

*   Header
*   Body
*   Footer

```bash
<type>(<scope>): <subject>
// 空一行
<body>
// 空一行
<footer>

```

## 1.1 Header

Header部分有3个字段： **type(必需), scope(可选), subject(必需)**

> type(必需) ： Type of change：commit的类别；
> 
> scope(可选)：Scope of this change：此次commit的影响模块；
> 
> subject(必需)：Short description：简短的描述此次代码变更的主要内容

### 1.1.1 type

type用于说明commit的类别，常用的标识如下：

*   **feat：新功能（feature）**
*   **fix：修补bug**
*   docs：文档（documentation）
*   style： 格式（不影响代码运行的变动,空格,格式化,等等）
*   refactor：重构（即不是新增功能，也不是修改bug的代码变动
*   perf: 性能 (提高代码性能的改变)
*   test：增加测试或者修改测试
*   build: 影响构建系统或外部依赖项的更改(maven,gradle,npm 等等)
*   ci: 对CI配置文件和脚本的更改
*   chore：对非 src 和 test 目录的修改
*   revert: Revert a commit

最常用的就是feat合fix两种type；

### 1.1.2 scope

`scope`用于说明 commit 影响的范围，比如数据层、控制层、视图层等等，视项目不同而不同。

### 1.1.3 subject

`subject`是 commit 目的的简短描述，不超过50个字符，主要介绍此次代码变更的主要内容。

举个例子：  
eg: feat(订单模块)：订单详情接口增加订单号字段

其中， feat对应type字段；订单模块对应scope(若果scope有内容，括号就存在)；“订单详情接口增加订单号字段”对应subject，简要说明此次代码变更的主要内容。

## 1.2 Body

Body 部分是对本次 commit 的详细描述，可以分成多行。

如：

（1）增加订单号字段；

（2）增加了订单退款接口；

日常项目开发中，如果Header中**subject**已经描述清楚此次代码变更的内容后，Body部分就可以为空。

## 1.3 Footer

（1）不兼容变动

（2）关闭 Issue

日常项目中开发，Footer不常用，可为空。

## 1.4 Revert

若需要撤销上一次的commit，header部分为：`revert: 上一次commit的header内容`；

body部分为：`This reverts commit xxx`，xxx是上一次commit对应的SHA 标识符。

# 2. IDEA插件Git Commit Template使用

## 2.1 idea安装git commit template插件

![](https://img-blog.csdnimg.cn/20200108162815918.png)

![](https://img-blog.csdnimg.cn/20200108162907764.png)

## 2.2 重启idea

## 2.3 选择要提交的文件，右击，如下图：

![](https://img-blog.csdnimg.cn/20200108163153698.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L2xzX2NhbGw1MjA=,size_16,color_FFFFFF,t_70)

![](https://img-blog.csdnimg.cn/20200108163646855.png)

填写对应的内容，提交即可

![](https://img-blog.csdnimg.cn/20200108163742675.png)

type用于说明commit的类别,常用的标识如下：

- **feat**：新功能（feature）  
- **fix**：修补bug  
- **docs**：文档（documentation）  
- **style**：格式（不影响代码运行的变动,空格,格式化,等等）  
- **refactor**：重构（即不是新增功能，也不是修改bug的代码变动  
- **perf**：性能 (提高代码性能的改变)  
- **test**：增加测试或者修改测试  
- **build**：影响构建系统或外部依赖项的更改(maven,gradle,npm 等等)  
- **ci**：对CI配置文件和脚本的更改  
- **chore**：对非 src 和 test 目录的修改（日常事务; 例行工作; 令人厌烦的任务; 乏味无聊的工作;）  
- **revert**：Revert a commit