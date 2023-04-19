[TOC]

推荐！手把手教你使用Git

衔接：http://blog.jobbole.com/78960/?from=singlemessage

# idea 把一个add到Git的文件去掉

https://blog.csdn.net/Dongguabai/article/details/84074087?depth_1-utm_source=distribute.pc_relevant.none-task&utm_source=distribute.pc_relevant.none-task

# 在IDEA上Git的入门使用（IDEA+Git)

链接：https://blog.csdn.net/weixin_39274753/article/details/79722522

前言：Git是目前最常用的版本控制系统，而IDEA又是目前日渐流行的ide，因此现在来介绍在IDEA上Git的入门使用。

</br>
准备：Git、IDEA、GitHub账号

开始之前先创建一个简单的测试项目

![E5aFg.jpg](https://pic.downk.cc/item/5f24106814195aa594a8e87a.png)

## 将代码交由Git管理

![E5xhe.jpg](https://pic.downk.cc/item/5f24106814195aa594a8e87e.png)

VCS  ——>  Enable Version Control Integration...

![image](https://pic.downk.cc/item/5f24106814195aa594a8e885.png)

 ——>  选择要使用的版本控制系统，选择Git  ——>  OK
 
 [![image](https://www.helloimg.com/images/2022/08/22/Zpati1.jpg)](https://pic.imgdb.cn/item/6303347b16f2c2beb1e9720d.jpg)
 
完成后，IDEA下方会出现上述提示。到此，已将本项目与Git进行关联，即已将本项目交由Git管理。


## 将代码提交到本地仓库（commit）

[![VCS Operations Popup](https://www.helloimg.com/images/2022/08/22/ZpasoT.jpg)](https://s6.jpg.cm/2022/08/22/PVWaI6.jpg)

将项目交由Git管理后再点击VCS，会发现列举出的选项发生了变化。

VCS  ——>  VCS Operations Popup...<br>
![EPr6V.jpg](https://pic.downk.cc/item/5f24106814195aa594a8e882.png)<br>
点击VCS Operations Popup...后出现的是Git所能进行的操作，因为是提交到本地，所以点击commit

——>  commit...

![commit](https://pic.downk.cc/item/5f24106814195aa594a8e88b.png)

然后出现以下窗口，窗口上面部分是选择要提交的文件，Commit Message部分的填写每次提交的备忘信息

——>  commit

!["提交备注"](https://pic.downk.cc/item/5f24168914195aa594ab7a6c.png)

提交前IDEA会提醒项目存在问题，选择review会去查看问题，选择commit会忽略问题直接提交。

此处选择的是commit。然后ide下方会出现一条绿色提示

![EP5Fz.jpg](https://pic.downk.cc/item/5f24168914195aa594ab7a66.png)

到此已将代码提交到本地仓库。 

![EPPlP.jpg](https://pic.downk.cc/item/5f24168914195aa594ab7a6e.png)

需要注意的是，本地仓库地址默认就是项目地址

![EPj1w.jpg](https://pic.downk.cc/item/5f24168914195aa594ab7a64.png)


## 查看代码的提交历史

右击项目  ——>  Git  ——>  Show History

![EP1zQ.jpg](https://pic.downk.cc/item/5f24168914195aa594ab7a6a.png)

屏幕下方的区域会展示项目的提交历史，双击其中选项，会详细展示每一次的提交内容<br>
[![提交历史](https://pic.downk.cc/item/5f2417a714195aa594abdcf1.png)](https://shop.io.mi-img.com/app/shop/img?id=shop_d9169843333b23245770dee594046115.png)
[![提交历史2](https://shop.io.mi-img.com/app/shop/img?id=shop_b5fade2c91b28ef09bcd20750d587f40.png)](https://ps.ssl.qhmsg.com/t0291b28ef09bcd2075.jpg)

（此处进行了2次提交，第1次只提交了.java文件，第2次一并提交了该项目的其他文件）


## 将代码提交到远程仓库（push）

VCS  ——>  VCS Operations Popup...  ——>  Push...

![EPV7q.jpg](https://pic.downk.cc/item/5f2417a714195aa594abdcec.png)

出现上述窗口，因为还没选择要连接的远程仓库，因此需要明确远程仓库

——>  Define remote

![image](https://pic.downk.cc/item/5f2417a714195aa594abdcee.png)

此处需要远程仓库的url，登陆自己的GitHub，复制某个远程仓库的url

![github](https://pic.downk.cc/item/5f2417a714195aa594abdcea.png)

粘贴

![EPNrC.jpg](https://pic.downk.cc/item/5f2417a714195aa594abdcf3.png)

——>  OK

![EPRE4.jpg](https://pic.downk.cc/item/5f241a4314195aa594acd349.png)

——>  Push

![image](https://pic.downk.cc/item/5f241a4314195aa594acd341.png)

Git的凭证管理，输入GitHub的账号密码

![image](https://pic.downk.cc/item/5f241a4314195aa594acd34e.png)

然后IDEA上也要输入一次，然后等待push，结果push失败了

![image](https://pic.downk.cc/item/5f241a4314195aa594acd344.png)

解决方案如下：

1. 切换到自己项目所在的目录，右键选择Git Bash Here
2. 在terminal窗口中依次输入命令

```bash
git pull
git pull origin master
git pull origin master —allow-unrelated-histories
```

3. 在idea中重新push自己的项目，成功

![EPcfD.jpg](https://pic.downk.cc/item/5f241a4314195aa594acd33f.png)

登陆gitee检查

![EPb69.jpg](https://pic.downk.cc/item/5f241b1e14195aa594ad2b9c.png)

提交内容已存在与远程仓库中。到此，push完成。

# 从远程仓库克隆项目到本地（Clone）

![EPu3j.png](https://pic.downk.cc/item/5f241b1e14195aa594ad2b9a.png)

Check out from Version Control  ——>  Git

![EPEzh.jpg](https://pic.downk.cc/item/5f241b1e14195aa594ad2b98.png)

 ——> Clone
 
![EPJVN.png](https://pic.downk.cc/item/5f241b1e14195aa594ad2b9e.png)

克隆完成后会询问你是否打开项目

——>  yes

![EPI7B.png](https://pic.downk.cc/item/5f241b1e14195aa594ad2b96.png)

打开项目检查，发现与之前上传的内容一致。到此，已完成从远程仓库克隆代码到本地。

需要注意的是，由于克隆的时候是根据仓库的url进行克隆的，所以会将仓库的所有内容一并克隆。像这次克隆就将之前在eclipse用git上传的项目也克隆过来了。

![EP8lF.jpg](https://pic.downk.cc/item/5f241bd114195aa594ad7265.png)

## 从远程仓库中获取其他用户对项目的修改（pull）

可能会有人理解不了这与前者的区别，这里简单说明一下：

clone——无中生有。原来本地是没有这个项目的，因此将完整的整个项目从仓库clone到本地

pull——锦上添花。项目1.0已经在本地上存在，但其他人将项目修改成项目2.0并上传到远程仓库。因此你要做的是将远程仓库中别人做的修改部分pull到本地，让你本地的项目1.0成为项目2.0

说明过后现在开始操作，先是前期准备：

### 提交修改
首先打开commit用的项目，对其修改，使之升级为项目2.0

![EPUEM.jpg](https://pic.downk.cc/item/5f241bd114195aa594ad7263.png)

然后将代码上传到远程仓库

需要注意的是，在push前必须进行commit, 否则会显示no commits selected

![EPyfU.jpg](https://pic.downk.cc/item/5f241bd114195aa594ad726d.png)

![Commit and Push](https://pic.downk.cc/item/5f241bd114195aa594ad7261.png)

至于如何上传到远程仓库这里就不在赘述了，可以参照前文。值得提醒的是在commit的时候选择**Commit and Push**的话，就可以commit和push接连操作。

好的，现在对项目的修改已上传到远程仓库了。

![1.png](https://i0.wp.com/i.loli.net/2020/05/06/kaFWoDfZ5gnPwJS.png)

### 拉取代码

准备工作完成，现在正式进行pull：

打开刚才clone的“项目1.0”

![2.png](https://i0.wp.com/i.loli.net/2020/05/06/Mo69epAbY2ma4RX.png)

进行pull，对其更新

右击项目  ——>  Git  ——>  Repository  ——>  Pull...

![3.png](https://i0.wp.com/i.loli.net/2020/05/06/BwFjguh6c9VfzTs.png)

 ——>  Pull
 
 ![4.png](https://i0.wp.com/i.loli.net/2020/05/06/uIwdBMPWY3FVC75.png)
 
可以看出，原来的代码已作出了更新。下方显示的是更新信息。到此，pull完成。

![5.png](https://i0.wp.com/i.loli.net/2020/05/06/QwVsIJNK8XR3lt1.png)

**总结：**

1.要想用git管理项目，先要将本地项目与git关联，才能进行commit、push、pull等操作；

2.将本地项目于git关联后，本地仓库的地址默认就是项目地址；

3.从远程仓库进行项目clone后，已默认用git进行项目管理；

4.clone的时候会将仓库里的所有内容一并clone；

5.push之前需要进行commit；
