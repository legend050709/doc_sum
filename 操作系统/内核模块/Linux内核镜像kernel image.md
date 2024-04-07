```table-of-contents
```
# Linux内核镜像
## 内核镜像的介绍
Linux内核镜像是内核代码经过编译后生成的一个二进制文件。这个镜像文件通常是在系统引导(boot)时加载到内存中的，以启动操作系统。

在Linux系统中，内核是运行在更高权限的进程(**可以将内核看成一个进程**)，可以访问所有硬件资源，掌控整个系统的运行状态。内核映像是内核代码编译后的二进制文件，包含了内核的所有功能和特性。在启动Linux系统时，内核映像会更先被加载，然后按照一定的规则初始化硬件和文件系统等基本系统资源，从而启动整个系统。

Linux内核镜像通常包含了操作系统的一些基本功能，例如内存管理、进程管理、设备驱动等。在内核镜像之上，操作系统会进一步加载其他的用户空间程序和库，例如shell、应用程序等。

## 内核镜像的生成
Linux内核镜像(kernel image)的生成需要**对Linux内核源码进行编译**和构建。通常情况下，开发人员可以从Linux官方网站下载源代码，然后通过交叉编译工具链来生成内核镜像。在生成过程中，可以根据实际需要对内核进行配置和优化，以满足特定的需求和场景。

Linux内核镜像通常具有以下文件名格式：`vmlinuz-x.x.x`。其中，x.x.x表示内核版本号(uname -r 的输出格式)。它可以直接用来启动Linux系统。

`vmlinuz`是可引导的，经过压缩的linux内核镜像(`vmlinuz` 中的 vm表示虚拟内存，z表示`bzip`压缩)。`vmlinuz`是`vmlinux`经过`gzip`和`objcopy`制作出来的压缩文件。

**vmlinux介绍**
```
vmlinux（“vm”代表的“virtual memory”）是一个包括linux kernel的静态链接的可运行文件，编译内核源码得到的最原始的内核文件，未压缩，比较大，是elf格式的文件。
```

## 内核镜像文件位置
通常情况下，Linux内核镜像被放置在/boot目录下，可以通过bootloader（如GRUB）来加载。
要查看当前正在使用的内核版本，可以使用命令`uname -r`。要查看可用的内核镜像文件列表，可以在`/boot`目录中使用命令`ls vmlinuz*`或`ls bzImage*`。


# Linux内核映像的作用和重要性
## 系统启动的关键组件
作为系统的启动程序，内核映像是非常重要的。在系统开机时，BIOS首先会读取硬盘上的MBR（Master Boot Record）扇区，然后读取GRUB，GRUB再启动内核映像，内核映像负责初始化整个系统，包括开机自检、硬件初始化、文件系统加载等。整个系统启动过程中，内核映像一直处于更高权限，掌控着整个系统的运行状态，所以它必须极其可靠。

## 内核功能扩展的载体
内核是系统的核心，也是更底层的组件。其他系统组件如文件系统、网络协议栈等都依赖于它的支持。
**内核是一个插件式结构，通过加载不同的内核模块可以实现不同的功能和特性**。内核映像除了包含内核的基本功能外，还可以通过加载内核模块的方式扩展这些功能。这些内核模块可以增加新的网络功能、文件系统支持、驱动程序以及各种硬件设备的支持等。

# Linux内核映像的优化方法
## 压缩内核映像
Linux内核映像可以采用压缩的方式来缩小其体积，这对于资源受限的嵌入式系统十分重要。不同的压缩方式有不同的优势和劣势，根据不同的设备特性选择不同的压缩方式进行优化。

## 编译内核
Linux内核可以通过编译的方式来优化其性能。编译内核是指根据不同硬件环境和需求重新编译内核映像，去除不必要的组件并加入所需要的组件，以达到优化性能的目的。这种方式需要对内核代码有一定的了解和操作经验

## 加载必要的内核模块

Linux内核是一个插件式结构，通过加载不同的内核模块可以实现不同的功能和特性，但同时也会增加内存和系统开销。为了提升系统性能，建议只加载必要的内核模块，避免过多的内存和系统资源的浪费。

# Linux Kernel 镜像格式
Linux 内核有多种格式的镜像，例如 vmlinux、Image、zImage、bzImage、uImage、xipImage、bootpImage 等。

## vmlinux
vmlinux 是可引导的、未压缩、可压缩的内核镜像，vm 代表**Virtual Memory**，Linux 支持虚拟内存，因此得名 vm。它是由用户对内核源码编译得到，实质是 ELF 格式的文件，也就是说vmlinux 是编译出来的最原始的内核文件，未被压缩过。

## Image
Image是经过 **objcopy** 处理的只包含二进制数据的内核代码，它已经不是 ELF 格式了，但这种格式的内核镜像还没有经过压缩。

objcopy 工具的作用是拷贝一个目标文件的内容到另一个目标文件中，也就是说，可以将一种格式的目标文件转换成另一种格式的目标文件。通过使用 binary 作为输出目标(-o binary)，可产生一个原始的二进制文件，实质上是将所有的符号和重定位信息都将被抛弃，只剩下二进制数据。

## bzImage

zImage 是 ARM Linux 常用的一种压缩镜像文件，它是由vmlinux 加上**解压代码**经 gzip 压缩而成，命令格式是 `make zImage`，这种格式的 Linux 内核镜像文件多存放在 NAND Flash 上。

bzImage 不是用 bzip2 压缩的，**bz 表示 big zImage**,其格式与 zImage 类似，但采用了不同的压缩算法，注意，bzImage 是压缩率更高的内核映像。

zImage vs bzImage：它们不仅是一个压缩文件，而且在这两个文件的开头部分内嵌有解压缩代码。两者的**不同之处**在于，老的**zImage 解压缩内核到低端内存**(第一个 640K)，**bzImage解压缩内核到高端内存(1M以上)**。如果内核比较小，那么可以采用 zImage 或 bzImage 之一，两种方式引导的系统运行时是相同的。大的内核采用 bzImage，不能采用 zImage。

# 其他
## vmlinuz 和 vmlinux
vmlinuz由ELF文件vmlinux 经过`OBJCOPY`，并压缩后的文件。Linux下大量工具都是对 ELF 文件进行解析，因此，若我们想要逆向对 Linux 内核镜像进行二进制的分析，就需要先将 vmlinuz 文件还原成 vmlinux 文件。

### vmlinuz 到 vmlinux 的转换
#### 工具安装
Linux内核提供了脚本来实现vmlinuz 到 vmlinux 的转换。
在 Centos 和 Ubuntu 下可以通过包管理器直接安装，如下：
```bash
yum install kernel-devel
```
安装完成后，`extract-linux`脚本将保存在：
```bash
- Centos: `/usr/src/kernels/$(uname -r)/scripts/extract-vmlinux`
- Ubuntu: `/usr/src/linux-headers-$(uname -r)/scripts/extract-vmlinux`
```

#### 文件转换
```bash
/usr/src/kernels/$(uname -r)/scripts/extract-vmlinux vmlinuz-$(uname -r) > vmlinux
```

#### ELF 文件解析
Linux 下 常用的 ELF 文件解析工具是 `readelf`和`objdump`，以下记录一些常用的例子：
(1) `readelf` 查看 ELF 文件的信息：
```js
- -S --section-headers 查看段头
- -h --file-header 查看文件头
- -r --relocs 查看重定位相关信息
- -x --hex-dump 查看16进制
```

(2) `objdump`读取 ELF文件中的内容
```js
- -d 查看反编译结果
- -j 指定查看的段
- -r 查看重定位相关信息
- 例子： `objdump -d -j .text hello.o`
```

通过 objdump 命令则可反汇编 vmlinux 文件。
```bash
 objdump -D vmlinux | less
```

但该文件中没有保存符号名（例如函数名），这样分析不太方便。符号名相关的内容保存在`/boot/System.map`文件中。例如查找函数`tcp_v4_do_rcv`的地址，可执行以下命令获取到函数`tcp_v4_do_rcv`的地址为`ffffffff816c62e0`，再结合反汇编结果进行分析。
```bash
grep "tcp_v4_do_rcv" /boot/System.map-3.10.0-1160.15.2.el7.x86_64 

ffffffff816c62e0 T tcp_v4_do_rcv
```


## 内核模块(kernel module)和内核镜像的关系
既然内核镜像已经包含了硬件检测和驱动模块，那么我们为啥还需要核心模块呢？
这是因为，由于硬件的更新换代速度特别快，老的内核无法支持新硬件的运行，但是又不能仅仅为了一个小硬件去重新编译生成新的内核镜像，因此就产生了内核模块。

我们可以将一些不常用或者较新的硬件驱动编译成独立的模块，然后内核在正常运行过程中将需要的模块动态的加载到内核中以实现对其的支持

核心模块存放路径：`/lib/modules/$(uname -r)/kernel/`

## Linux 的版本号
Linux 的版本号分为两部分，即内核版本与发行版本。
### 内核版本号
内核版本号由3个数字组成：A.B.C。各数字含义如下：

- A：**内核主版本号**。这是很少发生变化，只有当发生重大变化的代码和内核发生才会发生。在历史上曾改变两次的内核：1994年的1.0及1996年的2.0。
- B：**内核次版本号**。是指一些重大修改的内核。偶数表示稳定版本；奇数表示开发中版本。
- C：**内核修订版本号**。是指轻微修订的内核。这个数字当有安全补丁,bug修复，新的功能或驱动程序，内核便会有变化。


第二种方式：
`major.minor.patch-build.desc`
- major : 主版本号，有结构变化才变更
- minor : 次版本号，新增功能时才发生变化，一般技术表示测试版，偶数表示生产版
- patch : 补丁包数或次版本的修改次数
- build : 编译（或构建）的次数，每次编译可能对少量程序做优化或修改，但一般没有大的（可控的）功能变化。
- desc : 当前版本的特殊信息，其信息由编译时指定，具有较大的随意性，有如下的标识是常用的：
rc（或r），表示发行候选版本（release candidate），rc后的数字表示该正式版本的第几个候选版本，多数情况下，各候选版本之间数字越大越接近正式版。
smp，表示对称多处理器（Symmetric MultiProcessing）。
pp，在Red Hat Linux中常用来表示测试版本（pre-patch）。
EL，在Red Hat Linux中用来表示企业版Linux（Enterprise Linux）。
mm，表示专门用来测试新的技术或新功能的版本。
fc，在Red Hat Linux中表示Fedora Core。

## System.map
### 介绍
System.map 内核符号映射表，记录了所有符号的运行地址,这里的符号可以理解成函数名和变量。通过查看System.map文件可以帮助我们理解内核编译。System.map文件不是一层不变的，每次编译内核都会重新生成System.map文件。

### 作用
对计算机而言是没有符号这个概念的，只有0和1；但是我们比较容易理解的是函数名这样的符号，System.map文件就是计算机和人类在理解程序中的桥梁。当程序报错的时候，计算机会在堆栈信息里保存出错的内存地址，但是我们光看内存地址是没法理解程序到底是哪里出错。于是可以把出错的内存地址通过System.map文件转换成函数名，这样我们就知道是哪个函数出错了。

我们用gdb调试程序的时候，可以通过函数名设置断点，也是因为在程序中有一份符号表，如果用strip后的程序做gdb调试，在用函数名设置断点的时候会提示找不到函数名，因为程序里的符号信息都被删除了。

### System.map文件解析
(1)System.map文件的格式：地址 类型 符号

(2)符号类型：大写为全局符号，小写为局部符号
```bash
A：该符号的值是不能改变的，等于const
B：该符号来自于未初始化代码段bss段
C: 该符号是通用的，通用的符号指未初始化的数据。当链接时，多个通用符号可能对应一个名称，如果该符号在某一个位置定义，这个通用符号被当做未定义的引用。不明白，内核中也没有该类型的符号
D: 该符号位于初始化的数据段
G: 位于初始化数据段，专门对应小的数据对象，比如global int x,对应的大数据对象为 数组类型等
I： 到其他符号的间接引用，是对于a.out文件的GNU扩展，使用非常少
N：调试符号
R：只读代码段的符号
S：BSS段（未初始化数据段）的小对象符号
T：代码段符号，全局函数，t为局部函数
U：未定义的符号
V：该符号是一个weak object，当其连接到为定义的对象上上，该符号的值变为0
W： 类似于V
—： 该符号是a.out文件中的一个stabs symbol，获取调试信息
？： 未知类型的符号
U：未定义的符号
```
![](attachments/Pasted%20image%2020240407110842.png)

(3) 范例
```bash
c0004000 A swapper_pg_dir
c0008000 T __init_begin
c0008000 T _sinittext
c0008000 T _stext
c0008000 T stext
c0008034 t __enable_mmu
c0008060 t __turn_mmu_on
c0008078 t __create_page_tables
c00080f0 t __switch_data
c0008118 t __mmap_switched
c0008160 t __error
c0008160 t __error_a
c0008160 t __error_p
·······
```

### 应用
 虽然内核本身并不真正使用System.map，但其它程序比如klogd， lsof和ps等软件需要一个正确的System.map。如果你使用错误的或没有System.map，klogd的输出将是不可靠的，这对于排除程序故障会带来困难。

#### klogd
Linux使用了一个称为 klogd( 内核日志后台程序)的 后台程序，klogd会截取内核oops(可以理解为内核的段错误)并且使用syslogd将其记录下来，并将某些象c010b860 的信息转换成我们可以识别和使用的信息。换句话说，klogd是一个内核消息记录器(logger)， 它可以进行名字-地址之间的解析。一旦klogd开始转换内核消息，它就使用手头的记录器， 将整个系统的消息记录下来，通常是使用syslogd记录器。 

为了进行名字-地址解析，klogd就要用到System.map文件。

##### klogd 的地址转换分类
 其实klogd会执行两类地址解析活动。
- 静态转换：将使用System.map文件。
- 动态转换：该方式用于可加载模块，不使用System.map。

假设你加载了一个产生oops的内核模块。于是就会产生一个oops消息，klogd就会截获它，并发现该oops发生在d00cf810处。由于该地址属于动态加载模块，因此在System.map文件中没有对应条目。klogd将会在其中寻找并会毫无所获，于是断定是一个可加载模块产生了oops。此时klogd就会向内核查询该可加载模块输出的符号。即使该模块的编制者没有输出其符号，klogd也起码会知道是哪个模块产生了oops，这总比对一个oops一无所知要好。

### System.map的位置
 执行：man klogd可知，如果没有将System.map作为一个变量的位置给klogd，那么它将按照下面的顺序，在三个地方查找System.map：
```bash
- /boot/System.map
- /System.map
- /usr/src/linux/System.map
```
System.map也有版本信息，klogd能够智能地查找正确的映象（map）文件。


#### 其他工具
不要认为System.map文件仅对内核oops有用。尽管内核本身实际上不使用System.map，其它程序，象klogd，lsof，
```bash
satan# strace lsof 2>&1 1> /dev/null | grep System readlink("/proc/22711/fd/4", "/boot/System.map-2.4.18", 4095) = 23
```

```bash
satan# strace ps 2>&1 1> /dev/null | grep System open("/boot/System.map-2.4.18", O_RDONLY|O_NONBLOCK|O_NOCTTY) = 6
```


### kallsyms
内核启动时候创建,供oops时定位错误，文件大小总为0，包含当前内核导出的、可供使用的变量或者函数；它只是内核数据的简单表示形式.
/proc/kallsyms是一个在启动时由Linux kernel实时产生的文件，当系统有任何变更时，它就会马上做出修正；可以理解为动态的符号表

### 注意
当你编译一个新内核时，原来的System.map中的符号信息就不正确了。随着每次内核的编译，就会产生一个新的 System.map文件，各个符号名的地址要发生变化，你的老的System.map具有的是错误的符号信息。每次内核编译时产生一个新的System.map，你应当用新的System.map来取代老的System.map。


## 内核发行版和内核的关系
### 内核发行版介绍
**背景**
仅有内核而没有应用软件的操作系统是无法使用的，所以许多公司或社团将内核、源代码及相关的应用程序组织构成一个完整的操作系统，让一般的用户可以简便地安装和使用Linux，这就是所谓的发行版本（distribution）。

**介绍**
linux发行版本：Linux发行版就是由Linux内核与各种常用软件的集合产品，如今全球大约有数百款的Linux发行版本，根据不同标准可以把Linux发行版本进行不同性质的分类，比如一种分类方式是根据它是社区维护还是商业公司维护，Linux发行版主要有三个分支：Debian、Slackware、Redhat。

### 区别
linux内核和发行版的区别是：linux内核安装完成后没有用户界面和软件，是提供硬件抽象层、硬盘以及文件系统控制的核心程序；
而linux发行版是在内核的基础上加入了用户界面和各种软件的支持。

1、linux核心只有内核部分，安装完后，用户界面/软件都没有。内核是系统的心脏，是linux中最基层的代码。内核是运行程序和管理像磁盘和打印机等硬件设备的核心程序，它**提供了一个在裸设备与应用程序间的抽象层**。例如，程序本身不需要了解用户的主板芯片集或磁盘控制器的细节就能在高层次上读写磁盘。

2、linux发行版，就是在内核的基础上，加入用户界面，各种软件的支持。比如CenterOS、小红帽等等。在内核的基础上，开发不同应用程序，组成的一个完整的操作系统。、

### 查看
查看系统的内核信息： 
```bash
# uname -a 
#cat /proc/version
```

查看系统的发行版本信息：
```bash
#lsb_release -a 
#cat /etc/issue
$ cat /etc/centos-release
CentOS Linux release 7.4.1708 (Core)

```

### 常见的发行版
**商业版本以Redhat为代表**。
CentOS 你会发现非常多的商业公司部署在生产环境上的服务器都是使用的CentOS系统，CentOS是从RHEL源代码编译的社区重新发布版。
CentOS简约，命令行下的人性化做得比较好，稳定，有着强大的英文文档与开发社区的支持。与Redhat有着相同的渊源。虽然不单独提供商业支持，但往往可以从Redhat中找到一丝线索。相对debian来说，CentOS略显体积大一点。是一个非常成熟的Linux发行版。

**开源社区版本则以debian为代表**
一般来说Debian作为适合于服务器的操作系统，它比Ubuntu要稳定得多。可以说稳定得无与伦比了。debian整个系统，只要应用层面不出现逻辑缺陷，基本上固若金汤，是个常年不需要重启的系统（当然，这是夸张了点，但并没有夸大其稳定性）。debian整个系统基础核心非常小，不仅稳定，而且占用硬盘空间小，占用内存小。128M的VPS即可以流畅运行Debian，而CentOS则会略显吃力。但是由于Debian的发展路线，使它的帮助文档相对于CentOS略少，技术资料也少一些。 由于其优秀的表现与稳定性，Debian非常受VPS用户的欢迎。 此外还有Arch Linxu、Gentoo、Slackware等一系列的Linux和FreeBSD、Unix等系统，由于其涉及领域更加专业，很少在VPS中出现，因此不作介绍。

#### Red Hat Linux

Red Hat是最成功的Linux发行版本之一，它的特点是安装和使用简单。Red Hat可以让用户很快享受到Linux的强大功能而免去繁琐的安装与设置工作。Red Hat是全球最流行的Linux，Red Hat已经成为Linux的代名词，许多人一提到Linux就会毫不犹豫地想到Red Hat。它曾被权威计算机杂志InfoWorld评为最佳Linux。

官方网站：[http://www.redhat.com/](https://link.zhihu.com/?target=http%3A//www.redhat.com/)

#### Debian Linux

Debian可以算是迄今为止最遵循GNU规范的Linux系统，它的特点是使用了Debian系列特有的软件包管理工具dpkg，使得安装、升级、删除和管理软件变得非常简单。Debian是完全由网络上的Linux爱好者负责维护的发行套件。这些志愿者的目的是制作一个可以同商业操作系统相媲美的免费操作系统。并且其所有的组成部分都是自由软件。

官方网站：[http://www.debian.org/](https://link.zhihu.com/?target=http%3A//www.debian.org/)


# `grub`选择要启动的内核镜像
![](attachments/Pasted%20image%2020240323141454.png)

# 参考
```bash
# [Linux内核映像：系统必备的关键组件 (linux kernel image)](https://zhuji.vsping.com/24821.html)

```