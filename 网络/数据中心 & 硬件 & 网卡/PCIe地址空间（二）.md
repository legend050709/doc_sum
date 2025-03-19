```table-of-contents
```
# 概述
PCIe中主要定义了4种请求：Memory Transaction，I/O request，Configuration Space Access和Message。除了最后一种以外，其余三种全都是基于内存访问的，甚至连中断发起都是基于内存访问的，所以如果我们能很好的理解内存的访问，我们就能很好的理解PCIe。

在现代的操作系统中，当CPU想去访问一段内存的时候，它访问的地址并不是真实内存的物理地址，而是一个虚拟地址，这个地址需要经过MMU进行地址转换，将其变为物理地址之后才能通过总线去物理内存拿到真实的数据 。
![](attachments/Pasted%20image%2020230806191435.png)


而PCIe中基于内存访问的请求的实现，也正是利用类似的机制：

- PCIe中的每一个设备，无论是Endpoint（Type 0）还是Switch（Type 1），都会分配自己的内存地址空间，而这个地址空间会被映射到系统的**物理地址空间**中，并最终映射到虚拟内存中去。
- 当CPU发起一个内存读写请求的时候，如果这个地址经过了MMU的翻译，最后的**物理地址**落到了PCIe某个设备的内存空间之后，就会触发Root Complex将其转换为PCIe的请求，并通过PCIe总线发给对应的设备。

注：物理地址空间中不仅仅有物理内存，还有PCIe设备的内存，在访问物理地址的时候，处理器会将请求发给内存控制器，内存控制器会根据总线上各个Root Complex的Host Bridge的配置，将其传递给DRAM或者对应的Root Complex，最终交给对应的PCIe的设备。

# 简介
## 存储器域和PCI总线域
PCI spec规定了PCI设备必须提供的单独地址空间，因此PCIe设备的地址空间和CPU可以访问的地址空间是分开的。
根据王齐老师的**《PCI Express体系结构导读》**一书，后者可以被定义为存储器域，该域包括CPU内部的寄存器、主存空间，即CPU域和DRAM域（在可以接入显卡等设备的处理器系统中，CPU域并不能包含所有的DRAM域）;而前者可以被定义为PCI总线域。二者之间的地址转换通过PCIe HOST主桥来实现。

存储器域和PCI域相互访问需要先进行地址映射，即当CPU要访问PCI外部设备时（正向映射），需要先访问该设备在存储器域的地址，之后经过HOST主桥转换为PCI总线域的物理地址，然后通过PCI总线事务进行数据访问；而当PCI设备访问主存储器时（反响映射），首先通过PCI总线事务访问PCI总线域的地址空间，然后经过HOST主桥转换为存储器域的地址后，再对这些空间进行数据访问。

## PCI总线域划分
在PCI体系结构中一共支持两种地址空间，因此PCI总线域又可以进一步划分为Memory Address Space（MMIO 存储器映射空间）和Configuration Address Space（配置空间）。

- MMIO空间
MMIO(Memory mapping I/O)即内存映射I/O，它是PCI规范的一部分，I/O设备被放置在内存空间而不是I/O空间。从处理器的角度看，内存映射I/O后系统设备访问起来和内存一样。
这样如果访问AGP/PCI-E显卡上的帧缓存，BIOS可以对PCI设备使用读写内存一样的汇编指令完成，简化了程序设计的难度和接口的复杂性。
I/O作为CPU和外设交流的一个渠道，主要分为两种，一种是Port I/O，一种是MMIO(Memory mapping I/O)。


-  配置空间
配置空间（configuration space）,前64个字节（其地址范围为0x00～0x3F）是所有PCI设备必须支持的（有不少简单的设备也仅支持这些），此外PCI/PCI-X还扩展了0x40~0xFF这段配置空间，在这段空间主要存放一些与MSI或者MSI-X中断机制和电源管理相关的Capability结构。

## 地址转换
- outbound/inbound机制
部分PCIe控制器的设计中采用inbound和outbound寄存器组来保存存储器域和PCIe域的地址转换关系。

- outbound寄存器组
outbount寄存器实现存储器域地址向PCIe域地址的转换

只有当CPU读写访问的地址范围在outbound寄存器组管理的地址空间内时，HOST主桥才能接收CPU的读写访问，并将CPU在存储器域上的读写访问转换为PCI总线域上的读写访问，然后才能对PCI设备进行读写操作

- inbound寄存器组
inbound寄存器实现PCIe的地址向CPU域地址的转换

## PCIe设备mem空间
当处理器访问PCI设备的地址空间时，首先需要访问该设备在存储器域的地址空间，并通过HOST主桥蒋这个存储器域的地址空间转换为PCI总线域的地址空间后，在使用PCI总线事务将数据发送给PCI设备。
**同样，PCI设备访问存储器域的地址空间，类似进行DMA操作时，首先访问该存储器地址空间所对应的PCI总线地址空间，之后通过HOST主桥将PCI地址转换为存储器地址。**

### 访问机制——地址路由

- 地址映射方式
存储器访问的地址映射方式如下：  
PCIe总线域地址 = 请求访问的存储器域地址 - MMIO基地址 + bar基址（RP）

计算结果PCIe总线域地址在PCIe总线上广播，当该地址在某PCI设备的bar地址空间范围内，则访问该设备的地址空间。

## PCIe设备的config配置空间
每个PCIe设备，有这么一段空间，Host软件可以读取它获得该设备的一些信息，也可以通过它来配置该设备，这段空间就叫做PCIe的配置空间。不同于每个设备的其它空间，PCIe设备的配置空间是协议规定好的，哪个地方放什么内容，都是有定义的。

整个配置空间就是一系列寄存器的集合，其中Type 0是Endpoint的配置，Type 1是Bridge（PCIe时代就是Switch）的配置，都由两部分组成：64 Bytes的Header+192Bytes的Capability结构，后者是设备告诉Host它有多牛逼，都会什么绝活。

进入PCIe时代，PCIe能耐更大，192 Bytes不足以罗列它的绝活。为了保持后向兼容，又要不把绝活落下，怎么办？很简单，我扩展后者的空间，整个配置空间由256 Bytes扩展成4KB，前面256 Bytes保持不变：
![](attachments/Pasted%20image%2020230806190657.png)

先看看只占64 Bytes的Configuration Header。
![](attachments/Pasted%20image%2020230806190726.png)
像Device ID，Vendor ID，Class Code和Revision ID，是只读寄存器，PCIe设备通过这些寄存器告诉Host软件，这是哪个厂家的设备、设备ID是多少、以及是什么类型的（网卡？显卡？桥？）设备。

在一个PCIe拓扑结构里，一条总线下面可以挂几个设备，而每个设备可以具有几个功能，如下所示：
![](attachments/Pasted%20image%2020230806193007.png)
因此，在整个PCIe系统中，只要知道了Bus+Device+Function，就能找到对应的Function。寻址基本单元是功能（function），它的ID就由Bus+Device+Function组成 （BDF)。一个PCIe系统，可以最多有256条Bus，每条Bus上可以挂最多32个Device，而每个Device最多又能实现8个Function，而每个Function对应着4KB的配置空间。上电的时候，这些配置空间都是需要映射到Host的内存空间，因此，需要占用内存空间是：256*32*8*4KB =256MB。在这个动辄4GB、8GB内存的时代，256MB算不了什么。

## PCIe设备IO空间
通常对于memory空间和IO空间分开的架构才需要使用PCIe的IO空间，例如x86架构，其访问IO空间有专门的指令in/out等。


# PCIe的设备配置空间
在PCIe中，每个设备都会至少拥有一块独立的配置空间（Configuuraiton Space），这块空间的大小是4096字节，其中头部和PCI3.0保持兼容，有64个字节，这块空间的大小是固定的，不会随着设备的类型或者系统的重启而改变。【每个PCIe设备至少有一个配置空间。一个PCIe设备，它可能具有多个功能（function），比如既能当硬盘，还能当网卡。每个功能对应一个配置空间。】
![](attachments/Pasted%20image%2020230806191706.png)
上一章我们提到过，PCIe中有两类设备：Type 0表示终端设备，和Type 1表示Switch]。由于职责的不同，其配置空间的内容也不同。但是为了保持一致，方便管理，这两类设备的配置有很多相同的部分，比如配置空间的头部，如下图所示：

![](attachments/Pasted%20image%2020230806191733.png)

## 配置空间的分配与访问
在系统启动时，BIOS会通过ACPI（Advanced Configuration and Power Interface）找到所有的PCIe设备，并为其分配配置空间，映射到物理地址空间中，然后通过ECAM（Enhanced Configuration Access Mechanism）转交给操作系统。
而为了方便访问，PCIe使用BDF（bus::device.function）来构造每个配置空间相对于ECAM的偏移。由于每个空间都是4096个字节，所以PCIe将BDF向左移位了12位，对其进行预留。
打个比方，如果某个设备的BDF是`46:00.1`，ECAM基址是0xE0000000，那么其配置空间起始地址就是：`0xE0000000 + (0x46 << 20) | (0x00 << 15) | (0x01 << 12) = 0xE46001000`。或者简单的记忆就是BDF的Hex后面跟三个0。我们这里也可以通过`lspci`和`/dev/mem`进行直接的物理内存访问来验证：

```c
$ lspci -s 46:00.1  -nn
46:00.1 Ethernet controller [0200]: Broadcom Inc. and subsidiaries NetXtreme BCM5720 Gigabit Ethernet PCIe [14e4:165f]

$ sudo hexdump -x --skip 0xe4601000 /dev/mem | head
e4601000    14e4    165f    0406    0010    0000    0200    0010    0080
...


说明：
这段内存的前面几个数字`14e4`和`165f`就是这个设备的Vendor ID和Device ID，这和我们通过`lspci`看到的完全一致：`[14e4:165f]`。

当然，每次这样进行计算和转换来查看原始的配置空间是非常麻烦的，所以我们可以通过`setpci`来直接访问：
$ setpci -s 46:00.1 00.w
14e4

$ setpci -s 46:00.1 02.w
165f

```
## Type 0配置空间
接下来，我们来看看每一类设备的配置空间。由于PCIe上大部分我们使用的设备都是Type 0的终端设备，我们就先从Type 0开始吧！

### Type 0配置空间的结构
![](attachments/Pasted%20image%2020230806192054.png)

## BAR（Base Address Register）
### 背景
每个PCIe设备，都有自己的内部空间，这部分空间如果开放给Host（软件或者CPU)访问，那么Host怎样才能往这部分空间写入数据，或者读数据呢？
我们知道，CPU只能直接访问Host内存（Memory）空间（或者IO空间，我们不考虑），不对PCIe等外设直接操作。怎么办？记得皇帝身边那个有根的太监吗？Root Complex，RC。RC可以为CPU分忧。
解决办法是：CPU如果想访问某个设备的空间，由于它不能（或者不屑）亲自跟那些PCIe外设打交道，因此叫太监RC去办。比如，如果CPU想读PCIe外设的数据，先叫RC通过TLP把数据从PCIe外设读到Host内存，然后CPU从Host内存读数据；如果CPU要往外设写数据，则先把数据在内存中准备好，然后叫RC通过TLP写入到PCIe设备。完美！
![](attachments/Pasted%20image%2020230806190837.png)

上图例子中，最左边虚线的表示CPU要读Endpoint A的数据，RC则通过TLP（经历Switch）数据交互获得数据，并把它写入到系统内存中，然后CPU从内存中读取数据（紫色箭头所示），从而CPU间接完成对PCIe设备数据的读取。

具体实现就是上电的时候，系统把PCIe设备开放的空间（系统软件可见）映射到内存空间，CPU要访问该PCIe设备空间，只需访问对应的内存空间。RC检查该内存地址，如果发现该内存空间地址是某个PCIe设备空间的映射，就会触发其产生TLP，去访问对应的PCIe设备，读取或者写入PCIe设备。

一个PCIe设备，可能有若干个内部空间（属性可能不一样，比如有些可预读，有些不可预读）需要映射到内存空间，设备出厂时，这些空间的大小和属性都写在Configuration BAR寄存器里面，然后上电后，系统软件读取这些BAR，分别为其分配对应的系统内存空间，并把相应的内存基地址写回到BAR。（BAR的地址其实是PCI总线域的地址，CPU访问的是存储器域的地址，CPU访问PCIe设备时，需要把总线域地址转换成存储器域的地址。）
![](attachments/Pasted%20image%2020230806191054.png)

如上图例子，一个Native PCIe Endpoint，只支持Memory Map，它有两个不同属性的内部空间要开放给系统软件，因此，它可以分别映射到系统内存空间的两个地方；还有一个Legacy Endpoint，它既支持Memory Map，还支持IO Map，它也有两个不同属性的内部空间，分别映射到系统内存空间和IO空间。

### BAR 说明
在Type 0的配置空间中，BAR区域有24个字节，可以保存6个指针/地址，每一个都可以用来描述一个不同的内存空间或者IO空间的地址和范围。

为了描述不同类型的地址空间，这里的指针不是单纯的指针，而有着自己的结构：
![](attachments/Pasted%20image%2020230806192158.png)

其中：

- 最低位Bit 0：是一个标志位，用于描述地址空间的类型，0表示内存空间，1表示IO空间
- Memory Space中的Bit [2:1] - Type：用于描述内存空间的类型，00表示32位地址空间，10表示64位地址空间
- Memory Space中的Bit 3 - Prefetchable：用于描述内存空间是否支持预取，0表示不支持，1表示支持。如果一段内存空间支持预取，它意味着读取时不会产生任何副作用，所以CPU可以随时将其预取到DRAM中。而如果预取被启用，在读取数据时，内存控制器也会先去DRAM查看是否有缓存。当然，这是一把双刃剑，如果数据本身不支持预取，那么除了可能导致数据不一致，多一次DRAM的查询还会导致速度下降。


### 查看
对于BAR空间中保存的所有的地址，我们都可以通过`lspci`来查看到：
```c
$ sudo lspci -s 81:00.0 -nn -vv
81:00.0 VGA compatible controller [0300]: NVIDIA Corporation TU104 [GeForce RTX 2080] [10de:1e82] (rev a1) (prog-if 00 [VGA controller])
        Subsystem: Gigabyte Technology Co., Ltd TU104 [GeForce RTX 2080] [1458:37c1]
        ...
        Region 0: Memory at f0000000 (32-bit, non-prefetchable) [size=16M]
        Region 1: Memory at 20030000000 (64-bit, prefetchable) [size=256M]
        Region 3: Memory at 20040000000 (64-bit, prefetchable) [size=32M]
        Region 5: I/O ports at b000 [size=128]
        ...

```
从上面我们可以看到，这块显卡中有4个地址空间，三块是内存空间，一块是I/O空间。Region的编号表示其地址在BAR中间的偏移，比如Region 1就是BAR中的第二个DWORD，Region 3就是BAR中的第4个DWORD（Region 1是64位，所以需要占用8个字节），以此类推。这里我们也可以把原始的物理内存dump出来，进行验证。这里我把不同的地址，用不同的颜色标记了出来：
![](attachments/Pasted%20image%2020230806193538.png)
### 软件是如何读取Configuration空间
系统软件是如何读取Configuration空间呢？不能通过BAR中的地址，为什么？别忘了BAR是在Configuration中的，你首先要读取Configuration，才能得到BAR。前面不是系统为所有可能的Configuration预留了256MB内存空间吗？系统软件想访问哪个Configuration，只需指定相应Function对应的内存空间地址，RC发现这个地址是Configuration映射空间，就会产生相应的Configuration Read TLP去获得相应Function的Configuration。

Bus Number + Device + Function就唯一决定了目标设备； Ext Reg Number + Register Number相当于配置空间的偏移。找到了设备，然后指定了配置空间的偏移，就能找到具体想访问的配置空间的某个位置。

注：只有RC才能发起Configuration的访问请求，其他设备是不允许对别的设备进行Configuration读写的。

## 配置空间访问流程
就从CPU出发，用对配置空间的读请求做一个例子，来对整体的流程来一个总结吧！

1. 首先，CPU执行内存访问指令来读取虚拟内存中映射的，在ECAM中的，某个配置空间的内容。比如：`mov ax, [0x10e8100000]`。
2. 然后，这个读请求的地址经过MMU，查询页表得到物理内存的地址。假设，这个物理地址是BDF为 `81:00.0` 的设备的配置空间地址：`0xe8100000`。
3. 这个读请求会被发送给Memory Controller，Memory Controller检查这个地址之后，发现这个地址不属于DRAM，于是转发给对应的PCIE控制器，到Root Complex中。
4. Root Complex的Host Bridge收到这个请求，发现这个请求属于设备的配置空间，于是将这个请求转换为一个配置空间的读请求（请求名称叫CfgR0，具体的结构后面会介绍），地址是BDF `81:00.0`，Offset是0，长度是2个字节，并利用BDF开始路由。
5. Root Complex根据所有连接到其上面的设备和桥的配置空间里的配置，将这个请求转发给对应的设备。如果是设备本身就检查Device Number和Function Number，如果是桥，就检查Secondary Bus Number和Subordinate Bus Number，然后进行递归的转发。
6. 最后，请求到达设备。
# 参考
```c
https://blog.csdn.net/qq_39815222/article/details/121047739
```

