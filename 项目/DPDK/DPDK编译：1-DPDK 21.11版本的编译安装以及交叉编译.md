```table-of-contents
```

# 版本信息

```bash
DPDK版本：21.11.1

内核版本：3.10.0

操作系统：CentOS Linux release 7.2.1511 (Core)

网卡驱动：MLNX_OFED_LINUX-5.6-2.0.9.0-rhel7.2-x86_64
```

# 安装依赖工具和驱动

DPDK在20版本以后需要使用`meson`和`ninja`编译安装。

## 安装pip3环境

```text
yum install python3 python3-pip
pip3 install --upgrade pip
```

## 安装meson

```text
 pip3 install meson
```

## 安装ninja

```text
pip3 install ninja

注意如果出现WARNING: The script ninja is installed in '/usr/local/bin' which is not on PATH.  
则需要添加环境变量  
vim /etc/profile  
文件末尾添加export PATH="/usr/local/bin:$PATH"
```

## 网卡驱动安装

Mellanox 网卡驱动

进入 Mellanox 官网：[https://cn.mellanox.com/](https://link.zhihu.com/?target=https%3A//cn.mellanox.com/)

打开“以太网驱动程序”支持
![](attachments/Pasted%20image%2020240320144253.png)

选择OFED驱动
> 注意：不能安装`Mellanox EN` 驱动，否则会在编译 DPDK 时出错，提示找不到 `<infiniband/verbs.h>`

![](attachments/Pasted%20image%2020240320144321.png)


在download选择对应操作系统`CentosOS 7.2`版本的驱动
![](attachments/Pasted%20image%2020240320144339.png)

下载得到文件：`MLNX_OFED_LINUX-5.6-2.0.9.0-rhel7.2-x86_64.tgz`；然后解压缩文件

### 安装Mellanox网卡驱动

```text
./mlnxofedinstall --upstream-libs --dpdk --add-kernel-support

其中 `--add-kernel-support` 用于解决驱动程序与当前 CentOS 系统 Kernel 版本不匹配的问题
```

安装时间较长，执行结束后根据提示在终端运行：

```text
dracut -f

/etc/init.d/openibd restart
```

完成后重启主机
```text
reboot
```


# DPDK源码下载与编译安装
在DPDK官网选择下载页面下载
![](attachments/Pasted%20image%2020240320145133.png)

选择版本`DPDK 21.11.1`；下载得到文件：`dpdk-21.11.1.tar.xz`；
![](attachments/Pasted%20image%2020241209104111.png)

解压缩后进入文件夹执行
```text
// 配置DPDK的构建环境
meson setup <options> build
比如：
meson setup build

// 编译dpdk并安装
cd build
[FIB nexthop Exception下一跳例外理解](../../网络/路由相关/FIB%20nexthop%20Exception下一跳例外理解.md)ninja
ninja install
ldconfig
```
参考：[#  Compiling the DPDK21.11 Target from Source](https://doc.dpdk.org/guides-21.11/linux_gsg/build_dpdk.html)


## 配置(config)dpdk的构建(build)环境
要配置一个DPDK构建，请使用以下命令：
```bash
meson setup <options> build
```
其中`"build"`是所需的输出构建目录，`"<options>"`可以为空或是一些`meson`的构建选项或`DPDK`特定的构建选项。
配置结束（configuration finish）后，将汇总要编译(build)和安装(install)的DPDK库和驱动程序，并对每个被禁用的项目提供禁用的原因。例如，可以使用这些信息来识别任何缺失的驱动所需的软件包。



### 调整构建选项(build options)
DPDK有许多选项可以在构建配置（build config）过程中进行调整。可以通过在配置的构建文件夹中运行`meson configure`命令来列出这些选项。

#### meson configure 查看选项
```text
[root@6f08e26157fa dpdk-22.11]# meson configure

Core properties:
  Source dir /dpdk-22.11

Main project options:

  Core options                   Default Value                    Possible Values                  Description
  ------------                   -------------                    ---------------                  -----------
  auto_features                  auto                             [enabled, disabled, auto]        Override value of all 'auto' features
  backend                        ninja                            [ninja, vs, vs2010, vs2012,      Backend to use
                                                                  vs2013, vs2015, vs2017, vs2019,
                                                                   vs2022, xcode]
  buildtype                      release                          [plain, debug, debugoptimized,   Build type to use
                                                                  release, minsize, custom]
  cmake_prefix_path              []                                                                List of additional prefixes for cmake to search
  debug                          false                            [true, false]                    Enable debug symbols and other information
  default_library                static                           [shared, static, both]           Default library type
  force_fallback_for             []                                                                Force fallback for those subprojects
  install_umask                  0022                             [preserve, 0000-0777]            Default umask to apply on permissions of installed files
  layout                         mirror                           [mirror, flat]                   Build directory layout
  optimization                   3                                [0, g, 1, 2, 3, s]               Optimization level
  pkg_config_path                []                                                                List of additional paths for pkg-config to search
  strip                          false                            [true, false]                    Strip targets on install
  unity                          off                              [on, off, subprojects]           Unity build
  unity_size                     4                                >=2                              Unity block size
  warning_level                  2                                [0, 1, 2, 3]                     Compiler warning level to use
  werror                         false                            [true, false]                    Treat warnings as errors
  wrap_mode                      default                          [default, nofallback,            Wrap mode
                                                                  nodownload, forcefallback,
                                                                  nopromote]
  build.cmake_prefix_path        []                                                                List of additional prefixes for cmake to search
  build.pkg_config_path          []                                                                List of additional paths for pkg-config to search

  Backend options                Default Value                    Possible Values                  Description
  ---------------                -------------                    ---------------                  -----------
  backend_max_links              0                                >=0                              Maximum number of linker processes to run or 0 for no limit

  Base options                   Default Value                    Possible Values                  Description
  ------------                   -------------                    ---------------                  -----------
  b_asneeded                     true                             [true, false]                    Use -Wl,--as-needed when linking
  b_colorout                     always                           [auto, always, never]            Use colored output
  b_coverage                     false                            [true, false]                    Enable coverage tracking.
  b_lto                          false                            [true, false]                    Use link time optimization
  b_lto_threads                  0                                                                 Use multiple threads for Link Time Optimization
  b_lundef                       true                             [true, false]                    Use -Wl,--no-undefined when linking
  b_ndebug                       false                            [true, false, if-release]        Disable asserts
  b_pch                          true                             [true, false]                    Use precompiled headers
  b_pgo                          off                              [off, generate, use]             Use profile guided optimization
  b_pie                          false                            [true, false]                    Build executables as position independent
  b_sanitize                     none                             [none, address, thread,          Code sanitizer to use
                                                                  undefined, memory,
                                                                  address,undefined]
  b_staticpic                    true                             [true, false]                    Build static libraries as position independent

  Compiler options               Default Value                    Possible Values                  Description
  ----------------               -------------                    ---------------                  -----------
  c_args                         []                                                                Extra arguments passed to the c compiler
  c_link_args                    []                                                                Extra arguments passed to the c linker
  c_std                          none                             [none, c89, c99, c11, gnu89,     C language standard to use
                                                                  gnu99, gnu11]
  build.c_args                   []                                                                Extra arguments passed to the c compiler
  build.c_link_args              []                                                                Extra arguments passed to the c linker
  build.c_std                    none                             [none, c89, c99, c11, gnu89,     C language standard to use
                                                                  gnu99, gnu11]

  python module options          Default Value                    Possible Values                  Description
  ---------------------          -------------                    ---------------                  -----------
  python.platlibdir                                                                                Directory for site-specific, platform-specific files.
  python.purelibdir                                                                                Directory for site-specific, non-platform-specific files.

  Directories                    Default Value                    Possible Values                  Description
  -----------                    -------------                    ---------------                  -----------
  bindir                         bin                                                               Executable directory
  datadir                        share                                                             Data file directory
  includedir                     include                                                           Header file directory
  infodir                        share/info                                                        Info page directory
  libdir                         lib64                                                             Library directory
  libexecdir                     libexec                                                           Library executable directory
  localedir                      share/locale                                                      Locale data directory
  localstatedir                  /var/local                                                        Localstate data directory
  mandir                         share/man                                                         Manual page directory
  prefix                         /usr/local                                                        Installation prefix
  sbindir                        sbin                                                              System executable directory
  sharedstatedir                 /var/local/lib                                                    Architecture-independent data directory
  sysconfdir                     etc                                                               Sysconf data directory

  Testing options                Default Value                    Possible Values                  Description
  ---------------                -------------                    ---------------                  -----------
  errorlogs                      true                             [true, false]                    Whether to print the logs from failing tests
  stdsplit                       true                             [true, false]                    Split stdout and stderr in test logs

  Project options                Default Value                    Possible Values                  Description
  ---------------                -------------                    ---------------                  -----------
  check_includes                 false                            [true, false]                    build "chkincs" to verify each header file can compile alone
  cpu_instruction_set            auto                                                              Set the target machine ISA (instruction set architecture). Will
                                                                                                   be set according to the platform option by default.
  developer_mode                 auto                             [enabled, disabled, auto]        turn on additional build checks relevant for DPDK developers
  disable_apps                                                                                     Comma-separated list of apps to explicitly disable.
  disable_drivers                                                                                  Comma-separated list of drivers to explicitly disable.
  disable_libs                   flow_classify,kni                                                 Comma-separated list of libraries to explicitly disable. [NOTE:
                                                                                                   not all libs can be disabled]
  drivers_install_subdir         dpdk/pmds-<VERSION>                                               Subdirectory of libdir where to install PMDs. Defaults to using
                                                                                                   a versioned subdirectory.
  enable_apps                                                                                      Comma-separated list of apps to build. If unspecified, build all
                                                                                                   apps.
  enable_docs                    false                            [true, false]                    build documentation
  enable_driver_sdk              false                            [true, false]                    Install headers to build drivers.
  enable_drivers                                                                                   Comma-separated list of drivers to build. If unspecified, build
                                                                                                   all drivers.
  enable_iova_as_pa              true                             [true, false]                    Support for IOVA as physical address. Disabling removes the
                                                                                                   buf_iova field of mbuf.
  enable_kmods                   false                            [true, false]                    build kernel modules
  enable_trace_fp                false                            [true, false]                    enable fast path trace points.
  examples                                                                                         Comma-separated list of examples to build by default
  ibverbs_link                   shared                           [static, shared, dlopen]         Linkage method (static/shared/dlopen) for NVIDIA PMDs with
                                                                                                   ibverbs dependencies.
  include_subdir_arch                                                                              subdirectory where to install arch-dependent headers
  kernel_dir                                                                                       Path to the kernel for building kernel modules. Headers must be
                                                                                                   in $kernel_dir or $kernel_dir/build. Modules will be installed
                                                                                                   in /lib/modules.
  machine                        auto                                                              Alias of cpu_instruction_set.
  max_ethports                   32                                                                maximum number of Ethernet devices
  max_lcores                     default                                                           Set maximum number of cores/threads supported by EAL; "default"
                                                                                                   is different per-arch, "detect" detects the number of cores on
                                                                                                   the build machine.
  max_numa_nodes                 default                                                           Set the highest NUMA node supported by EAL; "default" is
                                                                                                   different per-arch, "detect" detects the highest NUMA node on
                                                                                                   the build machine.
  mbuf_refcnt_atomic             true                             [true, false]                    Atomically access the mbuf refcnt.
  platform                       native                                                            Platform to build, either "native", "generic" or a SoC. Please
                                                                                                   refer to the Linux build guide for more information.
  tests                          true                             [true, false]                    build unit tests
  use_hpet                       false                            [true, false]                    use HPET timer in EAL
```

如上所示：`enable_kmods` 默认值为`false`， `disable_libs`的默认值为 ` flow_classify,kni`。

```
范例如下：
$ cd dpdk
$ mkdir build
$ meson setup --prefix=/usr -D platform=generic -D cpu_instruction_set=corei7 -D disable_libs=flow_classify -D enable_driver_sdk=true -D disable_drivers=net/mlx4 -D tests=false build
$ meson compile -C build
```

#### 调整buildtype
可用的 `build type` 有 `[plain, debug, debugoptimized, release, minsize, custom]`。默认为 `release`。

更改`buildtype` 为 `debug` 的方式：
（1）方式一：
通过`-Dbuildtype=debug`或`--buildtype=debug`参数传递给`meson`。

（2）方式二：
 在初始的`meson`运行之后，在构建文件夹中运行`meson configure -Dbuildtype=debug`命令。


#### 开启 lto 编译优化

![](attachments/Pasted%20image%2020250420190944.png)


使用范例，如下所示：
```bash
DPDK_VERSION=22.11;  mkdir -p dpdkbuild ; mkdir -p /usr/local/lib/dpdklib-${DPDK_VERSION}-nodebug-with-lto
meson -Ddisable_libs= -Denable_kmods=true -Db_lto=true -Dprefix=/usr/local/lib/dpdklib-${DPDK_VERSION}-nodebug-with-lto dpdkbuild
cd dpdkbuild
ninja -j8
ninja install
```


#### 调整构建的平台(platform)以及指令集

- `-Dplatform=native` 将根据构建机器进行配置。
- `-Dplatform=generic` 将使用适用于与构建机器相同体系结构的所有机器的配置。
- `-Dplatform=<SoC>` 将使用针对特定SoC进行优化的配置。请参考`config/arm/meson.build`中的"socs"字典以查看支持的SoC。


默认情况下，指令集将根据以下规则自动设置：
- `-Dplatform=native` 将`cpu_instruction_set`设置为`native`，从而将`-march`（x86_64）、`-mcpu`（ppc）、`-mtune`（ppc）配置为本机。

- `-Dplatform=generic` 将`cpu_instruction_set`设置为`generic`，从而将`-march`（x86_64）、`-mcpu`（ppc）、`-mtune`（ppc）配置为DPDK所需的通用最小基线。

要覆盖将使用的指令集，请将`cpu_instruction_set`参数设置为您选择的指令集（例如corei7、power8等）。
在Arm构建中不使用`cpu_instruction_set`，因为仅设置指令集而不设置其他参数会导致较差的构建结果。定制Arm构建的方法是使用上述提到的`-Dplatform=<SoC>`来构建特定SoC的版本。


### 在64位操作系统上构建32位DPDK程序
在64位操作系统上构建32位DPDK程序，需要向编译器和链接器传递`-m32`标志，以强制生成32位的目标文件和可执行文件。
可以通过在环境中设置`CFLAGS`和`LDFLAGS`，或者通过使用`-Dc_args=-m32`和`-Dc_link_args=-m32`参数将值传递给meson来实现。
为了正确识别和使用任何依赖包，还必须配置`pkg-config`工具，使其在适当的目录中查找32位库的`.pc`文件。可以通过将`PKG_CONFIG_LIBDIR`设置为适当的路径来完成这一步骤。
```bash
PKG_CONFIG_LIBDIR=/usr/lib/i386-linux-gnu/pkgconfig \
    meson setup -Dc_args='-m32' -Dc_link_args='-m32' build
```
一旦配置了构建目录，就可以像上面描述的那样使用ninja编译DPDK。


## 构建(build)以及安装(install)dpdk
配置完成后，要进行构建并将DPDK安装到整个系统中，请使用以下命令：
```bash
cd build
ninja
ninja install # root用户执行，将构建的对象复制到其最终的系统范围位置。
ldconfig # root用户执行，更新程序执行时查询动态库的缓存
```


## 编译kmod：igb_uio
`igb_uio` 模块从主线DPDK代码仓库中移除了，放入到了`dpdk-kmods`代码仓库中。

如果需要安装`igb_uio`；获取源码:
```text
git clone http://dpdk.org/git/dpdk-kmods
cp -r ./dpdk-kmods/linux/igb_uio ./dpdk/kernel/linux/ 
# Copy dpdk-kmods/linux/igb_uio/ to dpdk/kernel/linux/
```
配置文件
`vim dpdk/kernel/linux/meson.build`
将`subdirs = ['kni']` 修改为 `subdirs = ['kni', 'igb_uio']`
![](attachments/Pasted%20image%2020240320145311.png)

`vim dpdk/kernel/linux/igb_uio/meson.build`
加入如下内容
```text
# SPDX-License-Identifier: BSD-3-Clause
# Copyright(c) 2017 Intel Corporation
mkfile = custom_target('igb_uio_makefile',
output: 'Makefile',
command: ['touch', '@OUTPUT@'])
custom_target('igb_uio',
input: ['igb_uio.c', 'Kbuild'],
output: 'igb_uio.ko',
command: ['make', '-C', kernel_dir + '/build',
'M=' + meson.current_build_dir(),
'src=' + meson.current_source_dir(),
'EXTRA_CFLAGS=-I' + meson.current_source_dir() +
'/../../../lib/librte_eal/include',
'modules'],
depends: mkfile,
install: true,
install_dir: kernel_dir + '/extra/dpdk',
build_by_default: get_option('enable_kmods'))
```

重新编译：
```bash
meson build --reconfigure
cd build 
ninja
ninja install
ldconfig
```

## 使用已安装的DPDK构建应用程序
当DPDK在系统范围内安装时，它提供了一个pkg-config文件`libdpdk.pc`，供应用程序在构建过程中查询。建议使用pkg-config文件，而不是将DPDK的参数（cflags/ldflags）硬编码到应用程序的构建过程中。
在DPDK附带的每个示例应用程序的Makefile中可以找到一个查询和使用pkg-config文件的示例。下面是一个简化的示例片段：其中目标二进制文件名存储在变量`(APP)`中，构建的源代码存储在`(APP)`中，构建的源代码存储在`(SRCS-y)`中。
```bash
PKGCONF = pkg-config

CFLAGS += -O3 $(shell $(PKGCONF) --cflags libdpdk)
LDFLAGS += $(shell $(PKGCONF) --libs libdpdk)

$(APP): $(SRCS-y) Makefile
        $(CC) $(CFLAGS) $(SRCS-y) -o $@ $(LDFLAGS)
```
下面是使用meson作为构建系统构建简单DPDK应用程序的等效构建示例：
```bash
project('dpdk-app', 'c')

dpdk = dependency('libdpdk')
sources = files('main.c')
executable('dpdk-app', sources, dependencies: dpdk)
```

## 编译范例

### 配置编译环境
```bash
DPDK_VERSION=22.11
wget https://fast.dpdk.org/rel/dpdk-${DPDK_VERSION}.tar.xz
tar xvf dpdk-${DPDK_VERSION}.tar.xz >/dev/null
cd dpdk-${DPDK_VERSION}
rm -rf dpdkbuild
mkdir -p dpdkbuild
mkdir -p /usr/local/lib/dpdklib-${DPDK_VERSION}
CFLAGS=-std=c11 meson -Denable_kmods=true -Dprefix=/usr/local/lib/dpdklib-${DPDK_VERSION} dpdkbuild 
```

此时输出如下所示：
![](attachments/Pasted%20image%2020241209143904.png)

如上所示，编译的产出中不会编译KNI模块。如果想要编译KNI，则使用下面的命令。

```bash
meson -Denable_kmods=true -Ddisable_libs= -Dprefix=/usr/local/lib/dpdklib-${DPDK_VERSION} dpdkbuild
```

### 编译和安装

```bash
cd dpdkbuild
ninja -j8 // 多线程编译
ninja install // 将编译产生放入到指定的位置

# 设置pkg-config查找pc文件的目录
echo "export PKG_CONFIG_PATH=$PKG_CONFIG_PATH:/usr/lib/pkgconfig:/usr/lib64/pkgconfig:/usr/local/lib64/pkgconfig:/usr/local/lib/pkgconfig:/usr/local/lib/dpdklib-${DPDK_VERSION}/lib64/pkgconfig/" >> /etc/bashrc

source  /etc/bashrc

pkg-config --libs libdpdk // 查看库文件
pkg-config --cflags libdpdk //查看头文件
```

![](attachments/Pasted%20image%2020241209151549.png)


## 其他
### DPDK22.11的目录结构

![](attachments/Pasted%20image%2020241209105104.png)


# 交叉编译
## 前言
在DPDK使用meson管理后相对之前的编译方法已经变的简单和清晰了，为此我们简单介绍一下如何进行給21.11.1版本的交叉编译。

## 交叉编译DPDK库
meson提供了一个支持不同平台的编译的参数:

```text
meson build -Dcpu_instruction_set=generic
```

`generic`我们都知道是本地编译的意思，但是有时候我们需要将编译出来的程序在不同平台运行，所以以英特尔的平台为例:

### 查询CPU型号
使用`lscpu`查询`CPU`型号:
```text
# lscpu
Architecture:          x86_64
CPU op-mode(s):        32-bit, 64-bit
Byte Order:            Little Endian
CPU(s):                24
On-line CPU(s) list:   0-23
Thread(s) per core:    1
Core(s) per socket:    12
Socket(s):             2
NUMA node(s):          2
Vendor ID:             GenuineIntel
CPU family:            6
Model:                 79
Model name:            Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz
Stepping:              1
CPU MHz:               2499.975
CPU max MHz:           2500.0000
CPU min MHz:           1200.0000
BogoMIPS:              4399.96
Virtualization:        VT-x
L1d cache:             32K
L1i cache:             32K
L2 cache:              256K
L3 cache:              30720K
NUMA node0 CPU(s):     0,2,4,6,8,10,12,14,16,18,20,22
NUMA node1 CPU(s):     1,3,5,7,9,11,13,15,17,19,21,23
Flags:                 fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca cmov pat pse36 clflush dts acpi mmx fxsr sse sse2 ss ht tm pbe syscall nx pdpe1gb rdtscp lm constant_tsc arch_perfmon pebs bts rep_good nopl xtopology nonstop_tsc aperfmperf eagerfpu pni pclmulqdq dtes64 monitor ds_cpl vmx smx est tm2 ssse3 sdbg fma cx16 xtpr pdcm pcid dca sse4_1 sse4_2 x2apic movbe popcnt tsc_deadline_timer aes xsave avx f16c rdrand lahf_lm abm 3dnowprefetch epb cat_l3 cdp_l3 intel_pt ibrs ibpb stibp tpr_shadow vnmi flexpriority ept vpid fsgsbase tsc_adjust bmi1 hle avx2 smep bmi2 erms invpcid rtm cqm rdt_a rdseed adx smap xsaveopt cqm_llc cqm_occup_llc cqm_mbm_total cqm_mbm_local dtherm ida arat pln pts spec_ctrl intel_stibp
```

能看到CPU型号是：`Intel(R) Xeon(R) CPU E5-2650 v4 @ 2.20GHz`

### 英特尔官网查询平台名称
#### 查看CPU的微架构
英特尔官网地址：[http://ark.intel.com/](https://link.zhihu.com/?target=http%3A//ark.intel.com/)，然后输入我们的CPU型号查询
![](attachments/Pasted%20image%2020240320171952.png)
![](attachments/Pasted%20image%2020240320172152.png)
对应的平台全称则为:`broadwell`

下图为`intel`各平台的微架构(march: micro architecture)全称：
![](attachments/Pasted%20image%2020240320172231.png)
![](attachments/Pasted%20image%2020240321112010.png)


![](attachments/Pasted%20image%2020240321110712.png)
参考：[intel cpu 微架构介绍](https://en.wikichip.org/wiki/intel/microarchitectures)


#### 查看CPU的架构
查看当前机器的CPU的架构：
```bash
lfp@legion:~$ hostnamectl
   Static hostname: legion
         Icon name: computer-laptop
           Chassis: laptop
        Machine ID: b28d62113b6242exxxxxxxxxxxxxxxxxxx2
           Boot ID: b387c673b99xxxxxxxxxxf3a46a24
  Operating System: Ubuntu 18.04.4 LTS
            Kernel: Linux 5.3.0-51-generic
      Architecture: x86-64


lfp@legion:~$ arch
x86_64


lfp@legion:~$ uname -p # processor type 处理器类型
x86_64
```
![](attachments/Pasted%20image%2020240321111102.png)

CPU的架构的分类：
`Complex Instruction Set Computing；CISC）`
`Reduced Instruction Set Computing，RISC）`
![](attachments/Pasted%20image%2020240321111342.png)


#### CPU的微架构和指令集
CPU的不同微架构通常支持不同的指令集，或者在原有的指令集基础上添加了新的扩展指令集。

指令集是CPU理解和执行的二进制指令的集合，它是CPU设计的核心组成部分，决定了处理器能执行哪些操作和如何执行。

例如，Intel从早期的8086开始发展了多种指令集，包括：

- **x86**: 最初的16位指令集，后来扩展到了32位（x86-32）。
- **x86-64 (AMD64, Intel 64)**: 由AMD开发并首先引入的64位扩展，后来被Intel采纳，现在是大多数现代桌面和服务器CPU的标准。
- **MMX**: 多媒体扩展指令集，增加了处理多媒体数据的能力。
- **SSE (Streaming SIMD Extensions)**: 提供了单指令多数据（SIMD）能力，用于浮点运算和多媒体处理。
- **SSE2, SSE3, SSSE3, SSE4.1, SSE4.2**: 后续的SIMD扩展，增加了更多功能和性能提升。
- **AVX (Advanced Vector Extensions)**: 扩展了SIMD寄存器宽度和指令集，提高了向量计算性能。
- **AVX2, AVX-512**: 更进一步的向量计算扩展，AVX-512尤其在高性能计算和数据中心应用中常见。


> 因此：不同的微架构会支持不同级别的这些指令集，较新的微架构通常会支持更多的指令集和扩展。这意味着较新的CPU在处理某些特定任务时可能会有更高的效率或性能。软件开发者可以根据目标平台选择性地使用这些指令集来优化、编译他们的代码。**如果目标运行平台的CPU的微架构比较低级，在编译代码的时候，就要考虑进行适配，选择较低的微架构(march)进行编译**。

### 编译DPDK时使用平台名称编译
```text
meson build -Denable_kmods=true -Dcpu_instruction_set=broadwell
```

##  交叉编译DPDK应用程序

### 指定目标微架构进行编译
如果你的目标体系结构是`x86_64`，并且想要利用`AVX2`指令集，你不需要进行交叉编译，因为AVX2是现代`x86_64`处理器支持的指令集。
在`Makefile`中，你可以这样设置`CFLAGS`来启用`AVX2`：
```bash
# 定义源文件和目标文件
SOURCES := $(wildcard src/*.c)
OBJECTS := $(patsubst %.c,%.o,$(SOURCES))
EXECUTABLE := my_program

# 设置编译器和编译选项
CC := gcc
CFLAGS := -march=haswell -O3 -Wall -std=c11

# 如果需要链接其他库，可以在这里添加-L和-l选项
# LDFLAGS := 

# 编译规则
%.o: %.c
    $(CC) $(CFLAGS) -c $< -o $@

# 链接规则
all: $(OBJECTS)
    $(CC) $(CFLAGS) $(LDFLAGS) $(OBJECTS) -o $(EXECUTABLE)

# 清理规则
clean:
    rm -f *.o $(EXECUTABLE)
```
请注意，这个`Makefile`假设你的开发环境是`x86_64`，并且你的编译器（`gcc`）能够识别并正确处理`AVX2`指令。**在运行`make`之前，请确保你的开发环境(GCC)支持`AVX2`，并且目标机器也具有支持`AVX2的CPU`**。
```bash
查看GCC编译器针对当前硬件平台的-march=native会具体选择哪个CPU架构;

gcc -march=native -Q --help=target | grep march
注：同一个机器上，一个在宿主机，一个在容器中，不同的GCC版本，输出的结果可能不同。

```


**解析**：
`-march=haswell `: 指定了`AVX2`指令集，因为`Haswell`是第一个广泛支持`AVX2`的`Intel`处理器家族。如果你知道目标机器的处理器更先进，可以选择更新的架构，比如`skylake`或`broadwell`。

> 注：可基于目标运行机器的CPU型号的`微架构(march)`，在Intel官方[http://ark.intel.com/](https://link.zhihu.com/?target=http%3A//ark.intel.com/)，然后输入我们的CPU型号查询，查询对应的微架构。


## 注意事项

在编译程序时，需要注意目标机的驱动版本，因为`DPDK`默认是应用层驱动都编译。
如果目标机网卡没有安装指定的驱动，需要将默认编译的驱动去掉。比如：
```text
meson build -Ddisable_drivers=net/mlx5,crypto/mlx5,compress/mlx5,vdpa/mlx5,regex/mlx5,common/mlx5 -Denable_kmods=true -Dcpu_instruction_set=broadwell
```

### 注意GCC版本
注意GCC版本，以及是否能够支持目标机的指令集，一般GCC版本使用高版本编译都能兼容。

# 参考
```bash
# DPDK 21.11.1版本的交叉编译
https://zhuanlan.zhihu.com/p/643562657

# 编译安装DPDK21.11.1
https://zhuanlan.zhihu.com/p/537309977
```