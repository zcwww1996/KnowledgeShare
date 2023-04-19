[TOC]
NetSarang Xshell – 知名终端连接工具，非常强大的SSH远程终端客户端 ，非常好用的SSH终端管理器。Xshell功能超级强大，性能非常优秀，其特色功能支持多标签会话管理主机，支持远程协议Telnet、Rlogin、SSH/SSH PKCS＃11、SFTP、Serial，其它功能包括动态端口转发、自定义键盘映射、VB脚本支持、完全的 Unicode 支持等。



微夏博客在网上找了找相关的软件，发现都被和谐了。很明显，被某公司代理了。

XSHELL真正的官方网站：https://www.netsarang.com/products/xsh_overview.html

[![xshell6_官网](https://pic.downk.cc/item/5f23cde714195aa5948d2760.png)](https://s2.ax1x.com/2019/12/31/l3i9zt.png)

选择下载试用版，收到下载地址后将下载地址后加上r，例如https://download.netsarang.com/298e2321/XmanagerPowerSuite-6.0.0006.exe，改为https://download.netsarang.com/298e2321/XmanagerPowerSuite-6.0.0006r.exe，如果下载不带r的版本，无法输入序列号

**版本号为：XshellPlus-6.0.0005r.exe**


```bash
注册码

Xshell 5

181226-111351-999033

Xftp 5

181227-114744-999004

Xshell 6

181226-111725-999177

Xftp 6

181227-114016-999644

XshellPlus 6

181226-117860-999055
180505-117501-020791

XmanagerPowerSuite 6

181226-116119-999510
180429-116253-999126
```

下面来看下软件的安装激活图解教程：
# 1. 安装步骤
1、正常安装软件，中间需要填写序列号。

[![xshell6_序列号](https://pic.downk.cc/item/5f23cde714195aa5948d276f.png)](https://s2.ax1x.com/2019/12/31/l3ieij.png)

2、正常安装软件至结束。

[![xshell6_安装完成](https://pic.downk.cc/item/5f23cde714195aa5948d276a.png)](https://s2.ax1x.com/2019/12/31/l3iEdg.png)

3、正常打开软件，你会发现，虽然已经填了序列号，但是打开还是提示需要激活，并且无法激活成功。

[![xshell6_激活失败](https://pic.downk.cc/item/5f23cfec14195aa5948e595e.png)](https://s2.ax1x.com/2019/12/31/l3iSJA.png)

# 2. 解决方法

1、打开注册表程序（开始 -> 运行 -> regedit)

[![xshell6_打开注册表](https://pic.downk.cc/item/5f23cfec14195aa5948e5969.png)](https://s2.ax1x.com/2019/12/31/l3Pzid.png)

2、找到注册表路径 --> 右键 -->权限

Xshell修改注册表路径：**HKEY_CURRENT_USER\Software\NetSarang\Xshell\6\LiveUpdate**

Xftp修改注册表路径：**HKEY_CURRENT_USER\Software\NetSarang\Xftp\6\LiveUpdate**

3、点击 “高级”

[![xshell6_注册表_高级](https://pic.downk.cc/item/5f23cde714195aa5948d2774.png)](https://s2.ax1x.com/2019/12/31/l3iFL8.png)

4、点击高级后，窗口栏处如果没有编辑选项，点击 禁止继承 --> 将已继承的权限转换为此对象的显示权限，然后点击确定

（如果显示编辑选项可直接略过此步骤）

[![xshell6_禁止继承](https://pic.downk.cc/item/5f23cfec14195aa5948e595c.png)](https://s2.ax1x.com/2019/12/31/l3iPQP.png)

5、再次回到窗口后，可以看到已经显示了编辑选项，把**当前用户**权限全部都去掉，然后点确定保存即可。

[![xshell6_去掉权限](https://pic.downk.cc/item/5f23cfec14195aa5948e5966.png)](https://s2.ax1x.com/2019/12/31/l3PvIH.png)

6、再次打开软件，会显示已激活软件。

[![xshell6_激活成功验证](https://pic.downk.cc/item/5f23cfec14195aa5948e5963.png)](https://s2.ax1x.com/2019/12/31/l3iVoQ.png)

7、Xftp 同样这样操作，把权限去掉即可。

[![xshell6_xftp修改](https://pic.downk.cc/item/5f23cde714195aa5948d2767.png)](https://s2.ax1x.com/2019/12/31/l3iAeS.png)