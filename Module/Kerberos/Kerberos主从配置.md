[TOC]

参考：https://www.cnblogs.com/gentlemanhai/p/7804205.html

# 1. 服务端单节点部署
## 1.1 安装kdc server包
### 1.1.1 安装kdc server服务

```bash
yum install krb5-server krb5-libs krb5-devel krb5-workstation -y
```

### 1.1.2 检查安装环境

```bash
[root@ecs-hdp-1 ~]# rpm -qa|grep krb5

krb5-devel-1.15.1-46.el7.x86_64
krb5-server-1.15.1-46.el7.x86_64
krb5-workstation-1.15.1-46.el7.x86_64
krb5-libs-1.15.1-46.el7.x86_64
```

### 1.1.3 修改配置文件
配置文件第一台修改后，发送到第二台
涉及三个文件:
- /etc/krb5.conf
- /var/kerberos/krb5kdc/kdc.conf
- /var/kerberos/krb5kdc/kadm5.acl

1) /etc/krb5.conf
```bash
[logging]
  default = FILE:/var/log/krb5libs.log
  kdc = FILE:/var/log/krb5kdc.log
admin_server = FILE:/var/log/kadmind.log

[libdefaults]
  renew_lifetime = 7d
  forwardable = true
  default_realm = OSS3.COM
  ticket_lifetime = 24h
  dns_lookup_realm = false
  dns_lookup_kdc = false
  default_ccache_name = /tmp/krb5cc_%{uid}
  #default_tgs_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5
  #default_tkt_enctypes = aes des3-cbc-sha1 rc4 des-cbc-md5

[realms]
  OSS3.COM = {
    admin_server = ecs-hdp-1
    kdc = ecs-hdp-1
    kdc = ecs-hdp-2
  }
```

2) /var/kerberos/krb5kdc/kdc.conf

```bash
[kdcdefaults]
 kdc_ports = 88
 kdc_tcp_ports = 88

[realms]
 OSS3.COM = {
  master_key_type = aes256-cts
  max_life = 72h
  max_renewable_life = 7d
  acl_file = /var/kerberos/krb5kdc/kadm5.acl
  dict_file = /usr/share/dict/words
  admin_keytab = /var/kerberos/krb5kdc/kadm5.keytab
  supported_enctypes = aes256-cts:normal aes128-cts:normal des3-hmac-sha1:normal arcfour-hmac:normal camellia256-cts:normal camellia128-cts:normal des-hmac-sha1:normal des-cbc-md5:normal des-cbc-crc:normal
 }

```

3) /var/kerberos/krb5kdc/kadm5.acl

```bash
*/admin@OSS3.COM     *
```

## 1.2 服务端创建库
在kdc server服务器执行以下命令：<br>
`kdb5_util create -r OSS3.COM -s`

创建 kerberos 管理用户:

```bash
kadmin.local
addprinc root/admin
输入密码(123456)
```

然后执行 exit

## 1.3 启动服务
依次执行以下命令

```bash
systemctl enable krb5kdc
systemctl enable kadmin
systemctl start krb5kdc
systemctl start kadmin
```

验证：<br>
`kinit root/admin`

## 1.4 创建host keytab文件
在master kdc，执行

```bash
进入kdc:
kadmin.local

执行：
addprinc -randkey host/ecs-hdp-1
ktadd host/ecs-hdp-1
addprinc -randkey host/ecs-hdp-2
ktadd host/ecs-hdp-2
```

## 1.5 salve kdc配置

### 1.5.1 重复master操作
执行上述1.1 - 1.2 的步骤，其中配置文件从master拷贝，且权限为root用户：
1. 安装kdc server服务
2. 检查安装环境
3. 接收配置文件
4. 服务端创建库

### 1.5.2 创建kpropd.acl文件

```bash
vim /var/kerberos/krb5kdc/kpropd.acl
host/ecs-hdp-1@OSS3.COM
host/ecs-hdp-2@OSS3.COM
```

在slave上启动kpropd服务<br>
`systemctl start kprop`

查看kprop状态<br>
`systemctl status kprop`

## 1.6 数据同步
在master(ecs-hdp-1)上将相关数据同步到slave上(需定期手动同步)

创建备份<br>
`kdb5_util dump /var/kerberos/krb5kdc/master.dump`

master发送数据到slave<br>
`kprop -f /var/kerberos/krb5kdc/master.dump -d ecs-hdp-2`

如果同步成功，显示

```bash
82870 bytes sent.
Database propagation to ecs-hdp-2: SUCCEEDED
```

slave上/var/kerberos/krb5kdc/会多出一些文件，如:

from_master、principal、pricipal.kadm5、principal.kadmin5.lock、principal.ok

## 1.7 定时同步

### 1.7.1 同步脚本
**cd /var/kerberos/krb5kdc**<br>
**vim kprop_sync.sh**<br>

```bash
#!/bin/bash 
DUMP_FILE=/var/kerberos/krb5kdc/master.dump
PORT=754 
kdclist='ecs-hdp-2' 
TIMESTAMP=`date` 
echo "==========Start at $TIMESTAMP" 
echo "dump"
/usr/sbin/kdb5_util dump $DUMP_FILE 
for kdc in $kdclist 
do
  echo "kprop to $kdc" 
  /usr/sbin/kprop -f $DUMP_FILE -d -P $PORT $kdc 
done
```

### 1.7.2 创建定时任务

**crontab -e**

```bash
0 * * * * sh /var/kerberos/krb5kdc/kprop_sync.sh >/var/kerberos/krb5kdc/kprop_sync.log 2>&1
```

## 1.8 slave启动进程
在slave节点，启动krb

```bash
systemctl enable krb5kdc
systemctl enable kadmin
systemctl start krb5kdc
systemctl start kadmin
```

