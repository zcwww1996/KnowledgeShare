[TOC]
# 登陆
登录到管理员账户: 如果在本机上，可以通过kadmin.local直接登录。其它机器的，先使用kinit进行验证。
## 验证：
```shell
[root@localhost ~]# kinit root/admin@EXAMPLE.COM
Password for root/admin@EXAMPLE.COM: 123456
```
## 登录：
```shell
[root@localhost ~]# kadmin
Authenticating as principal root/admin@EXAMPLE.COM with password.
Password for root/admin@EXAMPLE.COM:
```
## 列出所有帐户

`listprincs`或者`list_principals`

方法一：maste KDC上执行kadmin.local，进入kerberos数据库:

```bash
list_principals
```

方法二：maste KDC上执行:

```bash
kadmin.local -q "list_principals"
```

```shell
K/M@EXAMPLE.COM
admin/admin@EXAMPLE.COM
kadmin/admin@EXAMPLE.COM
kadmin/changepw@EXAMPLE.COM
kadmin/localhost@EXAMPLE.COM
kiprop/localhost@EXAMPLE.COM
krbtgt/EXAMPLE.COM@EXAMPLE.COM
root/admin@EXAMPLE.COM
```

## 添加用户
生成随机key的principal，这种principal只能用来生成keytab使用<br>
`addprinc -randkey people@FEILONG.COM`

生成指定密码的principal，执行后会提示输入密码<br>
`addprinc admin/admin`
## 生成实例文件

ktadd, xst效果一样

 `-norandkey`表示不用很长的随机密码，使用自己设置的密码，意味着可以同时用密码和keytab认证<br>
`xst -k /etc/security/keytabs/people_people.keytab people@FEILONG.COM`

为principal生成keytab，可同时添加多个<br>
`ktadd -norandkey -k /keytab/root.keytab root/master1@EXAMPLE.COM host/master1@EXAMPLE.COM`

## 删除用户
`delete_principal people@FEILONG.COM`

# 编辑keytab
## 查看当前的认证用户：
`klist -e`等效于klist

```shell
[root@localhost ~]# klist
Ticket cache: KEYRING:persistent:0:0
Default principal: root/admin@EXAMPLE.COM
Valid starting Expires Service principal
2017-08-23T14:01:32 2017-08-24T14:01:32 krbtgt/EXAMPLE.COM@EXAMPLE.COM
renew until 2017-08-30T14:01:32
```

## 显示hdfs.keytab 文件列表
```shell
[root@hadoop02 keytabs]# klist -ket hdfs.headless.keytab
Keytab name: FILE:hdfs.headless.keytab
KVNO Timestamp           Principal
---- ------------------- ------------------------------------------------------
   1 2019-08-09T11:53:13 hdfs-caac@FEILONG.COM (aes256-cts-hmac-sha1-96) 
   1 2019-08-09T11:53:13 hdfs-caac@FEILONG.COM (des3-cbc-sha1) 
   1 2019-08-09T11:53:13 hdfs-caac@FEILONG.COM (arcfour-hmac) 
   1 2019-08-09T11:53:13 hdfs-caac@FEILONG.COM (aes128-cts-hmac-sha1-96) 
   1 2019-08-09T11:53:13 hdfs-caac@FEILONG.COM (des-cbc-md5)
```

## 合并keytab文件
ktutil
```shell
ktutil: rkt hdfs-unmerged.keytab
ktutil: rkt HTTP.keytab
ktutil: wkt hdfs.keytab
```
## kdc认证
```shell
kinit -kt /etc/security/keytabs/bigdata_audit_bigdata_audit.keytab bigdata_audit@FEILONG.COM
```

## 销毁认证

```shell
kdestroy
```

# kerberos命令列表

命令使用帮助中我们可以查询到哪些kerberos命令是一样的，用逗号分隔的命令就是相等的
```bash
[root@dounine ~]# kadmin.local
Authenticating as principal root/admin@EXAMPLE.COM with password.
kadmin.local:  ? #是查看帮助命令
Available kadmin.local requests:

add_principal, addprinc, ank
                         Add principal
delete_principal, delprinc
                         Delete principal
modify_principal, modprinc
                         Modify principal
rename_principal, renprinc
                         Rename principal
change_password, cpw     Change password
get_principal, getprinc  Get principal
list_principals, listprincs, get_principals, getprincs
                         List principals
add_policy, addpol       Add policy
modify_policy, modpol    Modify policy
delete_policy, delpol    Delete policy
get_policy, getpol       Get policy
list_policies, listpols, get_policies, getpols
                         List policies
get_privs, getprivs      Get privileges
ktadd, xst               Add entry(s) to a keytab
ktremove, ktrem          Remove entry(s) from a keytab
lock                     Lock database exclusively (use with extreme caution!)
unlock                   Release exclusive database lock
purgekeys                Purge previously retained old keys from a principal
get_strings, getstrs     Show string attributes on a principal
set_string, setstr       Set a string attribute on a principal
del_string, delstr       Delete a string attribute on a principal
list_requests, lr, ?     List available requests.
quit, exit, q            Exit program. 

```
