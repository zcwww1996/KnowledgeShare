[TOC]

参考：<br>
https://www.cnblogs.com/praybb/p/7095580.html
https://blog.51cto.com/12790274/2062955
http://blog.chinaunix.net/uid-23886490-id-3565908.html
https://blog.csdn.net/weixin_30832983/article/details/98035557

Centos7.2环境安装(安装桌面)

# 基础环境检查
## 查看版本
cat /etc/redhat-release
## 查看连接
ifconfig eth0
echo "127.0.0.1      testpray" >> /etc/hosts
## 测试IP连接
ping  testpray
## 停用CentOS的防火墙
systemctl stop firewalld
systemctl disable firewalld

# 安装依赖包和相关设置

## 安装包

```bash
yum -y install binutils compat-libcap1 compat-libstdc++-33 compat-libstdc++-33*.i686 elfutils-libelf-devel gcc gcc-c++ glibc*.i686 glibc glibc-devel glibc-devel*.i686 ksh libgcc*.i686 libgcc libstdc++ libstdc++*.i686 libstdc++-devel libstdc++-devel*.i686 libaio libaio*.i686 libaio-devel libaio-devel*.i686 make sysstat unixODBC unixODBC*.i686 unixODBC-devel unixODBC-devel*.i686 libXp
```


```bash
mkdir -p /Oracle
chmod -R 775 /Oracle
mkdir -p /OracleDatafile
chmod -R 775 /OracleDatafile
chown -R oracle:oinstall /OracleDatafile
mkdir -p /software
chmod -R 775 /software
```

## 安装下载工具WGET
`yum install wget`

更新系统,这里我们用国内的163的源来更新，速度比较快

```bash
mv /etc/yum.repos.d/CentOS-Base.repo.bak /etc/yum.repos.d/CentOS-Base.repo
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.163.com/.help/CentOS7-Base-163.repo
```

`yum clean all`

`yum makecache`

## 更新系统
`yum update -y`

## 创建用户组与用户

```bash
groupadd oinstall
groupadd dba
groupadd oper
useradd -g oinstall -G dba,oper oracle
echo "123456" | passwd --stdin oracle
```

## 安装目录

```bash
mkdir -p /Oracle/app/oracle/product/12.1.0/dbhome_1
chown -R oracle:oinstall /Oracle/app
chmod -R 775 /Oracle/app
```

## 修改内核参数

```bash
### 先备份
cp /etc/sysctl.d/99-sysctl.conf /etc/sysctl.d/99-sysctl.conf.bak
### 添加以下参数
echo "fs.aio-max-nr = 1048576" >>/etc/sysctl.d/99-sysctl.conf
echo fs.file-max = 6815744 >>/etc/sysctl.d/99-sysctl.conf
# shmall=Totalmem*40%
echo  kernel.shmall = 469203352 >>/etc/sysctl.d/99-sysctl.conf
# shmmax=Totalmem*50%
echo  kernel.shmmax = 586504192 >>/etc/sysctl.d/99-sysctl.conf
echo  kernel.shmmni = 4096 >>/etc/sysctl.d/99-sysctl.conf
echo  kernel.sem = 250 32000 100 128 >>/etc/sysctl.d/99-sysctl.conf
echo net.ipv4.ip_local_port_range = 9000 65500 >>/etc/sysctl.d/99-sysctl.conf
echo net.core.rmem_default = 262144 >>/etc/sysctl.d/99-sysctl.conf
echo net.core.rmem_max = 4194304 >>/etc/sysctl.d/99-sysctl.conf
echo net.core.wmem_default = 262144 >>/etc/sysctl.d/99-sysctl.conf
echo net.core.wmem_max = 1048586 >>/etc/sysctl.d/99-sysctl.conf
#使之生效
sysctl -p
```

## 改文件限制/etc/security/limits.d/20-nproc.conf

```bash
echo oracle soft nproc 2047 >>/etc/security/limits.d/20-nproc.conf
echo oracle hard nproc 16384 >>/etc/security/limits.d/20-nproc.conf
echo oracle soft nofile 1024 >>/etc/security/limits.d/20-nproc.conf
echo oracle hard nofile 65536 >>/etc/security/limits.d/20-nproc.conf
echo oracle soft stack 10240 >>/etc/security/limits.d/20-nproc.conf
echo oracle hard stack 32768 >>/etc/security/limits.d/20-nproc.conf
```

## 修改/etc/pam.d/login

```bash
echo session required /lib/security/pam_limits.so >>/etc/pam.d/login
echo session required pam_limits.so >>/etc/pam.d/login
#手动修改ulimit /etc/profile
echo 'if [ $USER = "oracle" ]; then' >>/etc/profile
echo 'if [ $SHELL = "/bin/ksh" ]; then' >>/etc/profile
echo 'ulimit -p 16384' >>/etc/profile
echo 'ulimit -n 65536a' >>/etc/profile
echo 'else' >>/etc/profile
echo 'ulimit -u 16384 -n 65536' >>/etc/profile
echo 'fi' >>/etc/profile
echo 'fi' >>/etc/profile
```

# 换用户安装

## 换用户
`su oracle`
## 修改 ~/.bash_profile

```bash
echo 'ORACLE_BASE=/Oracle/app/oracle'>>~/.bash_profile
echo 'ORACLE_HOME=$ORACLE_BASE/product/12.1.0/dbhome_1'>>~/.bash_profile
echo 'ORACLE_SID=orcl'>>~/.bash_profile
echo 'export ORACLE_BASE ORACLE_HOME ORACLE_SID'>>~/.bash_profile
echo 'PATH=$ORACLE_HOME/bin:$PATH'>>~/.bash_profile
echo 'export PATH'>>~/.bash_profile
```

## 上传JAVA环境安装文件至/software并安装
`rpm -ivh /software/jdk-8u131-linux-x64.rpm`
## 上传Oracle安装文件至/software

```bash
yum install zip unzip
unzip /software/linuxamd64_12102_database_1of2.zip -d /software
unzip /software/linuxamd64_12102_database_2of2.zip -d /software
```

## 解决C12乱码问题
### 创建TTF目录
`mkdir -p /software/update/jdk/jre/lib/fonts/fallback/`
### 上传zysong.ttf 到/software/update/jdk/jre/lib/fonts/fallback/

### 更新安装包的JAR包
```bash
cd /software/update/
jar -uf /software/database/stage/Components/oracle.jdk/1.6.0.75.0/1/DataFiles/filegroup2.jar jdk/jre/lib/fonts/fallback/zysong.ttf
```

# 安装界面安装Oracle
- `cd /software/database`
- `./runInstaller`
- (如须静态安装请见附件)安装界面>安全更新跳过>安装选项-创建和配置>系统类型-服务器>网格安装-单实例>安装类型-高级>语言-简体\英语>
- 版本-企业>安装目录默认(因为我们环境里已设置了)>清单-默认>配置类型-一般用途>数据库标识-默认>内存-默认>
- 数据库存储-目录:`/OracleDatafile/app/oracle/oracledata/>`

```bash
sh /Oracle/app/oraInventory/orainstRoot.sh
sh /Oracle/app/oracle/product/12.1.0/dbhome_1/root.sh
sqlplus "/as sysdba"
sql>startup
```
lsnrctl status 可以看到监听的状态

## 配置文件配置
```properties
####################################################################
## Copyright(c) Oracle Corporation 1998,2014. All rights reserved.##
##                                                                ##
## Specify values for the variables listed below to customize     ##
## your installation.                                             ##
##                                                                ##
## Each variable is associated with a comment. The comment        ##
## can help to populate the variables with the appropriate        ##
## values.                                                        ##
##                                                                ##
## IMPORTANT NOTE: This file contains plain text passwords and    ##
## should be secured to have read permission only by oracle user  ##
## or db administrator who owns this installation.                ##
##                                                                ##
####################################################################


#-------------------------------------------------------------------------------
# Do not change the following system generated value.
#-------------------------------------------------------------------------------
oracle.install.responseFileVersion=/oracle/install/rspfmt_dbinstall_response_schema_v12.1.0

#-------------------------------------------------------------------------------
# Specify the installation option.
# It can be one of the following:
#   - INSTALL_DB_SWONLY
#   - INSTALL_DB_AND_CONFIG
#   - UPGRADE_DB
#-------------------------------------------------------------------------------
oracle.install.option=INSTALL_DB_AND_CONFIG

#-------------------------------------------------------------------------------
# Specify the hostname of the system as set during the install. It can be used
# to force the installation to use an alternative hostname rather than using the
# first hostname found on the system. (e.g., for systems with multiple hostnames
# and network interfaces)
#-------------------------------------------------------------------------------
ORACLE_HOSTNAME=testpray

#-------------------------------------------------------------------------------
# Specify the Unix group to be set for the inventory directory. 
#-------------------------------------------------------------------------------
UNIX_GROUP_NAME=oinstall

#-------------------------------------------------------------------------------
# Specify the location which holds the inventory files.
# This is an optional parameter if installing on
# Windows based Operating System.
#-------------------------------------------------------------------------------
INVENTORY_LOCATION=/Oracle/app/oraInventory
#-------------------------------------------------------------------------------
# Specify the languages in which the components will be installed.            
#
# en   : English                  ja   : Japanese                 
# fr   : French                   ko   : Korean                   
# ar   : Arabic                   es   : Latin American Spanish   
# bn   : Bengali                  lv   : Latvian                  
# pt_BR: Brazilian Portuguese     lt   : Lithuanian               
# bg   : Bulgarian                ms   : Malay                    
# fr_CA: Canadian French          es_MX: Mexican Spanish          
# ca   : Catalan                  no   : Norwegian                
# hr   : Croatian                 pl   : Polish                   
# cs   : Czech                    pt   : Portuguese               
# da   : Danish                   ro   : Romanian                 
# nl   : Dutch                    ru   : Russian                  
# ar_EG: Egyptian                 zh_CN: Simplified Chinese       
# en_GB: English (Great Britain)  sk   : Slovak                   
# et   : Estonian                 sl   : Slovenian                
# fi   : Finnish                  es_ES: Spanish                  
# de   : German                   sv   : Swedish                  
# el   : Greek                    th   : Thai                     
# iw   : Hebrew                   zh_TW: Traditional Chinese      
# hu   : Hungarian                tr   : Turkish                  
# is   : Icelandic                uk   : Ukrainian                
# in   : Indonesian               vi   : Vietnamese               
# it   : Italian                                                  
#
# all_langs   : All languages
#
# Specify value as the following to select any of the languages.
# Example : SELECTED_LANGUAGES=en,fr,ja
#
# Specify value as the following to select all the languages.
# Example : SELECTED_LANGUAGES=all_langs 
#-------------------------------------------------------------------------------
SELECTED_LANGUAGES=zh_CN,en

#-------------------------------------------------------------------------------
# Specify the complete path of the Oracle Home.
#-------------------------------------------------------------------------------
ORACLE_HOME=/Oracle/app/oracle/product/12.1.0/dbhome_1

#-------------------------------------------------------------------------------
# Specify the complete path of the Oracle Base.
#-------------------------------------------------------------------------------
ORACLE_BASE=/Oracle/app/oracle

#-------------------------------------------------------------------------------
# Specify the installation edition of the component.                    
#                                                            
# The value should contain only one of these choices. 
     
#   - EE     : Enterprise Edition

#-------------------------------------------------------------------------------
oracle.install.db.InstallEdition=EE

###############################################################################
#                                                                             #
# PRIVILEGED OPERATING SYSTEM GROUPS                                          #
# ------------------------------------------                                  #
# Provide values for the OS groups to which OSDBA and OSOPER privileges       #
# needs to be granted. If the install is being performed as a member of the   #
# group "dba", then that will be used unless specified otherwise below.       #
#                                                                             #
# The value to be specified for OSDBA and OSOPER group is only for UNIX based #
# Operating System.                                                           #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# The DBA_GROUP is the OS group which is to be granted OSDBA privileges.
#-------------------------------------------------------------------------------
oracle.install.db.DBA_GROUP=dba

#------------------------------------------------------------------------------
# The OPER_GROUP is the OS group which is to be granted OSOPER privileges.
# The value to be specified for OSOPER group is optional.
#------------------------------------------------------------------------------
oracle.install.db.OPER_GROUP=dba

#------------------------------------------------------------------------------
# The BACKUPDBA_GROUP is the OS group which is to be granted OSBACKUPDBA privileges.
#------------------------------------------------------------------------------
oracle.install.db.BACKUPDBA_GROUP=dba

#------------------------------------------------------------------------------
# The DGDBA_GROUP is the OS group which is to be granted OSDGDBA privileges.
#------------------------------------------------------------------------------
oracle.install.db.DGDBA_GROUP=dba

#------------------------------------------------------------------------------
# The KMDBA_GROUP is the OS group which is to be granted OSKMDBA privileges.
#------------------------------------------------------------------------------
oracle.install.db.KMDBA_GROUP=dba

###############################################################################
#                                                                             #
#                               Grid Options                                  #
#                                                                             #
###############################################################################
#------------------------------------------------------------------------------
# Specify the type of Real Application Cluster Database
#
#   - ADMIN_MANAGED: Admin-Managed
#   - POLICY_MANAGED: Policy-Managed
#
# If left unspecified, default will be ADMIN_MANAGED
#------------------------------------------------------------------------------
oracle.install.db.rac.configurationType=

#------------------------------------------------------------------------------
# Value is required only if RAC database type is ADMIN_MANAGED
#
# Specify the cluster node names selected during the installation.
# Leaving it blank will result in install on local server only (Single Instance)
#
# Example : oracle.install.db.CLUSTER_NODES=node1,node2
#------------------------------------------------------------------------------
oracle.install.db.CLUSTER_NODES=

#------------------------------------------------------------------------------
# This variable is used to enable or disable RAC One Node install.
#
#   - true  : Value of RAC One Node service name is used.
#   - false : Value of RAC One Node service name is not used.
#
# If left blank, it will be assumed to be false.
#------------------------------------------------------------------------------
oracle.install.db.isRACOneInstall=false

#------------------------------------------------------------------------------
# Value is required only if oracle.install.db.isRACOneInstall is true.
#
# Specify the name for RAC One Node Service
#------------------------------------------------------------------------------
oracle.install.db.racOneServiceName=

#------------------------------------------------------------------------------
# Value is required only if RAC database type is POLICY_MANAGED
#
# Specify a name for the new Server pool that will be configured
# Example : oracle.install.db.rac.serverpoolName=pool1
#------------------------------------------------------------------------------
oracle.install.db.rac.serverpoolName=

#------------------------------------------------------------------------------
# Value is required only if RAC database type is POLICY_MANAGED
#
# Specify a number as cardinality for the new Server pool that will be configured
# Example : oracle.install.db.rac.serverpoolCardinality=2
#------------------------------------------------------------------------------
oracle.install.db.rac.serverpoolCardinality=0

###############################################################################
#                                                                             #
#                        Database Configuration Options                       #
#                                                                             #
###############################################################################

#-------------------------------------------------------------------------------
# Specify the type of database to create.
# It can be one of the following:
#   - GENERAL_PURPOSE                      
#   - DATA_WAREHOUSE
# GENERAL_PURPOSE: A starter database designed for general purpose use or transaction-heavy applications.
# DATA_WAREHOUSE : A starter database optimized for data warehousing applications.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.type=GENERAL_PURPOSE

#-------------------------------------------------------------------------------
# Specify the Starter Database Global Database Name.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.globalDBName=orcl

#-------------------------------------------------------------------------------
# Specify the Starter Database SID.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.SID=orcl

#-------------------------------------------------------------------------------
# Specify whether the database should be configured as a Container database.
# The value can be either "true" or "false". If left blank it will be assumed
# to be "false".
#-------------------------------------------------------------------------------
oracle.install.db.ConfigureAsContainerDB=true

#-------------------------------------------------------------------------------
# Specify the  Pluggable Database name for the pluggable database in Container Database.
#-------------------------------------------------------------------------------
oracle.install.db.config.PDBName=pdborcl

#-------------------------------------------------------------------------------
# Specify the Starter Database character set.
#                                              
#  One of the following
#  AL32UTF8, WE8ISO8859P15, WE8MSWIN1252, EE8ISO8859P2,
#  EE8MSWIN1250, NE8ISO8859P10, NEE8ISO8859P4, BLT8MSWIN1257,
#  BLT8ISO8859P13, CL8ISO8859P5, CL8MSWIN1251, AR8ISO8859P6,
#  AR8MSWIN1256, EL8ISO8859P7, EL8MSWIN1253, IW8ISO8859P8,
#  IW8MSWIN1255, JA16EUC, JA16EUCTILDE, JA16SJIS, JA16SJISTILDE,
#  KO16MSWIN949, ZHS16GBK, TH8TISASCII, ZHT32EUC, ZHT16MSWIN950,
#  ZHT16HKSCS, WE8ISO8859P9, TR8MSWIN1254, VN8MSWIN1258
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.characterSet=ZHS16GBK

#------------------------------------------------------------------------------
# This variable should be set to true if Automatic Memory Management
# in Database is desired.
# If Automatic Memory Management is not desired, and memory allocation
# is to be done manually, then set it to false.
#------------------------------------------------------------------------------
oracle.install.db.config.starterdb.memoryOption=true

#-------------------------------------------------------------------------------
# Specify the total memory allocation for the database. Value(in MB) should be
# at least 256 MB, and should not exceed the total physical memory available
# on the system.
# Example: oracle.install.db.config.starterdb.memoryLimit=512
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.memoryLimit=447

#-------------------------------------------------------------------------------
# This variable controls whether to load Example Schemas onto
# the starter database or not.
# The value can be either "true" or "false". If left blank it will be assumed
# to be "false".
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.installExampleSchemas=false

###############################################################################
#                                                                             #
# Passwords can be supplied for the following four schemas in the          #
# starter database:                                    #
#   SYS                                                                       #
#   SYSTEM                                                                    #
#   DBSNMP (used by Enterprise Manager)                                       #
#                                                                             #
# Same password can be used for all accounts (not recommended)               #
# or different passwords for each account can be provided (recommended)       #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# This variable holds the password that is to be used for all schemas in the
# starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.ALL=

#-------------------------------------------------------------------------------
# Specify the SYS password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.SYS=

#-------------------------------------------------------------------------------
# Specify the SYSTEM password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.SYSTEM=

#-------------------------------------------------------------------------------
# Specify the DBSNMP password for the starter database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.DBSNMP=

#-------------------------------------------------------------------------------
# Specify the PDBADMIN password required for creation of Pluggable Database in the Container Database.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.password.PDBADMIN=

#-------------------------------------------------------------------------------
# Specify the management option to use for managing the database.
# Options are:
# 1. CLOUD_CONTROL - If you want to manage your database with Enterprise Manager Cloud Control along with Database Express.
# 2. DEFAULT   -If you want to manage your database using the default Database Express option.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.managementOption=DEFAULT

#-------------------------------------------------------------------------------
# Specify the OMS host to connect to Cloud Control.
# Applicable only when oracle.install.db.config.starterdb.managementOption=CLOUD_CONTROL
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.omsHost=

#-------------------------------------------------------------------------------
# Specify the OMS port to connect to Cloud Control.
# Applicable only when oracle.install.db.config.starterdb.managementOption=CLOUD_CONTROL
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.omsPort=0

#-------------------------------------------------------------------------------
# Specify the EM Admin user name to use to connect to Cloud Control.
# Applicable only when oracle.install.db.config.starterdb.managementOption=CLOUD_CONTROL
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.emAdminUser=

#-------------------------------------------------------------------------------
# Specify the EM Admin password to use to connect to Cloud Control.
# Applicable only when oracle.install.db.config.starterdb.managementOption=CLOUD_CONTROL
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.emAdminPassword=

###############################################################################
#                                                                             #
# SPECIFY RECOVERY OPTIONS                                                   #
# ------------------------------------                                      #
# Recovery options for the database can be mentioned using the entries below  #
#                                                                             #
###############################################################################

#------------------------------------------------------------------------------
# This variable is to be set to false if database recovery is not required. Else
# this can be set to true.
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.enableRecovery=false

#-------------------------------------------------------------------------------
# Specify the type of storage to use for the database.
# It can be one of the following:
#   - FILE_SYSTEM_STORAGE
#   - ASM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.storageType=FILE_SYSTEM_STORAGE

#-------------------------------------------------------------------------------
# Specify the database file location which is a directory for datafiles, control
# files, redo logs.        
#
# Applicable only when oracle.install.db.config.starterdb.storage=FILE_SYSTEM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.fileSystemStorage.dataLocation=/OracleDatafile/app/oracle/oradata

#-------------------------------------------------------------------------------
# Specify the recovery location.
#
# Applicable only when oracle.install.db.config.starterdb.storage=FILE_SYSTEM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.starterdb.fileSystemStorage.recoveryLocation=

#-------------------------------------------------------------------------------
# Specify the existing ASM disk groups to be used for storage.
#
# Applicable only when oracle.install.db.config.starterdb.storageType=ASM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.asm.diskGroup=

#-------------------------------------------------------------------------------
# Specify the password for ASMSNMP user of the ASM instance.                
#
# Applicable only when oracle.install.db.config.starterdb.storage=ASM_STORAGE
#-------------------------------------------------------------------------------
oracle.install.db.config.asm.ASMSNMPPassword=

#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username.
#
#  Example   : MYORACLESUPPORT_USERNAME=abc@oracle.com
#------------------------------------------------------------------------------
MYORACLESUPPORT_USERNAME=

#------------------------------------------------------------------------------
# Specify the My Oracle Support Account Username password.
#
# Example    : MYORACLESUPPORT_PASSWORD=password
#------------------------------------------------------------------------------
MYORACLESUPPORT_PASSWORD=

#------------------------------------------------------------------------------
# Specify whether to enable the user to set the password for
# My Oracle Support credentials. The value can be either true or false.
# If left blank it will be assumed to be false.
#
# Example    : SECURITY_UPDATES_VIA_MYORACLESUPPORT=true
#------------------------------------------------------------------------------
SECURITY_UPDATES_VIA_MYORACLESUPPORT=false

#------------------------------------------------------------------------------
# Specify whether user doesn't want to configure Security Updates.
# The value for this variable should be true if you don't want to configure
# Security Updates, false otherwise.
#
# The value can be either true or false. If left blank it will be assumed
# to be false.
#
# Example    : DECLINE_SECURITY_UPDATES=false
#------------------------------------------------------------------------------
DECLINE_SECURITY_UPDATES=true

#------------------------------------------------------------------------------
# Specify the Proxy server name. Length should be greater than zero.
#
# Example    : PROXY_HOST=proxy.domain.com
#------------------------------------------------------------------------------
PROXY_HOST=

#------------------------------------------------------------------------------
# Specify the proxy port number. Should be Numeric and at least 2 chars.
#
# Example    : PROXY_PORT=25
#------------------------------------------------------------------------------
PROXY_PORT=

#------------------------------------------------------------------------------
# Specify the proxy user name. Leave PROXY_USER and PROXY_PWD
# blank if your proxy server requires no authentication.
#
# Example    : PROXY_USER=username
#------------------------------------------------------------------------------
PROXY_USER=

#------------------------------------------------------------------------------
# Specify the proxy password. Leave PROXY_USER and PROXY_PWD 
# blank if your proxy server requires no authentication.
#
# Example    : PROXY_PWD=password
#------------------------------------------------------------------------------
PROXY_PWD=

#------------------------------------------------------------------------------
# Specify the Oracle Support Hub URL.
#
# Example    : COLLECTOR_SUPPORTHUB_URL=https://orasupporthub.company.com:8080/
#------------------------------------------------------------------------------
COLLECTOR_SUPPORTHUB_URL=
```



# 客户端配置连接信息(免安装Oracle)


**如下为客户端配置**

免安装Oracle客户端使用PL/SQL

1) 下载instantclient-basic-nt-12.1.0.2.0.zip
2) 解压目录到D:\oracle\
3) 创建目录和文件D:\oracle\Network\ADMIN\tnsnames.ora<br>
内容如下:

```bash
ORCL =
  (DESCRIPTION =
    (ADDRESS_LIST =
      (ADDRESS = (PROTOCOL = TCP)(HOST = 192.168.19.249)(PORT = 1521))
    )
    (CONNECT_DATA =
      (SERVICE_NAME = orcl)
    )
  )
```

4) 修改系统环境变量(注,用户也可以,只有这个用户使用才有效)

```bash
ORACLE_HOME  =   D:\oracle
  TNS_ADMIN    =   %ORACLE_HOME%\NETWORK\ADMIN
  NLS_LANG     =   SIMPLIFIED CHINESE_CHINA.ZHS16GBK
```

5) 然后进入PLSQL 
```bash
Developer>工具>首选项>Oracle主目录:D:\oracle  OCI库:D:\oracle\oci.dll
```

6) 登陆名用system
7) 查询数据库名称
`select name from v$service_name`
