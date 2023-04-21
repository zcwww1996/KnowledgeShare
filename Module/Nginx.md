[TOC]
# 1. Nginx相关概念
## 1.1 反向代理
**正向代理**

在如今的网络环境下，我们如果由于技术需要要去访问国外的某些网站，此时你会发现位于国外的某网站我们通过浏览器是没有办法访问的，此时大家可能都会用一个操作FQ进行访问，FQ的方式主要是找到一个可以访问国外网站的代理服务器，我们将请求发送给代理服务器，代理服务器去访问国外的网站，然后将访问到的数据传递给我们！

上述这样的代理模式称为正向代理，正向代理最大的特点是客户端非常明确要访问的服务器地址；服务器只清楚请求来自哪个代理服务器，而不清楚来自哪个具体的客户端；正向代理模式屏蔽或者隐藏了真实客户端信息。

[![正向代理](https://cdn7.232232.xyz/58/2022/09/05-63155b9cc155c.jpg)](https://c.mipcdn.com/i/s/cdn.gksec.com/2020/07/31/b8aa647a9d84b/zxdl.jpg)

**反向代理**

反向代理（Reverse Proxy）方式是指以代理服务器来接受internet上的连接请求，然后将请求转发给内部网络上的服务器，并将从服务器上得到的结果返回给internet上请求连接的客户端，此时代理服务器对外就表现为一个服务器。

![反向代理](https://www.z4a.net/images/2020/07/14/fanxiamgdaili.png)

反向代理，主要用于服务器集群分布式部署的情况下，反向代理隐藏了服务器的信息！

[![反向代理](https://ae01.alicdn.com/kf/H0cba9d33ff40432f9dad86c232ec4f29V.jpg)](https://ftp.bmp.ovh/imgs/2020/07/ccd65a9e14917cad.png)

## 1.2 负载均衡
负载均衡，英文名称为Load Balance，是指建立在现有网络结构之上，并提供了一种廉价有效透明的方法扩展网络设备和服务器的带宽、增加吞吐量、加强网络数据处理能力、提高网络的灵活性和可用性。其原理就是数据流量分摊到多个服务器上执行，减轻每台服务器的压力，多台服务器共同完成工作任务，从而提高了数据的吞吐量。

[![负载均衡](https://pic1.imgdb.cn/item/64423e840d2dde57774cca1a.png)](https://ftp.bmp.ovh/imgs/2020/07/e98d8827a3710e8e.png)
# 2. Nginx的安装
## 2.1 下载nginx
官网：http://nginx.org/</br>
ftp服务器：http://nginx.org/download/
## 2.2 上传并解压nginx
tar -zxvf nginx-1.8.1.tar.gz -C /usr/local/src
## 2.3 编译nginx
1. 进入到nginx源码目录

```bash
cd /usr/local/src nginx-1.8.1
```

2. 检查安装环境

```bash
./configure --prefix=/usr/local/nginx
```

3. <font color="red">缺包报错 `./configure: error: C compiler cc is not found`</font>

使用YUM安装缺少的包

```bash
yum -y install zlib zlib-devel openssl openssl-devel pcre pcre-devel openssl openssl-devel gcc
```

4. 编译安装

```bash
make && make install
```

在nginx.conf中增加以下内容

```
server {
        listen       6666;    #  修改要监听的端口
        server_name  localhost;
        location / {
            root   /data/pack/;   # 这里请换成你的实际目录绝对路径
			autoindex  on;        # 打开目录浏览
			autoindex_exact_size off; # 关闭详细文件大小统计，让文件大小显示MB，GB单位，默认为b；
			autoindex_localtime on;   # 开启以服务器本地时区显示文件修改日期！
            index  index.html index.htm;
        }
    }
```

> **这里开启的是全局的目录浏览功能，那么如何实现限定部分目录浏览功能呢？**
> 
> 只打开localhost:6666/soft 目录浏览功能
> 
> ```
> vi  /usr/local/nginx/conf/nginx.conf，在server {下面添加以下内容：
> 
> location   /soft {
>
> root   /data/pack/;
>
> autoindex on;
> 
> autoindex_exact_size off;
> 
> autoindex_localtime on;
> 
> }
> ```

5. 启动

```bash
测试Nginx：
sbin/nginx -t
 
启动Nginx：
sbin/nginx -c /usr/local/nginx/conf/nginx.conf

启动后，动态更新配置文件
sbin/nginx -s reload
```

[基于nginx搭建yum源服务器](https://www.cnblogs.com/omgasw/p/10194698.html)