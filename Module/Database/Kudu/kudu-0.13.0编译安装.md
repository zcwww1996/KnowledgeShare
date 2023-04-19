[TOC]

编译版本：kudu-0.13.0

操作系统要求：RHEL 7, RHEL 8, CentOS 7, CentOS 8, Ubuntu 18.04 (bionic), Ubuntu 20.04 (focal)

本次采用CentOS-7-x86_64-DVD-1810
# 1. 安装java

JDK 8是构建Kudu所必需的，但运行时不需要JRE（测试除外）

# 2.安装必备库


```bash
yum install  -y autoconf automake cyrus-sasl-devel cyrus-sasl-gssapi \
  cyrus-sasl-plain flex gcc gcc-c++ gdb git  \
  krb5-server krb5-workstation libtool make openssl-devel patch \
  pkgconfig redhat-lsb-core rsync unzip vim-common which
```

# 3. 安装Red Hat Developer Toolset

要在RHEL或CentOS8.0之前的版本上进行构建，必须安devtoolset（以便能够访问具有C ++ 17功能的编译器）

查看gcc版本

```bash
gcc --version

结果：gcc (GCC) 4.8.5 20150623 (Red Hat 4.8.5-44)
```

安装devtoolset
```bash
yum install centos-release-scl-rh
yum install devtoolset-8
```

切换到新版gcc

```bash
cd /opt/rh/devtoolset-8
scl enable devtoolset-8 bash

gcc --version #再次查看gcc版本
结果：gcc (GCC) 8.3.1 20190311 (Red Hat 8.3.1-3)
```

# 4. 安装memkind库(非必要)
可选：如果需要支持Kudu的NVM（非易失性内存）块缓存，请安装memkind库。


```bash
yum install -y memkind
```

如果Linux发行版随附的memkind软件包过旧（需要1.8.0或更高版本），请从源代码构建并安装它。


> ps: 查看yum软件版本<br>
> 方法1：`yum list |grep memkind`<br>
> 方法2：`yum info memkind`

```bash
yum install -y numactl-libs numactl-devel
 git clone https://github.com/memkind/memkind.git
cd memkind
./build.sh --prefix = / usr
sudo yum remove memkind
make&make install
ldconfig
```

# 5.构建文档(非必要)
可选：如果您打算构建文档，请安装一些其他软件包，包括ruby。


```bash
yum install -y gem graphviz ruby-devel zlib-devel
```

> - 如果在7.0之前的RHEL或CentOS上构建，则可能需要用rubygems替换gem软件包。
> - 需要Doxygen 1.8.19或更高版本来构建文档，该文档必须 从源代码手动构建。在CentOS或8.0之前的RHEL上构建此版本的Doxygen也需要 devtoolset。

# 6. 下载源码包

```bash
tar -zxvf apache-kudu-1.13.0.tar.gz -C /usr/local
mv apache-kudu-1.13.0 kudu-1.13.0
```

# 7. 下载第三方依赖包
使用build-if-necessary.sh脚本构建依赖。不使用devtoolset将导致`Host compiler appears to require libatomic, but cannot find it.`


```bash
cd /usr/local/kudu-1.13.0
build-support/enable_devtoolset.sh thirdparty/build-if-necessary.sh
# 这个命令会把下载的依赖全部安装,执行Kudu的./configura
```
> **补充**:<br>
> 如果依赖下载慢，可以下载后放入`/usr/local/kudu-1.13.0/thirdparty/src`
> 
> 
> ```
> mkdir thirdparty/src/
> cd thirdparty/src/
> 
> wget http://d3dr9sfxru4sde.cloudfront.net/chrony-3.5.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/gumbo-parser-aa91b27b02c0c80c482e24348a457ed7c3c088e0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/gumbo-query-c9f10880b645afccf4fbcd11d2f62a7c01222d2e.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/postgresql-12.2.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/postgresql-42.2.10.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/yaml-cpp-yaml-cpp-0.6.2.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/glog-0.3.5.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/googletest-release-1.8.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/gflags-2.2.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/gperftools-2.6.90.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/protobuf-3.12.3.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/cmake-3.16.4.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/snappy-1.1.8.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/zlib-1.2.11.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/libev-4.20.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/rapidjson-1.1.0.zip
> wget http://d3dr9sfxru4sde.cloudfront.net/sparsehash-c11-cf0bffaa456f23bc4174462a789b90f8b6f5f42f.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/mustache-b290952d8eb93d085214d8c8c9eab8559df9f606.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/google-styleguide-7a179d1ac2e08a5cc1622bec900d1e0452776713.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/gcovr-3.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/curl-7.68.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/crcutil-81f8a60f67190ff1e0c9f2f6e5a07f650671a646.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/libunwind-1.4.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/python-2.7.13.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/llvm-9.0.0.src.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/lz4-1.9.2.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/bitshuffle-0.3.5.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/kudu-trace-viewer-21d76f8350fea2da2aa25cb6fd512703497d0c11.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/boost_1_61_0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/breakpad-9eac2058b70615519b2c4d8c6bdbfca1bd079e39.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/squeasel-030ccce87359d892e22fb368c5fc5b75d9a2a5f7.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/sparsepp-1.22.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/thrift-0.11.0.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/bison-3.5.4.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/hive-3.1.1.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/hadoop-3.2.0-stripped.tar.gz
> wget http://d3dr9sfxru4sde.cloudfront.net/ranger-f37f5407eee8d2627a4306a25938b151f8e2ba31-admin.tar.gz
> ```

# 8. 编译kudu
使用上一步中安装的依赖来构建Kudu。为中间输出结果选择一个构建目录，该目录位于文件系统中除kudu目录本身之外的任何位置(不能使用`/usr/local/kudu-1.13.0`目录)。注意，仍必须指定devtoolset，否则将得到`cc1plus: error: unrecognized command line option "-std=c++17"`。

**`-DNO_TESTS=1`,cmake编译剔除test文件，test文件特别大**

```bash
mkdir -p build/release
cd build/release

../../build-support/enable_devtoolset.sh \
../../thirdparty/installed/common/bin/cmake \
-DNO_TESTS=1 \
-DCMAKE_BUILD_TYPE=release ../..

make -j4
```

编译报错：`curl: (7) Failed connect to raw.githubusercontent.com:443; Connection refuse`

打开网站https://www.ipaddress.com/<br>
查询一下`raw.githubusercontent.com`对应的IP 地址<br>
在/etc/hosts文件的最后加上ip和raw.githubusercontent.com

# 9. 指定目录安装kudu(非必要)

可选：安装Kudu可执行文件，库和头文件，默认安装目录为/usr/local。可以通过DESTDIR 环境变量自定义它

```bash
make DESTDIR=/usr/local/kudu install
```

# 10. 构建文档(非必要)

```bash
make docs
```

