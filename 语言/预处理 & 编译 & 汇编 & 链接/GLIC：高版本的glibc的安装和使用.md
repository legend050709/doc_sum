```table-of-contents
```
# gcc 和 glibc
## gcc介绍
GCC（GNU Compiler Collection）是一个编译器套件，它可以编译C、C++、Java、Go等多种编程语言的代码。通过将源代码编译成机器码，GCC使程序可以在操作系统上直接运行。

## glibc介绍
GNU C 库 (glibc) 是 Linux 操作系统的基本组件，为各种应用程序提供基本功能。
glibc是linux系统中**最底层的api，几乎其它任何运行库都会依赖于glibc**。
glibc除了封装linux操作系统所提供的系统服务外，它本身也提供了许多其它一些必要功能服务的实现。
由于**glibc和内核不是一块开发的**，所以**glibc需要去兼容不同版本的内核，而内核也要去兼容不同版本的 glibc**，双方都背负了太多的历史包袱。

### glibc的版本要求
某些由glibc提供的函数可能只在特定的glibc版本或更高版本中存在。
**如果编译的应用程序代码使用了glibc这些特性，那么生成的二进制就需要相应版本或更高版本的glibc**。
这种依赖性通常是由于新增的API函数或对旧函数的改进。

### 程序如何保持向后兼容
#### 静态链接

通过将应用程序与所有需要的库文件进行静态链接，可以生成不依赖于特定版本glibc的可执行文件。
这样，即使在含有较低版本glibc的系统中，程序也能正常运行。
然而，这会导致程序体积增大并可能带来其他兼容性问题。

#### 使用旧版本的API

如果希望编译的程序能够在较旧的系统上运行，应使用在老版本glibc中已存在的API。
这可能限制了程序能使用的新特性，但在它提高了程序的兼容性。

## gcc 和 glibc的关系

### 高版本gcc编译出来的代码是否一定需要高版本glibc支持？
高版本gcc编译出来的代码并不一定需要高版本glibc支持。
gcc和glibc是两个独立的软件包，它们**一个供编译器使用，一个是运行时库使用**。
**gcc主要用于编译源代码生成可执行文件，而glibc是用于提供运行时环境的C库**。
当你使用高版本的gcc编译代码时，它会使用与gcc版本兼容的glibc版本进行链接。因此，只有在运行时，你需要确保相应版本的glibc存在于目标系统中。

> 如果你使用的是非常新的gcc版本，可能需要更新glibc以确保代码在运行时能够正常工作。



## 相关问题
**(1) 怎么使多个gcc/glib版本共存**
**(2) 在多个gcc/glib版本共存的情况，怎么样指定一个版本进行编译**
**(3) 怎么在一个与编译环境不同(gcc/glibc版本不同)的机器上运行服务**


# 高版本gcc编译出的程序在低版本glibc机器上运行的问题

比如我们用gcc 9.3.0编译程序，但需要发布的机器gcc版本是4.8.5，怎么办？

参考：[高版本gcc编译出的程序在低版本glibc机器上运行](https://blog.csdn.net/bandaoyu/article/details/121476940)

**问题**：
高版本GCC编译的程序能否在低版本GLIBC环境下顺利运行？


**回答**：
我们需要明确GCC和GLIBC的关系。
GCC是GNU Compiler Collection（GNU编译器集合）的简称，是Linux环境下最常用的编译器之一。
而GLIBC则是Linux系统中默认的C库，提供了大量的系统调用接口和函数实现。
GCC在编译程序时，会链接到相应的GLIBC库，因此生成的程序对GLIBC版本有一定的依赖关系。

当使用高版本的GCC编译程序时，可能会引入一些低版本GLIBC不支持的特性或函数。如果直接在低版本GLIBC环境下运行这些程序，很可能会出现运行时错误，如找不到符号、动态链接失败等。这是因为高版本的GCC可能使用了低版本GLIBC中没有的函数或特性，导致生成的程序无法在低版本环境下正确运行。
> 总结：==其实编译环境的gcc版本高，本质上还是glibc版本高，运行机器上glibc版本较低，不存在某些函数==。

## 解决思路
你可能想到如下方法
1. 编译时静态链接
2. 运行时动态链接，但指定glibc版本
3. 使用兼容性库
4. 条件编译
5. 打包依赖的so，使用本地so运行程序


### 编译设备上处理
#### 编译时静态链接
**思路**：
在编译程序时，使用静态链接方式（--static）将所需的GLIBC库与程序一起打包。这样，程序运行时就不会受到目标系统GLIBC版本的影响。
但需要注意的是，静态链接会增加程序的体积，并可能导致一些潜在的符号冲突问题。

**解决**：
```bash
1.在linux中用yum下载安装glibc和libstdc++的静态库
sudo yum install glibc-static libstdc++-static

2.在编译选项LDFLAGS中添加--static即可正常编译运行。
--satic会将所有库都变成静态的。gcc有内置加入libc的。
```


#### 在低版本 GLIBC 环境下重新编译
如果可能的话，在低版本的 GLIBC 环境下重新编译程序，确保程序在目标系统上能够正确运行。

#### 使用兼容性库
**思路**
开发者可以寻找一些兼容性库，这些库提供了高版本GLIBC中的函数实现，但可以在低版本GLIBC环境下正常工作。通过在编译时链接这些兼容性库，可以让程序在低版本GLIBC环境下顺利运行。

#### 条件编译
在代码中使用条件编译，根据目标系统的GLIBC版本来选择性地调用不同的函数或特性。这样，可以确保程序在不同版本的GLIBC环境下都能正确运行。但这种方式需要开发者对GLIBC的版本差异有深入的了解，并需要花费一定的时间和精力来编写和维护条件编译的代码。

### 运行设备上处理
#### 运行时动态链接，但指定glibc版本
可以使用 `LD_LIBRARY_PATH` 环境变量来指定程序运行时使用的 GLIBC 版本。这样可以强制程序使用特定版本的 GLIBC 库。

#### 容器中运行程序
将程序打包到一个容器中，如 Docker，确保容器中包含了程序所需的依赖库和环境，这样可以避免 GLIBC 版本不匹配的问题。

#### 升级目标系统的GLIBC

如果条件允许，开发者也可以考虑升级目标系统的GLIBC版本。这样可以让系统支持更多高版本的GCC编译的程序，但需要注意升级GLIBC可能会影响到系统其他部分的稳定性和兼容性。

##### 安装部署devtoolset(开发工具集)
因为源码安装的难度不低，安装`devtoolset可能是不错的选择`。

RedHat推出`Software Collections(SCL)`的目的就是为了解决想在`RedHat`系统下能使用新版本的工具，让同一个工具（如gcc）的不同版本能在系统中共存，在需要的时候切换到对应的版本中。类似 pyenv(python)、rvm(ruby) 或者 nvm(node)。`RedHat`与`CentOS`师出同源，一样可以使用。


#### 打包依赖的so库发布（通用方案）
##### 在编译时设置rpath和dynamic linker
当你有条件获得程序源码，并能够重新编译时，可以直接在编译时指定相关参数来解决。
在编译时候，指定运行时 库的查找路径，以及版本。先说编译时要增加的参数：
```bash
# 绝对路径
gcc -Wl,-rpath='/my/lib',-dynamic-linker='/my/lib/ld-linux.so.2'


# 说明：
-rpath=dir: 
	Add a directory to the runtime library search path. This is used when linking an ELF executable with shared objects.
--dynamic-linker=file:
	Set the name of the dynamic linker.

```

上面的两个参数分别设置的elf文件中的rpath和interpreter字段。

**rpath**  
全名`run-time search path`，是elf文件中一个字段，它指定了可执行文件执行时搜索so文件的第一优先位置，一般编译器默认将该字段设为空。elf文件中还有一个类似的字段runpath，其作用与rpath类似，但搜索优先级稍低。搜索优先级:
```bash
rpath > LD_LIBRARY_PATH > runpath > ldconfig缓存 > 默认的/lib,/usr/lib等
```
如果你需要使用相对路径指定lib文件夹，可以使用ORIGIN变量，ld会将ORIGIN理解成可执行文件所在的路径。
```bash
gcc -Wl,-rpath='$ORIGIN/../lib'
```

**interpreter** 
全名elf interpreter，用于加载elf文件。这个字段在链接时会帮你自动设置，64bit程序一般为/lib64/ld-linux-x86-64.so.2。这也是打包so的坑之一，很多人（比如我）通过ldd找出程序依赖的so，进行打包后，在目标机器修改rpath或者LD_LIBRARY_PATH指向本地lib目录，但ldd程序，发现/lib64/ld-linux-x86-64.so.2这个so仍然指向系统so。原因就是这个字段是写死在elf文件中的，并不受LD_LIBRARY_PATH影响。


编译时带上这两个参数，下面只需要将你程序依赖的so打包一份，随程序进行发布即可。

##### 直接修改二进制程序的rpath和interpreter

当你无法编译程序时，也可以通过其他方式修改rpath和interpreter。

这种情况需要使用到一个工具`patchelf`，通过`dnf install patchelf`即可安装。你可以通过它修改elf文件的rpath和interpreter：
```bash
patchelf --set-rpath /my/lib your_program
patchelf --set-interpreter /my/lib/ld-linux.so.2 your_program
```

除了绝对路径，一种比较常见的方式是在部署前，使用`pwd`获取当前路径，使用相对路径指向本地lib。
```bash
patchelf --set-rpath `pwd`/../lib your_program
patchelf --set-interpreter `pwd`/../lib/ld-linux-x86-64.so.2 ./your_program
```


## 高版本的gcc
### 升级gcc到gcc8
```bash
# 安装devtoolset-8-gcc
yum install centos-release-scl
yum install devtoolset-8
scl enable devtoolset-8 -- bash
 
# 启用工具
source /opt/rh/devtoolset-8/enable 
 
# 安装GCC-8
yum install -y devtoolset-8-gcc devtoolset-8-gcc-c++ devtoolset-8-binutils
 
# 设置环境变量
echo "source /opt/rh/devtoolset-8/enable" >> /etc/profile
source /etc/profile
```
### 手动安装高版本的gcc
```bash
# 1.安装升级依赖
yum install -y gcc-c++ glibc-devel mpfr-devel libmpc-devel gmp-devel glibc-devel.i686
​
# 2.下载gcc9.3.1安装包
cd /backup
wget https://ftp.gnu.org/gnu/gcc/gcc-9.3.0/gcc-9.3.0.tar.gz --no-check-certificate
​
# 3.解包并执行编译前的准备
tar -xf gcc-9.3.0.tar.gz
cd gcc-9.3.0
# 下载依赖包
./contrib/download_prerequisites
# 建立构建目录
mkdir build
# 进入构建目录
cd build
​
# 4.指定安装到具体的目录下，此示例表示将make安装到/usr下(说明：若安装到非/usr目录,如安装到/opt/gcc，则在编译完成后需要配置环境变量、建立软连接。)
../configure --enable-checking=release --enable-language=c,c++ --disable-multilib --prefix=/usr
​
# 5.编译安装
make -j4 # -j代表编译时的任务数，一般有几个cpu核心就写几，构建速度会更快一些。该步骤执行时间很长
make install
​
# 6.上一步若安装目录不是/usr，则需要在编译完成后配置环境变量、建立软连接，若为/usr目录则跳过此步骤
# 假设将gcc编译安装到了/opt/gcc目录
# 6.1配置环境变量
vi /etc/profile.d/gcc.sh
# gcc环境配置
export PATH=/opt/gcc/bin:$PATH
export LD_LIBRARY_PATH=/opt/gcc/lib
# 编辑完成后:wq保存并退出
# 6.2重载环境变量
source /etc/profile
# 6.3重新生成新的链接
# 取消原始链接
unlink /usr/bin/cc
# 建立新链接
ln -sf /opt/gcc/bin/gcc /usr/bin/cc
ln -sf /opt/gcc/lib/gcc/x86_64-pc-linux-gnu/9.3.0/include /usr/include/gcc
# 设置库文件
echo "/opt/gcc/lib64" >> /etc/ld.so.conf.d/gcc.conf
# 加载动态连接库
ldconfig -v
# 查看加载结果
ldconfig -p | grep gcc
​
# 7.安装完成后检查gcc版本，若gcc升级失败则需查找失败原因并重新进行升级操作
gcc -v
Using built-in specs.
COLLECT_GCC=gcc
COLLECT_LTO_WRAPPER=/usr/libexec/gcc/x86_64-pc-linux-gnu/9.3.0/lto-wrapper
Target: x86_64-pc-linux-gnu
Configured with: ../configure --enable-checking=release --enable-language=c,c++ --disable-multilib --prefix=/usr
Thread model: posix
gcc version 9.3.0 (GCC)
```

### 安装部署devtoolset(开发工具集)

因为源码安装的难度不低，安装`devtoolset可能是不错的选择`。

RedHat推出`Software Collections(SCL)`的目的就是为了解决想在`RedHat`系统下能使用新版本的工具，让同一个工具（如gcc）的不同版本能在系统中共存，在需要的时候切换到对应的版本中。类似 pyenv(python)、rvm(ruby) 或者 nvm(node)。`RedHat`与`CentOS`师出同源，一样可以使用。


（1）查询合适的 devtoolset。
比如linux kernel 4.16对应的systemtap的源码版本要求systemtap-3.3以上。在`devtoolset-7`的安装包中[devtoolset-7-systemtap-3.1-4s.el7](https://cbs.centos.org/koji/buildinfo?buildID=22765)对应的源码版本只有systemtap-3.1。所以只能寻求更高版本的支持。因此在该网站上找到了[devtoolset-8-systemtap-3.3-1.el6](https://cbs.centos.org/koji/buildinfo?buildID=23609)。

![](attachments/Pasted%20image%2020240814155605.png)



# 指定gcc/g++，glibc的版本进行编译
在编译时需要指定gcc及库的依赖路径，包括以下几点：

**（1）指定gcc/g++的版本**
```bash
export CC=gcc的路径
export CXX=g++的路径

```

**(2) 指定连接器的版本**
将连接器的路径，放在`LD_LIBRARY_PATH`的最前面

**(3) 指定glibc的版本**
- 通过gcc 的-L参数指定glibc库(libc.so)的路径
- 在gcc的编译参数中指定 -Wl,–dynamic-linker=glibc中动态链接器的路径，如下：
```bash
-Wl,--dynamic-linker=/动态连接器的路径/ld-linux-x86-64.so.2
```
- 在gcc中链接libc.so(-lc)
- 将`glibc`的路径，引入`LD_LIBRARY_PATH`中.

# 程序运行机器上的依赖
## 打包依赖的so库发布
如果编译环境与运行环境不同，则需要将gcc，glibc的一些库打包到程序安装包中，并且指定库的路径。

**(1)依赖的库 **

`libstdc++.so`,`libc.so`库及它们的依赖库，动态连接器都需要放入程序的依赖库的目录中，基本是包含如下几个库
```bash
librt.so.1
libdl.so.2
libpthread.so.0
libstdc++.so.6
libm.so.6
libc.so.6 -> glibc库
libgcc_s.so.1
libresolv.so.2
libcrypt.so.1
ld-linux-x86-64.so.2 ->其实是个执行程序，为动态连接器
```

 **(2)指定依赖库的路径**
 
在编译时通过gcc的编译参数`-Wl,-rpath=程序的依赖库路径`
这些库最好都放在指定的，固定的目录中，在编译时通过这个编译选项指定该路径

**(3)将动态连接器`ld-linux-x86-64.so.2`的路径配置到`PATH`中**


总结：就是千方百计的，将程序链接/运行时的依赖路径指向期望的版本，手段包括：
- `-Wl,-rpath=`编译参数
- `-Wl,--dynamic-linker`编译参数
- 设置`LD_LIBRARY_PATH`
- 设置`PATH`

# glibc的问题与解决
## 依赖高版本的glibc的问题

![](attachments/Pasted%20image%2020240814120952.png)

如上，安装libbpf，需要高版本的glibc，这个是因为 libbpf中 的 libbpf.so.0，依赖高版本的 glibc 中的函数，最高依赖的是某个函数用到的 GLIBC_2.28 。
如下所示：

![](attachments/Pasted%20image%2020240814160637.png)

## 版本查看
### 查看当前的版本
```bash
gcc --version
ldd --version
rpm --version
make --version
```

### 查看已经安装的glibc版本
```bash
strings /lib64/libc.so.6 | grep ^GLIBC
strings /lib64/libc.so.6 | grep -E "^GLIBC" | sort -V -r | uniq
```
![](attachments/Pasted%20image%2020240814121736.png)

### 查看libstdc++.so 版本
==`libstdc++`是 gcc的标准C++库(`libc++`是clang的标准C++库)；`libstdc++`是被包含在gcc中的，对应为gcc的版本 `gcc --version`==。

```bash
# ldconfig -p | grep stdc++   
libstdc++.so.6 (libc6) => /usr/lib/libstdc++.so.6

# strings /usr/lib/libstdc++.so.6 | grep LIBCXX
```

#### libstdc++ 和 glibc的关系
`libstdc++`与gcc是捆绑在一起的，也就是说安装gcc的时候会把libstdc++装上。 
那为什么glibc和gcc没有捆绑在一起呢？因为程序可以不依赖`libstdc++`，但是必须依赖`glibc`。

#### 确定可执行程序需要的`glibc/libstdc++`的版本
```bash
readelf -s qt_cef_poc | grep -oP "GLIBC_[\d\.]*" | sort | uniq
```

## 安装高版本的glibc
### 首先安装busybox
手动升级高版本的  glibc，可能会导致系统中的各个命令都无法使用，ssh也无法登陆。
那么就提前按照busybox，因为busybox 不依赖任何动态库。可以作为失败时的操作。

### 手动升级高版本的glibc
升级前需查看当前环境的glibc是否存在符合taos3的版本，若存在则跳过升级，此文档假设glibc当前最高的版本为2.17


```bash
# 1.查看glibc函数库版本
strings /lib64/libc.so.6 | grep -E "^GLIBC" | sort -V -r | uniq
# 输出
GLIBC_2.2.5
GLIBC_2.2.6
GLIBC_2.3
GLIBC_2.3.2
GLIBC_2.3.3
GLIBC_2.3.4
GLIBC_2.4
GLIBC_2.5
GLIBC_2.6
GLIBC_2.7
GLIBC_2.8
GLIBC_2.9
GLIBC_2.10
GLIBC_2.11
GLIBC_2.12
GLIBC_2.13
GLIBC_2.14
GLIBC_2.15
GLIBC_2.16
GLIBC_2.17  # 当前最高版本
GLIBC_PRIVATE
 
# 2.下载glibc-2.31安装包
cd /backup
wget https://mirrors.aliyun.com/gnu/glibc/glibc-2.31.tar.gz
 
# 3.进入到解压目录
tar -xf glibc-2.31.tar.gz
cd glibc-2.31
 
# 4.查看安装glibc的前提依赖，对于不满足的依赖需要进行升级，使用yum -y install xxx 升级或安装即可
cat INSTALL | grep -E "newer|later" | grep "*"
# 输出
* GNU 'make' 4.0 or newer
* GCC 6.2 or newer
* GNU 'binutils' 2.25 or later
* GNU 'texinfo' 4.7 or later
* GNU 'bison' 2.7 or later
* GNU 'sed' 3.02 or newer
* Python 3.4 or later
* GDB 7.8 or later with support for Python 2.7/3.4 or later
* GNU 'gettext' 0.10.36 or later
# 假设上述依赖条件已全部满足
 
# 5.建立构建目录，执行编译安装
mkdir build
 
# 6.指定安装到具体的目录下，此示例表示将make安装到/opt下
cd build/
../configure  --prefix=/usr/local/glibc-2.31 --disable-profile --enable-add-ons --with-headers=/usr/include --with-binutils=/usr/bin --disable-sanity-checks --disable-werror
 
# 7.编译安装
make -j4  # 此处时间较长
make install
# 解决新启动远程终端时报一个WARNING
make localedata/install-locales
 
# install结束会出现一个错误，此错误可忽略
# 错误输出
Execution of gcc -B/usr/bin/ failed!
The script has found some problems with your installation!
Please read the FAQ and the README file and check the following:
- Did you change the gcc specs file (necessary after upgrading from
  Linux libc5)?
- Are there any symbolic links of the form libXXX.so to old libraries?
  Links like libm.so -> libm.so.5 (where libm.so.5 is an old library) are wrong,
  libm.so should point to the newly installed glibc file - and there should be
  only one such link (check e.g. /lib and /usr/lib)
You should restart this script from your build directory after you've
fixed all problems!
Btw. the script doesn't work if you're installing GNU libc not as your
primary library!
make[1]: *** [Makefile:120: install] Error 1
make[1]: Leaving directory '/backup/glibc-2.31'
make: *** [Makefile:12: install] Error 2'
 
# 8.安装完成后检查glibc版本
strings /lib64/libc.so.6 | grep -E "^GLIBC" | sort -V -r | uniq
# 输出
GLIBC_PRIVATE
GLIBC_2.30
GLIBC_2.29
GLIBC_2.28
GLIBC_2.27
GLIBC_2.26
GLIBC_2.25
GLIBC_2.24
GLIBC_2.23
GLIBC_2.22
GLIBC_2.18
GLIBC_2.17
GLIBC_2.16
GLIBC_2.15
GLIBC_2.14
GLIBC_2.13
GLIBC_2.12
GLIBC_2.11
GLIBC_2.10
GLIBC_2.9
GLIBC_2.8
GLIBC_2.7
GLIBC_2.6
GLIBC_2.5
GLIBC_2.4
GLIBC_2.3.4
GLIBC_2.3.3
GLIBC_2.3.2
GLIBC_2.3
GLIBC_2.2.6
GLIBC_2.2.5


#9. 使用高版本的glibc
临时运行：
LD_PRELOAD=/usr/local/glibc-2.39/lib/ld-2.30.so ./your_application

临时更改：（可以在一个终端作为测试用）
export LD_LIBRARY_PATH=/usr/local/glibc-2.31/lib:$LD_LIBRARY_PATH

注意：这里环境变量要如上一样临时修改，建议不要直接在 /etc/profile文件里修改，否则会导致某些shell命令执行不了。

永久更改：
cat /etc/ld.so.conf // 查看之前的内容

echo '/usr/local/glibc-2.31/lib' | sudo tee -a /etc/ld.so.conf

ldconfig
```

### 注意
#### 使用高版本glibc的问题
1. 升级glibc存在系统崩溃风险！！！升级前尽可能在个人环境下进行反复测试，确保无问题后再升级生产环境！

2. 当glibc版本为2.17时千万不要直接升级到2.25！！！2.17与2.25直接差4个版本(2.18、2.22、2.23、2.24)，经反复测试确认发现直接升级到2.25时不会自动安装缺失的版本，而2.25又对之前的版本有依赖(个人猜测)，强行安装2.25不但安装失败，且会造成系统崩溃、异常(比如无法使用ls、cp等命令，无法进行远程连接)。

#### 故障处理
**故障现象**：假设在glibc2.17时直接升级到glibc2.25，将会出现操作系统崩溃的情况，如：大部分命令不可用、无法远程登录、yum报错等。

**说明**：出现此类问题时千万不要重启服务器，不要关闭当前的终端！！！因为这时候这个ssh 连接已是唯一还能操作这台服务器的。后续的ssh链接， 可能会由于glibc异常，无法连接上。
> 比如：不是临时更改glibc，而是通过更改文件，永久更改，那么后续的ssh链接就无法连接上。

```bash
# 编译报错输出
make[2]: *** [Makefile:84: da.mo] Segmentation fault (core dumped)
make[2]: Leaving directory '/root/glibc-2.25/po'
make[1]: *** [Makefile:215: po/subdir_install] Error 2
make[1]: Leaving directory '/root/glibc-2.25'
make: *** [Makefile:12: install] Error 2
​
# 命令执行输出错误
[root@centos84 build]# ls
ls: relocation error: /lib64/libpthread.so.0: symbol __libc_dl_error_tsd, version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
​
# yum执行输出
[root@centos84 build]# yum
/usr/bin/python: relocation error: /lib64/libpthread.so.0: symbol __libc_dl_error_tsd, version GLIBC_PRIVATE not defined in file libc.so.6 with link time reference
```

**故障原因**：glibc2.25未编译安装成功，但部分组件依赖的函数库软链接指向到了glibc2.25上。

**解决办法**：
（1）将软链接指向原glibc-2.17 
（2）使用   LD_PRELOAD指定之前的 glibc-2.17的位置。
（3）使用不依赖glibc 动态库的程序，比如 busybox 等等看看是否可以挽救。 因为ldd ./busybox 发现也不依赖glibc。



```bash
# 1.先解决ls不能使用的问题
sln /usr/lib64/libc-2.17.so /lib64/libc.so.6
sln /usr/lib64/ld-2.17.so /usr/lib64/ld-linux-x86-64.so.2
# 再次执行ls时已经恢复正常了
# 上述的方式只是临时恢复了，如果执行ldconfig又会报错，需要执行以下操作进行彻底修复
​
# 2.彻底解决崩溃问题
# 链接指回libc-2.17 删除glibc2.25有关的文件
sln /usr/lib64/libc-2.17.so /lib64/libc.so.6
sln /usr/lib64/ld-2.17.so /usr/lib64/ld-linux-x86-64.so.2
sln /usr/lib64/libdl-2.17.so /usr/lib64/libdl.so.2
# 删除2.25文件
rm -rf /usr/lib64/libc-2.25.so /usr/lib64/ld-2.25.so /usr/lib64/libdl-2.25.so
```

```bash
进入lib64目录执行以下命令：

LD_PRELOAD=/lib64/libc-2.12.so ln -fs /lib64/libc-2.12.so /lib64/libc.so.6  //强制链接
```



# 其他
## 程序依赖高版本的glibc的什么函数
参考：[Linux 修改 ELF 解决 glibc 兼容性问题](https://cloud.tencent.com/developer/article/1758586)

### 问题描述
相信有不少 Linux 用户都碰到过运行第三方（非系统自带软件源）发布的程序时的 glibc 兼容性问题，这一般是由于当前 Linux 系统上的 GNU C 库（glibc）版本比较老导致的，例如我在 CentOS 6 64 位系统上运行某第三方闭源软件时会报：
```bash
[root@centos6-dev ~]# ldd tester
./tester: /lib64/libc.so.6: version `GLIBC_2.17' not found (required by ./tester)
./tester: /lib64/libc.so.6: version `GLIBC_2.14' not found (required by ./tester)
        linux-vdso.so.1 =>  (0x00007ffe795fe000)
        libpthread.so.0 => /lib64/libpthread.so.0 (0x00007fc7d4c73000)
        libOpenCL.so.1 => /usr/lib64/libOpenCL.so.1 (0x00007fc7d4a55000)
        libdl.so.2 => /lib64/libdl.so.2 (0x00007fc7d4851000)
        libm.so.6 => /lib64/libm.so.6 (0x00007fc7d45cd000)
        libgcc_s.so.1 => /lib64/libgcc_s.so.1 (0x00007fc7d43b7000)
        libc.so.6 => /lib64/libc.so.6 (0x00007fc7d4023000)
        /lib64/ld-linux-x86-64.so.2 (0x00007fc7d4e90000)
```

### 查看

首先我们可以检查一下程序使用了新版本 glibc 的哪些符号，使用 objdump 命令可以查看 ELF 文件的动态符号信息：
```bash
[root@centos6-dev ~]# objdump -T tester | grep GLIBC_2.1.*
0000000000000000      DF *UND*  0000000000000000  GLIBC_2.14  memcpy
0000000000000000      DF *UND*  0000000000000000  GLIBC_2.17  clock_gettime
```

```bash
用 readelf -s xxx;  可以查看 c 程序里哪里用到了哪个版本的库函数。

比如：
readelf -s qt_cef_poc  | grep -oP "GLIBC_[\d\.]*" | sort | uniq
GLIBC_
GLIBC_2
GLIBC_2.14
GLIBC_2.2.5
GLIBC_2.4
```

从上面的输出可以看到程序使用了 glibc 2.14 版本的 memcpy 函数和 glibc 2.17 版本的 clock_gettime 函数，而这两个常用的函数按说应该是 glibc 很早就已经支持了的，我们可以确认一下当前系统 glibc 提供的符号版本：

```bash
[root@centos6-dev ~]# objdump -T /lib64/libc.so.6 | grep memcpy
0000000000091300  w   DF .text  0000000000000009  GLIBC_2.2.5 wmemcpy
0000000000101070 g    DF .text  000000000000001b  GLIBC_2.4   __wmemcpy_chk
00000000000896b0 g    DF .text  0000000000000465  GLIBC_2.2.5 memcpy
00000000000896a0 g    DF .text  0000000000000009  GLIBC_2.3.4 __memcpy_chk
[root@centos6-dev ~]# objdump -T /lib64/libc.so.6 | grep clock_gettime
000000000038f800 g    DO .bss   0000000000000008  GLIBC_PRIVATE __vdso_clock_gettime
```
这里可以看出 CentOS 6 的 glibc 库提供的 memcpy 实现是 2.2.5 版本的，另外 libc 没有直接实现 clock_gettime 函数，因为老版本 glibc 里 clock_gettime 是由 librt 库提供 clock_gettime 支持的，而且同样也是 2.2.5 版本：

```bash
[root@centos6-dev ~]# objdump -T /lib64/librt.so.1 | grep clock_gettime
0000000000000000      DO *UND*  0000000000000000  GLIBC_PRIVATE __vdso_clock_gettime
0000000000003e70 g    DF .text  000000000000008b  GLIBC_2.2.5 clock_gettime
```

**问题小结**
看过这里就基本明白了，第三方程序的开发者是**在新版本 glibc 的 Linux 系统上编译**的，memcpy 和 clock_gettime 的实现默认使用了该系统上 glibc 所提供的最新版本，这样**在低版本 glibc 系统中就无法正常运行**。

## 高版本的 rpm
## 高版本的make
### 手动安装高版本make
```bash
# 1.安装依赖
yum -y install gcc gcc+
 
# 2.建立安装包存放目录
mkdir /backup
 
# 3.下载make安装包
cd /backup
wget https://mirrors.aliyun.com/gnu/make/make-4.3.tar.gz
 
# 4.解压压缩包并建立构建目录
tar -xf make-4.3.tar.gz
cd make-4.3
mkdir build
cd build
 
# 5.指定安装到具体的目录下，此示例表示将make安装到/opt下
../configure --prefix=/opt/make
 
# 6.编译安装
make && make install
 
# 7.建立软连接
ln -sf /opt/make/bin/make /usr/bin/make
 
# 8.检查make版本
make --version
# 输出
GNU Make 4.3
Built for x86_64-pc-linux-gnu
Copyright (C) 1988-2020 Free Software Foundation, Inc.
License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.
```

# 参考
```bash
# centos7升级glibc2.25避坑指南
https://blog.csdn.net/SerMa/article/details/131226445

# GLIBC 版本低的临时解决办法
https://mp.weixin.qq.com/s/G27AZdDt1MWWqjaIvJLK6w
```