[TOC]

# 如何删除GitHub或者GitLab上的文件夹
## 方法一

这里以删除 .setting 文件夹为案例

```bash
git rm -r --cached  .setting #--cached不会把本地的.setting删除
git commit -m 'delete .setting dir'
git push -u origin master
```

## 方法二
如果误提交的文件夹比较多，方法一也较繁琐

```bash
直接修改.gitignore文件,将不需要的文件过滤掉，然后执行命令
git rm -r --cached .
git add .
git commit
git push  -u origin master
```

# git push上传代码到gitlab上，报错401/403(或需要输入用户名和密码)

来源： https://www.cnblogs.com/kevingrace/p/6118297.html

之前部署的gitlab，采用<font style="background-color: Yellow
;">ssh</font>方式连接gitlab，在客户机上产生公钥上传到gitlab的SSH-Keys里，git clone下载和git push上传都没问题，这种方式很安全。

后来应开发同事要求采用http方式连接gitlab，那么首先将project工程的“Visibility Level”改为“Public”公开模式，要保证<font style="background-color: Yellow
;">gitlab的http端口已对客户机开放</font>。

后面发现了一个问题:

http方式连接gitlab后，git clone下载没有问题，但是git push上传有报错：

> error: The requested URL returned error: 401 Unauthorized while accessing http://git.xqshijie.net:8081/weixin/weixin.git/info/refs
> fatal: HTTP request failed

或者

> The requested URL returned error: 403 Forbidden while accessing

## 解决办法：

在代码的.git/config文件内[remote "origin"]的url的gitlab域名前添加gitlab注册时的“<font style="background-color: Yellow
;color: red;">**用户名:密码@**</font>”

另外发现这个用户要在对应项目下的角色是<font color="red">**Owner**</font>或<font color="red">**Master**</font>才行，如果是<font color="blue">Guest、Reporter、Developer</font>，则如下操作后也是不行。

如下，gitlab的用户名是**xiaobaitu**，假设密码是**baiyoubai**
然后修改代码里的 **.git/config**文件

```bash
[root@test-huanqiu weixin]# cd .git
[root@test-huanqiu .git]# cat config 
[core]
repositoryformatversion = 0
filemode = true
bare = false
logallrefupdates = true
[remote "origin"]
fetch = +refs/heads/*:refs/remotes/origin/*
url = http://gitee.com/zc_vip/DemandSidePlatform.git
[branch "master"]
remote = origin
merge = refs/heads/master
```

修改如下：

```bash
[remote "origin"]
fetch = +refs/heads/*:refs/remotes/origin/*
url = http://xiaobaitu:baiyoubai@gitee.com/zc_vip/DemandSidePlatform.git
```

然后再次git push，发现可以正常提交了！

# Push to origin/master was rejected

## 【问题描述】
　　在使用Git Push代码的时候，会出现 Push to origin/master was rejected 的错误提示。

　　在第一次提交到代码仓库的时候非常容易出现，因为初始化的仓库和本地仓库是没有什么关联的，因此，在进行第一次的新代码提交时，通常会出现这个错误。

## 【问题原因】
　　远程仓库和本地仓库的内容不一致
　　
## 【解决方法】
在git项目对应的目录位置打开Git Bash

[![Git Bash](https://cdn7.232232.xyz/58/2022/09/05-63155d574bd4f.png)](https://pic.downk.cc/item/5eb27637c2a9a83be5f507ad.png)

然后在命令窗输入下面命令：

[![窗口输入命令](https://sc04.alicdn.com/kf/H81212b289b61411585b9e7ee0642fadfs.png)](https://pic.downk.cc/item/5eb27637c2a9a83be5f507b0.png)

```bash
git pull origin master --allow-unrelated-histories
```


最后出现完成信息，则操作成功！

再次**Push**代码，可以成功进行提交！！！


来源： https://www.cnblogs.com/shenyixin/p/8310226.html