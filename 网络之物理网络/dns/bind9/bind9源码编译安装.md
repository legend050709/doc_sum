```table-of-contents
```
# bind 9安装
## 默认安装目录
**默认情况下，BIND 将安装在 /usr/local 中，并将文件放在它的这些子目录中：**
```text
sbin    # named 和所有与 BIND 相关的系统管理工具，例如 rndc、dnssec-keygen、named-checkconf 等。
bin      # 非管理员用户的工具 - 你会在这里找到 dig、host 和 nsupdate
lib       # 目标代码库
share   # BIND的手册页和各种子目录
include    # C头文件
```

编译时**未更改默认目录**的 **BIND** 将使用以下目录（相对于根目录 "/"）
```text
/etc    # 配置文件（例如 named.conf、rndc.conf）
/var/run    # named 创建和使用的运行时文件
```


## 源码编译安装

（1）新建构建目录 下载的源码包放置在该目录中
```text
root@dns01:/home# mkdir bulids
root@dns01:/home/bulids# pwd
/home/bulids

# wget可能会比较慢
root@dns01:/home/bulids# wget https://downloads.isc.org/isc/bind9/9.18.9/bind-9.18.9.tar.xz
root@dns01:/home/bulids# tar -xvf bind-9.18.9.tar.xz

# 进入解压后的目录中
root@dns01:/home/bulids/bind-9.18.9# pwd
/home/bulids/bind-9.18.9
```

（2）编译：
```text
# 查看编译选项参数帮助
root@dns01:/home/bulids/bind-9.18.9# ./configure --help

# 编译依赖库安装
root@dns01:/home/bulids/bind-9.18.9# apt-get install gcc zlib1g-dev build-essential pkg-config autoconf automake libtool libssl-dev libcap-dev libxml2 libxml2-dev libfstrm-dev libprotobuf-c-dev liblmdb-dev libmaxminddb-dev libidn2-dev libreadline-dev libjson-c-dev fstrm-bin protobuf-c-compiler

# 源码编译libuv依赖 辅助服务器使用了 libuv-1.44.2 版本
root@dns01:/home/bulids/libuv-1.42.0# wget https://dist.libuv.org/dist/v1.42.0/libuv-v1.42.0.tar.gz
root@dns01:/home/bulids# tar -zxvf libuv-v1.42.0.tar.gz
root@dns01:/home/bulids# cd libuv-1.42.0/
root@dns01:/home/bulids/libuv-1.42.0# pwd
/home/bulids/libuv-1.42.0
root@dns01:/home/bulids/libuv-1.42.0# sh autogen.sh
root@dns01:/home/bulids/libuv-1.42.0# ./configure
root@dns01:/home/bulids/libuv-1.42.0# make -j4
root@dns01:/home/bulids/libuv-v1.42.0# make install
root@dns01:/home/bulids/libuv-v1.42.0# ldconfig

# 源码编译jemalloc 官方推荐更换内存分配器为jemalloc 解决OOM问题 提高性能
root@dns01:/home/bulids# wget https://github.com/jemalloc/jemalloc/releases/download/5.3.0/jemalloc-5.3.0.tar.bz2
root@dns01:/home/bulids# tar -xjvf jemalloc-5.3.0.tar.bz2
root@dns01:/home/bulids# cd jemalloc-5.3.0
root@dns01:/home/bulids/jemalloc-5.3.0# ./configure
root@dns01:/home/bulids/jemalloc-5.3.0# make -j8
root@dns01:/home/bulids/jemalloc-5.3.0# make install
root@dns01:/home/bulids/jemalloc-5.3.0# ldconfig

# 官方关于内存OOM问题分析
https://www.isc.org/blogs/jemalloc-glitch/

# 源码编译bind 9  编译帮助执行 ./configure --help
root@dns01:/home/bulids# cd bind-9.18.9
root@dns01:/home/bulids/bind-9.18.9# ./configure --prefix=/usr/local/bind --with-openssl --with-json-c --enable-largefile --with-libidn2 --enable-full-report --enable-dnstap --disable-doh
root@dns01:/home/bulids/bind-9.18.9# make
root@dns01:/home/bulids/bind-9.18.9# make install
```
**编译参数说明：**
```text
--prefix=/usr/local/bind    # 定义安装目录 如不定义则安装默认目录安装
--sysconfdir=$PERFIX/etc    # 如定义安装目录 就根据--prefix参数定义的目录 创建下一级配置文件所在目录
--localstatedir=$PERFIX/var    # 如定义安装目录 就根据--prefix参数目录 创建下一级目录named运行动态生成的文件目录
--with-openssl    # 开启openssl
--with-json-c    # HTTP 统计信息格式
--enable-largefile    # 2G以上大文件支持
--with-libidn2    # dig 国际化通用域名 IDN支持
--enable-full-report    # 编译显示全部报告
--enable-dnstap    # 开启dnstap日志记录
--disable-doh    # 关闭DNS over HTTPS
```

（3）编译安装完成后
如下所示：**/usr/local/bind**
![](attachments/Pasted%20image%2020240304110149.png)


## 编译文档说明
```text
# 官方关于编译的说明
https://bind9.readthedocs.io/en/v9_18_9/chapter10.html
```
# 参考
```bash
# 构建企业级DNS系统（一）CentOS7源码安装bind9
https://blog.csdn.net/u011288801/article/details/106736607

# 构建企业级DNS系统（二）源码安装bind开机启动服务设置
https://blog.csdn.net/u011288801/article/details/106737855
```