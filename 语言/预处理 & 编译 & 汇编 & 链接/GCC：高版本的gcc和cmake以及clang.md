```table-of-contents
```
# 背景
xdp的编译依赖clang，需要先编译clang。但是clang又没有可用的预编译的版本，需要自己收到编译。而clang的编译依赖高版本的gcc、高版本的cmake、Python3.8都需要安装。

# 高版本的clang
## 高版本的gcc
centos 7.4 默认提供的GCC的版本只有 4.8.5, 这对于编译一些应用 程序是不够用的。

安装高版本gcc，可以使用`devtoolset-9`，是`centos`为低版本的GCC环境提供的工具链，会安装到`/opt/rh`目录下，通过`source /opt/rh/devtoolset-9/enable`文件，会在`PATH`的最开始位置添加`/opt/rh/devtoolset-9/root/usr/bin/`，同时也会添加一些其他的环境变量（比如：`LD_LIBRARY_PATH`, `PKG_CONFIG_PATH`, `MANPATH`等等）。

### SCL
SCL(Software Collections)是一个 CentOS/RHEL Linux 平台的软件多版本共存解决方案，为 RHEL/CentOS Linux 用户提供一种方便、安全地安装和使用应用程序和运行时环境的多个版本的方式，同时避免把系统搞乱。

不同平台的编译环境不一样，所以 RedHat 就推出了 scl (Software Collections) ，它可以根据 devtoolset 一起配合来快速统一开发环境，不用一个个的去找各个官网再去编译源码安装。

使用 scl 可以暂时的改变当前用户的编译工具，例如你的系统版本 gcc 4.4.7 但是你可以使用 scl 工具它可以临时的把你的 gcc 版本提升到 4.8。

当然，除了 devtoolset 这些专门用于编译开发的工具集，SCL 上还有其他的很多工具集，如 Ruby]，Redis，nginx 等等。

![](attachments/Pasted%20image%2020241211111709.png)

### Devtoolset
Devtoolset（Developer Toolset），devtoolset 就是 SCL 提供的一套专门用于 CentOS 或 Red Hat Enterprise Linux 平台编译开发的一套工具集。

![](attachments/Pasted%20image%2020241211111553.png)


### 安装配置 SCL YUM 源
在 CentOS 7 可以通过 yum 直接安装 SCL 源。
```bash
yum install centos-release-scl centos-release-scl-rh -y
```
安装完成后，会默认在 **/etc/yum.repos.d** 下生成 2 个 repo 源文件：
`CentOS-SCLo-scl.repo` 和 `CentOS-SCLo-scl-rh.repo`。

#### CentOS-SCLo-scl.repo 调整
由于CentOS 7、8 和 Stream 8 已停止更新、停止维护；官方已将原本的镜像列表（Mirrorlist）下架并归档，导致通过`yum.repo`中的原始链接无法再访问原来的镜像源。

**不调整的影响**：
如果不调整repo文件，那么centos 7 停止维护，那么提供的镜像列表(Mirrorlist)无法访问。那么后续的通过 yum 安装 devtoolset 等安装包就会出错。

**更改方法**：
```text
注释掉 mirrorlist，放开 baseurl 的注释，
并且将baseurl 中的 mirror.centos.org改成vault.centos.org。
```

**更改的命令**：
```bash
#这条命令使用sed编辑器将所有/etc/yum.repos.d/CentOS-*文件中的mirrorlist行注释掉。
sudo sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*

#这条命令将所有/etc/yum.repos.d/CentOS-*文件中被注释掉的baseurl行改为指向vault.centos.org，并取消注释。
sudo sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
```

**更改后的：`/etc/yum.repos.d/CentOS-SCLo-scl.repo` 文件**，
如下所示：
```
# CentOS-SCLo-sclo.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/SCLo for more
# information

[centos-sclo-sclo]
name=CentOS-7 - SCLo sclo
baseurl=http://vault.centos.org/centos/7/sclo/$basearch/sclo/
#mirrorlist=http://mirrorlist.centos.org?arch=$basearch&release=7&repo=sclo-sclo
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-sclo-testing]
name=CentOS-7 - SCLo sclo Testing
baseurl=http://buildlogs.centos.org/centos/7/sclo/$basearch/sclo/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-sclo-source]
name=CentOS-7 - SCLo sclo Sources
baseurl=http://vault.centos.org/centos/7/sclo/Source/sclo/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-sclo-debuginfo]
name=CentOS-7 - SCLo sclo Debuginfo
baseurl=http://debuginfo.centos.org/centos/7/sclo/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
```

#### CentOS-SCLo-scl-rh.repo 调整

同上。
更改后的：`/etc/yum.repos.d/CentOS-SCLo-scl-rh.repo` 文件如下所示：
```text
# CentOS-SCLo-rh.repo
#
# Please see http://wiki.centos.org/SpecialInterestGroup/SCLo for more
# information

[centos-sclo-rh]
name=CentOS-7 - SCLo rh
baseurl=http://vault.centos.org/centos/7/sclo/$basearch/rh/
#mirrorlist=http://mirrorlist.centos.org?arch=$basearch&release=7&repo=sclo-rh
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-testing]
name=CentOS-7 - SCLo rh Testing
baseurl=http://buildlogs.centos.org/centos/7/sclo/$basearch/rh/
gpgcheck=0
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-source]
name=CentOS-7 - SCLo rh Sources
baseurl=http://vault.centos.org/centos/7/sclo/Source/rh/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

[centos-sclo-rh-debuginfo]
name=CentOS-7 - SCLo rh Debuginfo
baseurl=http://debuginfo.centos.org/centos/7/sclo/$basearch/
gpgcheck=1
enabled=0
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo

```


#### 清理yum缓存并生成新的缓存
```bash
sudo yum clean all // 清理缓存
sudo yum makecache // 生成新的缓存

sudo yum update // yum 更新
```

#### 查看对应源的软件包信息
yum 源更新完后，就可以使用以下的命令查看对应源的软件包信息。

```bash
# 查看 centos-sclo-rh 源所有可用的软件包
$ yum list all --enablerepo='centos-sclo-rh'

# 查看 centos-sclo-rh 源中名为 scl-utils 的软件包
$ yum search scl-utils --enablerepo='centos-sclo-rh'
```

#### 小结

感觉安装 SCL YUM源，主要是 生成了 `yum.repo` 文件，用于后续的 `devtoolset` 的 安装提供 yum 源设置。

### 安装 devtoolset
![](attachments/Pasted%20image%2020241211114721.png)

不同的 devtoolset 对应了不同的 gcc 版本。
```bash
devtoolset-1 是 gcc 4.7
devtoolset-2 是 gcc 4.8
devtoolset-3 是 gcc 4.9
devtoolset-4 是 gcc 5.2/5.3
devtoolset-6 是 gcc 6.2/6.3
devtoolset-7 是 gcc 7.2/7.3
devtoolset-9 是 gcc 9.3
```

比如：
```bash
yum install devtoolset-9 -y
```

### 启用高版本的GCC

使用命令：
`source /opt/rh/devtoolset-9/enable`

原理：
将 `/opt/rh/devtoolset-9/root`下的 `lib` 以及 `bin` 放入到 之前的 `LD_LIBRARY_PATH` 以及 `PATH`的前面，这样在查找可执行文件以及`lib`库时，就优先查找到  `/opt/rh/devtoolset-9/root` 目录下的可执行文件以及lib库文件。

![](attachments/Pasted%20image%2020241211141201.png)

```bash
# which gcc
/opt/rh/devtoolset-9/root/usr/bin/gcc
```

#### 其他
==`libstdc++`是 gcc的标准C++库(`libc++`是clang的标准C++库)；`libstdc++`是被包含在gcc中的，对应为gcc的版本 `gcc --version`==。

更新 `libstdc++` 动态库：如果gcc动态库版本太老，还需要升级libstdc++.so.6才行。
```bash
# 更新lib libstdc++.so.6.0.26
wget https://cdn.frostbelt.cn/software/libstdc%2B%2B.so.6.0.26

# 替换系统中的/usr/lib64
把下载的libstdc++.so.6.0.26 cp 到 /usr/lib64/

cd /usr/lib64/
ln -snf ./libstdc++.so.6.0.26 libstdc++.so.6
```

### 卸载 devtoolset
可能大家用完开发工具集后就会想要删除它，其实很简单，输入以下命令
```bash
yum remove devtoolset-9 -y
```

## 高版本的python（python3.8）
```bash
安装：
yum install -y rh-python38

启用：
source /opt/rh/rh-python38/enable
```
## 高版本的cmake
### 编译cmake
cmake使用的是官方预编译版本，可以源码编译，但是太慢了。
因此，选择预编译版本。

https://github.com/Kitware/CMake/releases/download/v3.31.0-rc1/cmake-3.31.0-rc1-linux-x86_64.tar.gz

解压到 `/opt/rh/cmake-3.31.0/root/` 即可
```bash
mkdir -p /opt/rh/cmake-3.31.0/root/; 
tar xf cmake-3.31.0-rc1-linux-x86_64.tar.gz -C /opt/rh/cmake-3.31.0/root/
```

### 启动cmake

自己创建 `/opt/rh/cmake-3.31.0/enable` 文件，让`cmake`可以手动开启，文件内容如下:
```bash
# General environment variables
DIR=/opt/rh/cmake-3.31.0/root/
export PATH=${DIR}/bin${PATH:+:${PATH}}
export MANPATH=${DIR}/share/man:${MANPATH}
```

启动的命令：
```bash
source /opt/rh/cmake-3.31.0/enable
```


## 高版本的clang

先编译`llvm`再编译`clang`。

### 编译llvm
llvm 没有预编译版本，需要自己源码编译。
```bash
# 源码下载
https://github.com/llvm/llvm-project/archive/refs/tags/llvmorg-19.1.2.tar.gz
```
下载`llvm`之后，解压；

参考了高版本`gcc`的方式，将`llvm`的编译产出安装放在了`/opt/rh/llvm-19.1.2/root/`目录下。需要先创建这个目录。

`llvm`在源码目录的`llvm`目录下，本次编译的目录是`/llvm-project-llvmorg-19.1.2/llvm/`;

编译命令如下所示：
```bash
mkdir build
cd build
cmake ..   --install-prefix /opt/rh/llvm-19.1.2/root/
make
make install
```

### clang的编译

clang的编译编译依赖`llvm-gtest`，添加 `-DLLVM_INCLUDE_TESTS=OFF`选项之后就不用这个依赖了，可以直接编译

本次编译的目录是`/llvm-project-llvmorg-19.1.2/clang`

编译命令，如下所示：
```bash
mkdir build
cd build
cmake ..  -DLLVM_INCLUDE_TESTS=OFF --install-prefix /opt/rh/llvm-19.1.2/root/
make
make install
```

### 启动clang
创建`/opt/rh/llvm-19.1.2/enable`文件，内容如下
```bash
# General environment variables
DIR=/opt/rh/llvm-19.1.2/root/
export PATH=${DIR}/bin${PATH:+:${PATH}}
export MANPATH=${DIR}/share/man:${MANPATH}
export LD_LIBRARY_PATH=${DIR}/lib${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}
```

使用命令
```bash
source /opt/rh/llvm-19.1.2/enable
```


### 制作二进制版本的clang
#### 背景
如上所示，上面的很多工具都是源码编译，预编译的比较少。

编译生成高版本的`clang`,需要一系列的动作，并且编译的过程很漫长。如果存在多个机器都需要安装高版本的 clang，那么编译过程以及下载过程，以及环境的准备将是一个漫长的过程。

#### 思路
将源码编译尽可能的手动转换为预编译的产出。

可以在一个机器行，执行上面的编译工作。然后将产出给打包。
其他的机器，只需要将压缩包解压即可。

#### 解决方法
```bash
打包命令：
    tar czf opt_clang.tar.xz /opt/rh/ 

解压命令:
    tar xf opt_clang.tar.xz -C /

使能：
    source /opt/rh/devtoolset-9/enable # 使能高版本 gcc
    source /opt/rh/rh-python38/enable  # 使能高版本 python
    source /opt/rh/llvm-19.1.2/enable  # 使能高版本的 clang
    source /opt/rh/cmake-3.31.0/enable # 使能高版本 cmake
  
检查：
  gcc --version
  python --version
  clang --version
  cmake --version
```
# 应用
## xdp 库的安装与编译
### 安装libpcap
#### 源码编译
```bash
源码：
https://www.tcpdump.org/release/libpcap-1.10.5.tar.xz
```

`configure`的时候，需要指定目录，保证`libpcap.pc`安装到可以被`pkg-config`找到的位置，否则需要更改 `PKG_CONFIG_PATH` 环境变量。
```bash
yum install -y flex bison
wget https://www.tcpdump.org/release/libpcap-1.10.5.tar.xz
tar xvf libpcap-1.10.5.tar.xz
cd libpcap-1.10.5
./configure --prefix=/usr/ --libdir=/usr/lib64/
make
make install
```
### 编译xdp
```bash
# 其他依赖
yum install -y automake  elfutils elfutils-libelf-devel

# 编译xdp
source /opt/rh/devtoolset-9/enable
source /opt/rh/llvm-19.1.2/enable
git clone https://github.com/xdp-project/xdp-tools.git
cd xdp-tools
git checkout v1.4.3
./configure
make
DESTDIR=/usr/ LIBDIR=lib64 make install
```


# 参考
```bash
# SCL 笔记之 Devtoolset 安装与使用
https://weiyan.cc/yuque/%E5%BC%80%E5%8F%91%E8%BF%90%E7%BB%B4/%E7%B3%BB%E7%BB%9F%E4%B8%8E%E7%BC%96%E8%AF%91/2021-09-02-scl-devtoolset-note/#devtoolset
```