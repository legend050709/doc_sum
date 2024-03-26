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

（1）安装pip3环境

```text
yum install python3 python3-pip
pip3 install --upgrade pip
```

(2) 安装meson

```text
 pip3 install meson
```

(3)安装ninja

```text
pip3 install ninja

注意如果出现WARNING: The script ninja is installed in '/usr/local/bin' which is not on PATH.  
则需要添加环境变量  
vim /etc/profile  
文件末尾添加export PATH="/usr/local/bin:$PATH"
```

（4）网卡驱动安装

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

## 安装Mellanox网卡驱动

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
解压缩后进入文件夹执行
```text
meson build
cd build
ninja
ninja install
ldconfig
```

## 编译kmod
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
cd build ninja
ninja install
ldconfig
```


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
**（1）查看CPU的微架构**
英特尔官网地址：[http://ark.intel.com/](https://link.zhihu.com/?target=http%3A//ark.intel.com/)，然后输入我们的CPU型号查询
![](attachments/Pasted%20image%2020240320171952.png)
![](attachments/Pasted%20image%2020240320172152.png)
对应的平台全称则为:`broadwell`

下图为`intel`各平台的微架构(march: micro architecture)全称：
![](attachments/Pasted%20image%2020240320172231.png)
![](attachments/Pasted%20image%2020240321112010.png)


![](attachments/Pasted%20image%2020240321110712.png)
参考：[intel cpu 微架构介绍](https://en.wikichip.org/wiki/intel/microarchitectures)


**（2）查看CPU的架构**
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


**（3）CPU的微架构和指令集**
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

**注意GCC版本**
注意GCC版本，以及是否能够支持目标机的指令集，一般GCC版本使用高版本编译都能兼容。

# 参考
```bash
# DPDK 21.11.1版本的交叉编译
https://zhuanlan.zhihu.com/p/643562657

# 编译安装DPDK21.11.1
https://zhuanlan.zhihu.com/p/537309977
```