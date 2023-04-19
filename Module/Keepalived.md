[TOC]

参考：https://www.jianshu.com/p/a6b5ab36292a<br>
参考：[Nginx+keepalived 高可用双机热备(主从模式/双主模式)](https://blog.csdn.net/u012599988/article/details/82152224?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase)

高可用：两台业务系统启动着相同的服务，如果有一台故障，另一台自动接管,我们将将这个称之为高可用；

Keekpalived工作原理：通过vrrp协议实现

[![Keepalived](https://yanxuan.nosdn.127.net/9d9613e04cba0fee7ec5020e1c4d7bac.png)](https://s3.jpg.cm/2020/07/31/blrTe.png)

Keepalived工作方式：**抢占式、非抢占式**

# 1. 安装

yum install keepalived -y

日志存放位置：/var/log/messages

# 2. 配置

## 2.1 keepaliaved 抢占式配置

### 2.1.1 master配置

```bash
[root@lb01 ~] rpm -qc keepalived
/etc/keepalived/keepalived.conf
/etc/sysconfig/keepalived

[root@lb01 ~] cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id lb02 #标识信息，一个名字而已；
}
vrrp_instance VI_1 {
    state MASTER    #角色是master
    interface eth0  #vip 绑定端口
    virtual_router_id 50    #让master 和backup在同一个虚拟路由里，id 号必须相同；
    priority 150            #优先级,谁的优先级高谁就是master ;
    advert_int 1            #心跳间隔时间
    authentication {
        auth_type PASS      #认证
        auth_pass 1111      #密码 
}
    virtual_ipaddress {
        10.0.0.3            #虚拟ip
    }
}
```

### 2.2.2 backup配置

```bash
[root@lb02 ~]# cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {     
    router_id lb02   
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 100
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
}
    virtual_ipaddress {
        10.0.0.3
    }
}
```

## 2.2 keepalived非抢占式配置：

非抢占式不再有主从之分，全部都为BACKUP,并且配置文件中添加nopreempt，用来标识为非抢占式；

```bash
[root@lb01 /etc/nginx/upstream] cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id lb01
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
}
    virtual_ipaddress {
        10.0.0.3
    }
}
```


```bash
[root@lb02 /etc/nginx/upstream] cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {     
    router_id lb02   
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 100
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
}
    virtual_ipaddress {
        10.0.0.3
    }
}
```

## 2.3 nginx+keepalived

**实现思路：将keepalived 中的vip作为nginx负载均衡的监听地址，并且域名绑定的也是vip的地址。**

**说明：Nginx 负载均衡实现高可用，需要借助Keepalived地址漂移功能。**

在不考虑后端数据库和存储的时候如下架构</br>
[![Keekpalived_nginx](https://shop.io.mi-img.com/app/shop/img?id=shop_0656c57146c47654e2de8f1bf7a661f4.png)](https://s3.jpg.cm/2020/07/31/blEQy.png)

两台负载均衡配置：

MASTER

```bash
[root@lb01 /etc/nginx/upstream] ip add show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:97:e1:ff brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.5/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet 10.0.0.3/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe97:e1ff/64 scope link 
       valid_lft forever preferred_lft forever

[root@lb01 /etc/nginx/upstream] cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {
    router_id lb01
}

vrrp_instance VI_1 {
    state MASTER
    interface eth0
    virtual_router_id 50
    priority 150
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
}
    virtual_ipaddress {
        10.0.0.3
    }
}
```

BACKUP


```bash
[root@lb02 /etc/nginx/upstream] ip add show eth0
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 00:0c:29:6f:18:48 brd ff:ff:ff:ff:ff:ff
    inet 10.0.0.6/24 brd 10.0.0.255 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe6f:1848/64 scope link 
       valid_lft forever preferred_lft forever

[root@lb02 /etc/nginx/upstream] cat /etc/keepalived/keepalived.conf 
! Configuration File for keepalived
global_defs {     
    router_id lb02   
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 50
    priority 100
    nopreempt
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
}
    virtual_ipaddress {
        10.0.0.3
    }
}
```

## 2.4 keepalived脑裂现象

由于某些原因，导致两台keepalived高可用服务器在指定时间内，无法检测到对方存活心跳信息，从而导致互相抢占对方的资源和服务所有权，然而此时两台高可用服务器有都还存活。

可能出现的原因：

1. 服务器网线松动等网络故障；
2. 服务器硬件故障发生损坏现象而崩溃；
3. 主备都开启了firewalld 防火墙。
4. 在Keepalived+nginx 架构中，当Nginx宕机，会导致用户请求失败，但是keepalived不会进行切换，所以需要编写一个检测nginx的存活状态的脚本，如果nginx不存活，则kill掉宕掉的nginx主机上面的keepalived。(所有的keepalived都要配置)

架构如下：

[![keepalived脑裂现象](https://ae01.alicdn.com/kf/U7ca8a6116e0f4869a637a728e365c287d.jpg)](https://s.pc.qq.com/tousu/img/20200821/6198113_1597974546.jpg)

**脚本如下：**


```bash
[root@lb01 /server/scripts] cat /server/scripts/check_list

#!/bin/sh
nginxpid=$(ps -C nginx --no-header|wc -l)
#1.判断Nginx是否存活,如果不存活则尝试启动Nginx
if [ $nginxpid -eq 0 ];then
    systemctl start nginx
    sleep 3
    #2.等待3秒后再次获取一次Nginx状态
    nginxpid=$(ps -C nginx --no-header|wc -l) 
    #3.再次进行判断, 如Nginx还不存活则停止Keepalived,让地址进行漂移,并退出脚本  
    if [ $nginxpid -eq 0 ];then
        systemctl stop keepalived
   fi
fi
```

keepalived.conf配置文件

```bash
[root@lb01 /server/scripts] cat /etc/keepalived/keepalived.conf
! Configuration File for keepalived
global_defs {         #全局变量
    router_id lb01
}

vrrp_script check {
    script "/server/scripts/check_list"
    interval  10  #脚本执行间隔,每10s检测一次
    weight -5     #脚本结果导致的优先级变更,检测失败(脚本返回非0)则优先级-5
    fall 2        #检测连续2次失败才算确定是真失败。会用weight减少优先级（1-255之间）
    rise 1        #检测1次成功就算成功。但不修改优先级
}

vrrp_instance VI_1 {  #keepalive或者vrrp的一个实例
    state MASTER      #状态
    interface eth0    #通信端口
    virtual_router_id 50 #虚拟路由标识，这个标识是一个数字。同一vrrp_instance下，MASTER和BACKUP必须是一致的
    priority 150      #数字越大，优先级越高，在同一个vrrp_instance下，MASTER的优先级必须大于BACKUP的优先级
    advert_int 1      #MASTER与BACKUP心跳的间隔,秒
    authentication {  #设置验证类型和密码。主从必须一样
        auth_type PASS #设置vrrp验证类型，主要有PASS和AH两种
        auth_pass 1111
}
    virtual_ipaddress {  #执行监控的服务。注意这个设置不能紧挨着写在vrrp_script配置块的后面
        10.0.0.3   #VRRP HA 虚拟地址 如果有多个VIP，继续换行填写
    }
    track_script  {
    check   #调用检测脚本
}
}
```