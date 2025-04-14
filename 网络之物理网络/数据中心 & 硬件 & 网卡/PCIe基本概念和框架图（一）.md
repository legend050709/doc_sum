```table-of-contents
```
# 概述
PCIe的全称是Peripheral Component Interconnect Express，是一种用于连接外设的总线。它于2003年提出来，作为替代PCI和PCI-X的方案，现在已经成了现代CPU和其他几乎所有外设交互的标准或者基石，比如，我们马上能想到的GPU，网卡，USB控制器，声卡，网卡等等，这些都是通过PCIe总线进行连接的，然后现在非常常见的基于m.2接口的SSD，也是使用NVMe协议，通过PCIe总线进行连接的，除此以外，Thunderbolt 3 ，USB4 ，甚至最新的CXL互联协议，都是基于PCIe的！

# 简介
PCIe 1.0，一改之前PCI共享总线的架构，改为点对点（Point to Point）的连接（Link），一个Link的两端分别是两个PCIe设备（PCIe Device），一个Link可以最大包含16条数据通道（Lane），每条Lane采用全双工（Full Duplex）传输，每条Lane包含方向相反的2对差分线，共计4根线。

PCIe1.0的最大带宽为16Lane x 2.5Gbit/s=5GByte/s。或者鸡贼点可说PCIe 1.0的双向带宽最大为16Lane x 2.5Gbit/s x2 =10GByte/s

![](../限速%20&%20流控/attachments/Pasted%20image%2020230731204125.png)
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731204201.png)
```c
- PCIe 1.0, Gen1, 2.5Gbps/Lane
- PCIe 2.0, Gen2, 5Gbps/Lane
- PCIe 3.0, Gen3, 8Gbps/Lane
- PCIe 4.0, Gen4, 16Gbps/Lane
- PCIe 5.0, Gen5, 32Gbps/Lane
- PCIe 6.0 In progress, Gen6, 64Gbps/Lane, PAM4
```

一个PCI Express连接可以被配置成x1， x2， x4， x8， x12， x16和x32的数据带宽。 (x2 and x12 link widths are optional) PCI-E 各种位宽Device可以自由搭配使用，比如x1 的卡可以插到x8的插槽中使用， x8 的卡可以插到x16的插槽中使用，升级方便。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731204532.png)

一些常见的PCI-E设备如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731204654.png)

# 传统的PCI的架构图
计算机网络的最主要的拓扑结构有总线型拓扑、环形拓扑、树形拓扑、星形拓扑、混合型拓扑以及网状拓扑。

PCI采用的是**总线型拓扑结构**，一条PCI总线上挂着若干个PCI终端设备或者PCI桥设备，大家共享该条PCI总线，哪个人想说话，必须获得总线使用权，然后才能发言。下面是一个基于PCI的传统计算机系统：
![](attachments/Pasted%20image%2020230806175815.png)
北桥下面的那根PCI总线，挂载了以太网设备、SCSI设备、南桥以及其他设备，他们共享那条总线，某个设备只有获得总线使用权才能进行数据传输。

# PCIe架构图
PCIe的架构主要由五个部分组成：Root Complex，PCIe Bus，Endpoint，Port and Bridge，Switch。其整体架构呈现一个**树形拓扑结构**，如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801231604.png)


## link and lane

**Lane的概念：**一组不同的信号对，一对用来传送，一对用来接收。由于PCIE使用差分信号传输，一条lane四条线，两条线组成一对，供发送。另外两条接收！
![](attachments/Pasted%20image%2020230806175532.png)
Link Width这一行，我们看到X1，X2，X4…，这是什么意思？这是指PCIe连接的通道数（Lane）

**Link的概念**：两个设备之间的PCIe连接，叫做一个Link。即两个Ports和他们之间所连接Lanes的集合。一个Link是一个双工通信通道在两个部件之间。
![](attachments/Pasted%20image%2020230806175624.png)
```c
Link:
	The collection of two Ports and their interconnecting Lanes. A Link is a dualsimplex communications path between two components.
```
从A到B，之间是个双向连接，车可以从A驶向B，同时，车也可以从B驶向A，各行其道。两个PCIe设备之间，有专门的发送和接收通道，数据可以同时往两个方向传输，PCIe spec称这种工作模式为双单工模式（dual-simplex），可以理解为全双工模式。

## Root Complex
![](attachments/Pasted%20image%2020230806180503.png)

```c
Root complex  
   
A defined System Element that includes zero or more Host Bridges, zero or more  
Root Complex Integrated Endpoints, zero or more Root Complex Event  
Collectors, and one or more Root Ports.
```
**Root Complex的概念：** 一个的系统元素，包含一个Host Bridge, 0个或多个集成EndPoints的Root Complex, 0个或多个Root Complex时间收集器，0个或多个Root Ports.

Root Complex是整个PCIe设备树的根节点，CPU通过它与PCIe的总线相连，并最终连接到所有的PCIe设备上。

由于Root Complex是管理外部IO设备的，所以在早期的CPU上，Root Complex其实是放在了北桥（MCU）上，后来随着技术的发展，现在已经都集成进了CPU内部了 。（注意下图的System Agent的部分，他就是PCIe Root Complex所在的位置。）
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801231858.png)
虽然是根节点，但是系统里面可以存在不只一个Root Complex。随着PCIe Lane的增加，PCIe控制器和Root Complex的数量也随之增加。比如，我的台式机的CPU是i9-10980xe，上面就有4个Root Complex，而我的笔记本是i7-9750H，上面就只有一个Root Complex。我们在Windows上可以通过设备管理器来查看：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230802104429.png)



而我们可以通过`lspci`命令来查看所有的Root Complex：
```c
$ lspci -t -v
-+-[0000:c0]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
 +-[0000:80]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
 +-[0000:40]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
 \-[0000:00]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex

```

### host bridge
**Host bridge**：Root Complex中用来链接一个主CPU或多个CPU和一个层次结构的一个部分。
```c
Host Bridge:
	Part of a Root Complex that connects a host CPU or CPUs to a Hierarchy.
```

## PCIe总线（PCIe Bus)
### 介绍
- 总线的背景
任何一个微处理器都要与一定数量的部件和外围设备连接，但如果将各部件和每一种外围设备都分别用一组线路与CPU直接连接，那么连线将会错综复杂，甚至难以实现。为了简化硬件电路设计、简化系统结构，常用一组线路，配置以适当的接口电路，与各部件和外围设备连接，这组共用的连接线路被称为总线。采用总线结构便于部件和设备的扩充，尤其制定了统一的总线标准则容易使不同设备间实现互连。

- 总线的物理分类
微机中总线一般有内部总线、系统总线和外部总线。内部总线是微机内部各外围芯片与处理器之间的总线，用于芯片一级的互连；而系统总线是微机中各插件板与系统板之间的总线，用于插件板一级的互连；外部总线则是微机和外部设备之间的总线，微机作为一种设备，通过该总线和其他设备进行信息与数据交换，它用于设备一级的互连。

- 总线的传输分类
从广义上说，计算机通信方式可以分为并行通信和串行通信，相应的通信总线被称为**并行总线**和**串行总线**。
串行总线：字面意思来看，串行就是数据是一位一位的发送，并行就是数据一组一组的发送。并行总线由于是多个数据同时传输，需要考虑数据的协同性，这就导致了并行传输的频率不能做的很高。存储芯片DDR就是并行传输，它有一组数据线D0—D7，加DQS，DQM，这组线是一起传输的，只要有其中一位出错，数据就不能够正确传输过去，需要重新传输。而串行数据是一位一位的传，位与位之间没有联系，不会因为这位有错误，使下一位不能传输，串行总线因为只有一条链路，就可以把频率做的很高，提高传输速度，速度提高了就能够弥补一次只能传输一个数据的缺陷。。

此外，并行总线两根相邻的链路其数据是同时传输的，这就会导致它们彼此之间会产生严重干扰，并行的链路越多，干扰越强。因此并行总线需要加强抗干扰的能力，否则传输过程中数据就可能被损坏。如果传输过程中数据故障了，就需要重新对齐数据再传输。而串行总线如果一个数据出错了，只需要重新传输一次就好了，由于串行总线频率高，很快就可以把错误数据重新传输过去。
再次，由于并行总线是多链路一块传输数据，就需要很多线，接口需要很多针脚，老式计算机里的并行接口做得很大，接线比较宽，针脚非常多。这样一来装机也很麻烦，因为走线不方便、接口体积很大。
正是上面的这些缺点，电脑总线就逐渐从并行传输替换成了串行传输，比如USB、硬盘的SATA等。
> 需要注意的是，显卡底部的金手指密密麻麻一大排，接口是PCIE x16，外形很像并行总线，但实际上是一种串行总线。串行总线可以做多链路传输，和并行链路不一样，它的每根链路是独立数据，相互之间没有关系，不会受到其他数据的干扰。

### 特点
PCIe总线（Bus）,PCIe上的设备通过PCIe总线互相连接。虽然PCIe是从PCI发展而来的，并且甚至有很多地方是兼容的，但是它与老式的PCI和PCI-X有两点特别重要的不同：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801232411.png)
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801232427.png)
>1. PCIe的总线并不是我们传统意义上共享线路的总线（Bus），而是一个点对点的网络，我们如果把PCI比喻成网络中的集线器（Hub），那么PCIe对应的就是交换机了。换句话说，当Root Complex或者PCIe上的设备之间需要通信的时候，它们会与对方直接连接或者通过交换电路进行点对点的信号传输。
>2. 老式的PCI使用的是单端并行信号进行连接，但是由于干扰过大导致频率无法提升，所以后来就演变成PCIe之后就开始使用了高速串行信号。这也导致了PCI设备和PCIe设备无法兼容，只能通过PCI-PCIe桥接器来进行连接。当然这些我们都不需要再去关心了，因为现在已经很少看见PCI的设备了。

- PCIe总线的全双工和多通道Lane
和很多的串行总线一样，PCIe采用了全双工的传输设计，即允许在同一时刻，同时进行发送和接收数据。如下图所示，设备A和设备B之间通过双向的Link相连接，每个Link支持1到32个通道（Lane）。由于是串行总线，因此所有的数据（包括配置信息等）都是以数据包为单位进行发送的。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731214951.png)

与绝大部分的高速连接一样，PCIe采用了差分对进行收发，以提高总线的性能。一个PCIe Lane的例子如下图：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731215037.png)

- PCIe总线的可扩展性
PCIe相对于PCI总线的另一个大的优势是其的Scalable Performance，即可以根据应用的需要来调整PCIe设备的带宽。如需要很高的带宽，则采用多个Lane（比如显卡）；如果并不需要特别高的带宽，则只需要一个Lane就可以了（比如说网卡等）。

- PCIe的点对点连接
由于非常高的传输速度，PCIe是一种点对点连接的总线，而不像PCI那样的共享总线。但是PCIe总线系统可以通过Switch连接多个PCIe设备，也可以通过PCIe桥连接传统的PCI和PCI-X设备。一个简单的PCIe总线系统的拓扑结构图如下所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731215323.png)

## PCIe Device

PCIe上连接的设备可以分为两种类型：

- Type 0：它表示一个PCIe上**最终端**的设备，比如我们常见的显卡，声卡，网卡等等。
- Type 1：它表示一个PCIe Switch或者Root Port。和终端设备不同，它的主要作用是用来连接其他的PCIe设备，其中PCIe的Switch和网络中的交换机类似。

```c
常用命令：
1》 查看设备的类型
	lspci -nn
	lspci -nn | grep -i eth
2》 查看设备树
	lspci -t -v
	注：通过pci树可以看到某个EP的上游端口。
	
3》 查看具体的设备
	lspci -s PCI_ID -vv

```

### BDF（Bus Number, Device Number, Function Number）
PCIe上所有的设备，无论是Type 0还是Type 1，在系统启动的时候，都会被分配一个唯一的地址，它有三个部分组成：

- **B**us Number：8 bits，也就是最多256条总线
- **D**evice Number：5 bits，也就是最多32个设备
- **F**unction Number：3 bits，也就是最多8个功能

这就是我们常说的**BDF**，它类似于网络中的IP地址，一般写作`BB:DD.F`的格式。在Linux上，我们可以通过`lspci`命令来查看每个设备的BDF，比如，下面这个FCH SMBus Controller就是`00:14.0`
```c
$ lspci -t -v
 # [Domain:Bus]
 \-[0000:00]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
         # Device.Function
             +-14.0  Advanced Micro Devices, Inc. [AMD] FCH SMBus Controller

```
由于默认BDF的方式最多只支持8个Function，可能不够用，所以PCIe还支持另一种解析方式，叫做ARI（Alternative Routing-ID Interpretation），它将Device Number和Function Number合并为一个8bit的字段，只用于表示Function，所以最多可以支持256个Function，但是它是可选的，需要通过设备配置启用.
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801233102.png)

### Type 0 Device和Endpoint
所有连接到PCIe总线上的Type 0设备（终端设备），都可以来实现PCIe的Endpoint，用来发起或者接收PCIe的请求和消息。**每个设备可以实现一个或者多个Endpoint，每个Endpoint都对应着一个特定的功能Function**。比如：
- 一块双网口的网卡，可以每个为每个网口实现一个单独的Endpoint；
- 一块显卡，其中实现了4个Endpoint：一个显卡本身的Endpoint，一个Audio Endpoint，一个USB Endpoint，一个UCSI Endpoint；

这些我们都可以通过`lspci`或者Windows上的设备管理器来查看：
```c
$ lspci -t -v
-+-[0000:c0]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
             # A NIC card with 2 ports:
 |           +-01.1-[c1]--+-00.0  Mellanox Technologies MT2892 Family [ConnectX-6 Dx]
 |           |            \-00.1  Mellanox Technologies MT2892 Family [ConnectX-6 Dx]
 +-[0000:80]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
             # A graphic card with 4 endpoints:
 |           +-01.1-[81]--+-00.0  NVIDIA Corporation TU104 [GeForce RTX 2080]
 |           |            +-00.1  NVIDIA Corporation TU104 HD Audio Controller
 |           |            +-00.2  NVIDIA Corporation TU104 USB 3.1 Host Controller
 |           |            \-00.3  NVIDIA Corporation TU104 USB Type-C UCSI Controller

```
### RCIE（Root Complex Integrated Endpoint）
说到PCIe设备，脑海里面可能第一反应就是有一个PCIe的插槽，然后把显卡或者其他设备插在里面，就像我们上面看到的这样。但是其实系统中有大量的设备是主板上集成好了的，比如，内存控制器，集成显卡，Ethernet网卡，声卡，USB控制器等等。这些设备在连接PCIe的时候，可以直接连接到Root Complex上面。这种设备就叫做RCIE（Root Complex Integrated Endpoint），如果我们去查看的话，他们的Bus Number都是0，代表Root Complex。

![](../限速%20&%20流控/attachments/Pasted%20image%2020230802105415.png)

### root Port / Bridge



其他的需要通过插槽连接的设备呢？这些设备就需要通过PCIe Port来连接了。
在Root Complex上，有很多的Root Port，这些Port每一个都可以连接一个PCIe设备（Type 0或者Type 1）。

![](../限速%20&%20流控/attachments/Pasted%20image%2020230802105657.png)

#### Pci Bridge

- Pci bridge 作用
本质上，所有这些连接其他设备用的部件都是由桥（Bridge）来实现的，这些桥的两端连接着两个不同的PCIe Bus（Bus Number不同）。

- Pci bridge 分类
PCI桥主要包括以下三种：

1). Host/PCI桥(host bridge): 用于连接CPU与PCI根总线，第1个根总线的编号为0。在PC中，内存控制器也通常被集成到Host/PCI桥设备芯片中，因此Host/PCI桥通常也被称为“北桥芯片组(North Bridge Chipset)”。

2). PCI/ISA桥: 用于连接旧的ISA总线。通常，PCI中类似i8359A中断控制器这样的设备也会被集成到PCI/ISA桥设备中。因此，PCI/ISA桥通常也被称为“南桥芯片组(South Bridge Chipset)”

3). PCI-to-PCI桥(以下称为PCI-PCI桥): 用于连接PCI主总线(Primary Bus)和次总线(Secondary Bus)。PCI-PCI桥所处的PCI总线称为主总线，即次总线的父总线；PCI-PCI桥所连接的PCI总线称为次总线，即主总线的子总线。

#### Root Port
一个Root Port其实是靠两个Bridge来实现的：一个（共享的）Host Bridge（上游连接着CPU，下游连接着Bus 0）和一个PCI Bridge用来连接下游设备（上游连着的是Bus 0（Root Complex），下游连着的PCIe的设备（Bus Number在启动过程中自动分配））

![](../限速%20&%20流控/attachments/Pasted%20image%2020230802110004.png)

**Root Port的概念：**一个位于Root Complex上通过相关联的虚拟PCI-PCI Bridge映射一个层次结构整体部分的的PCIE Port
```c
Root Port:
	A PCI Express Port on a Root Complex that maps a portion of a Hierarchy through an associated virtual PCI-PCI Bridge.
```


我们通过`lspci`命令可以看到这些桥的存在（注意设备详情中的Kernel driver in use: pcieport）：
```c
 +-[0000:80]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
             # This is the Host bridge that connects to the root port and CPU:
 |           +-01.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge
             # This is the PCI bridge that connects to the root port and device with a new bus - 0x81:
 |           +-01.1-[81]--+-00.0  NVIDIA Corporation TU104 [GeForce RTX 2080]
 |           |            +-00.1  NVIDIA Corporation TU104 HD Audio Controller
 |           |            +-00.2  NVIDIA Corporation TU104 USB 3.1 Host Controller
 |           |            \-00.3  NVIDIA Corporation TU104 USB Type-C UCSI Controller

# Host bridge
$ sudo lspci -s 80:01.0 -v
80:01.0 Host bridge: Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge
        Flags: fast devsel, IOMMU group 13

# PCI bridge
$ sudo lspci -s 80:01.1 -v
80:01.1 PCI bridge: Advanced Micro Devices, Inc. [AMD] Starship/Matisse GPP Bridge (prog-if 00 [Normal decode])
        Flags: bus master, fast devsel, latency 0, IRQ 35, IOMMU group 13
        Bus: primary=80, secondary=81, subordinate=81, sec-latency=0
        I/O behind bridge: 0000b000-0000bfff [size=4K]
        Memory behind bridge: f0000000-f10fffff [size=17M]
        Prefetchable memory behind bridge: 0000020030000000-00000200420fffff [size=289M]
        ....
        Kernel driver in use: pcieport

```
注意：是否使用PCIe Bridge和是否通过插槽连接不能直接划等号，这取决于你系统的硬件实现，比如，从上面RCIE的截图中我们可以看到USB Controller作为RCIE存在，而下面EPYC的CPU则不同，USB控制器是通过Root Port连接的，但是它在主板上并没有插槽。
```c
$ lspci -t -v
 +-[0000:40]-+-00.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse Root Complex
             +-03.0  Advanced Micro Devices, Inc. [AMD] Starship/Matisse PCIe Dummy Host Bridge
 |           +-03.3-[42]----00.0  ASMedia Technology Inc. ASM1042A USB 3.0 Host Controller
             # ^====== 40:03.3 here is a Bridge. And USB controller is connected
             #         to this Bridge with a new Bus Number 42.

```

### Switch
如果我们需要连接不止一个设备怎么办呢？这时候就需要用到PCIe Switch了。
PCIe Switch内部主要有三个部分：
- 一个Upstream Port和Bridge：用于连接到上游的Port，比如，Root Port或者上游Switch的Downstream Port
- 一组Downstream Port和Bridge：用于连接下游的设备，比如，显卡，网卡，或者下游Switch的Upstream Port
- 一根虚拟总线：用于将上游和下游的所有端口连接起来，这样，上游的Port就可以访问下游的设备了
![](../限速%20&%20流控/attachments/Pasted%20image%2020230802110759.png)
另外，这里再说明一次 —— 由于PCIe的信号传输是点对点的，所以Switch中间的这个总线只是一个逻辑上的虚拟的总线，其实并不存在，里面真正的结构是一套用于转发的交换电路。


Switch扩展了PCIe端口，靠近RC的那个端口，我们叫上游端口（upstream port），而分出来的其他端口，我们叫下游端口（downstream port）。一个Switch只有一个上游端口，可以扩展出若干个下游端口。下游端口可以直接连接Endpoint，也可以连接Switch，扩展出更多的PCIe端口。
![](attachments/Pasted%20image%2020230806180820.png)
对每个Switch来说，它下面的Endpoint或者Switch，都是归他管的：上游下来的数据，它需要甄别数据是传给它下面哪个设备，然后进行转发；下面设备向RC传数据，也要通过Switch代为转发的。因此，Switch的作用就是扩展PCIe端口，并为挂在它上面的设备（endpoint 或者switch)提供路由和转发服务。
每个Switch内部，也是有一根内部PCIe总线的，然后通过若干个Bridge，扩展出若干个下游端口，如下图所示：
![](attachments/Pasted%20image%2020230806180709.png)

最后，看到这里也许你会突然想到Root Complex是不是也可以看成是一个Switch呢？我觉得这两个概念最好还是分开，虽然从很多框图上看着确实很像，只不过Root Complex没有Upstream Port，连接上游的Host Bridge是连接到CPU上，不过Root Complex内部的功能要远比Switch复杂的多，里面不仅仅是简单的包转发，比如，后面会说到的PCIe请求的生成和转换等等。

## QA
**EndPoint是否可以直接访问另外一个EndPoint**
在PCIE这种点对点的模型中，设备之间之间的互联访问是可以的。　　
- 情况一：不需要CPU参与  
最典型的应用就是在一个带有DMA功能的Switch下，挂载两个EP，CPU需要首先配置DMA控制器，包括设置一些源地址，目标地址，传输数据以及数据量。然后每个设备发起DMA传输的时候，会直接透过Switch中的DMA控制器，发数据到另外一个设备，这个过程不需要CPU干预

- 情况二：CPU参与
这个过程就相对来说简单了，CPU从一个PCIE设备中读取出要发送的数据，然后直接发送给指定的目的PCIE设备节点即可。


## 小结
如果我们把所有这些部件连接在一起，那么其整体的结构就是这样的：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230802111009.png)

PCIe采用的是树形拓扑结构，RC是树的根，或者主干，它为CPU代言，与PCIe系统其它部分通讯，一般为通讯的发起者；Switch是树枝，树枝上有叶子（Endpoint），也可节外生枝，Switch上连Switch，归根结底，是为了连接更多的Endpoint。
Switch为它下面的Endpoint或Switch提供路由转发服务；Endpoint是树叶，诸如SSD，网卡，显卡等等，实现某些特定功能（function）。
>我们还看到有所谓的Bridge，用以将PCIe总线转换成PCI总线，或者反过来，不是我们要讲的重点，忽略之。PCIe与采用总线共享式通讯方式的PCI不同，PCIe采用点到点（Endpoint to Endpoint）通讯方式，每个设备独享通道带宽，速度和效率都比PCI好。

![](attachments/Pasted%20image%2020230806181157.png)

# PCIe速率
PCIe GEN1最大传输速率是2.5Gbps,PCIe GEN2最大传输速率是5Gbps，PCIe GEN3最大传输速率是8Gbps。这些速率只的是raw bit传输速率，并不是系统的有效data传输速率。有效的数据传输速率因为overhead要比此低。

各种类型的数据，例如内存、I/O 或配置数据，通过 PCI Express 系统传输。大多数设计都专注于尽可能有效地传输数据 。就本白皮书而言，_性能_定义为 系统的有效数据传输。 此后， data统称有效数据。


![](../限速%20&%20流控/attachments/Pasted%20image%2020230731204733.png)
**注：**这里的Switch实际上包含了多个类似于PCI总线中桥的概念。

> 注意：PCIe1.0, PCIe2.0会存在20%的编码开销。即PCIe的带宽*80%=实际的bps。
> 对于PCIe 3.0则，只有1.5%的开销。
**Note:** The main difference between the generations besides the supported speed is the encoding overhead of the packet. For generations 1 and 2, each packet sent on the PCIe has 20% PCIe headers overhead. This was improved in **generation 3,** where the overhead was reduced to 1.5% (**2/130**). See the actual PCIe bandwidth calculation below for more details.
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731230013.png)

# PCIe和PCI区别
对于一般用户来说，PCIe对用户可见的部分就是主板上大大小小的PCIe插槽了，有时还和PCI插槽混在一起，造成了一定的混乱，其实也很好区分：
![](attachments/Pasted%20image%2020230806182252.png)
如图，PCI插槽都是等长的，防呆口位置靠上，大部分都是纯白色。PCIe插槽大大小小，最小的x1，最大的x16，防呆口靠下。各种PCIe插槽大小如下：
![](attachments/Pasted%20image%2020230806182326.png)

# 常见问题
- Q:我主板上没有x1的插槽，我x1的串口卡能不能插在x4的插槽里。
A: 可以，完全没有问题。除了有点浪费外，串口卡也将已x1的方式工作。

- Q:我主板上只有一个x16的插槽，被我的显卡占据了。我还有个x16的RAID卡可以插在x8的插槽内吗？
A: 你也许会惊讶，但我的答案同样是：可以！你的RAID卡将以x8的方式工作。实际上来说，你可以将任何PCIe卡插入任何PCIe插槽中! PCIe在链接training的时候会动态调整出双方都可以接受的宽度。最后还有个小问题，你根本插不进去！呵呵，有些主板厂商会把PCIe插槽尾部开口，方便这种行为，不过很多情况下没有。这时怎么办？你懂的。

- Q: 我的显卡是PCIe 3.0的，主板是PCIe2.0的，能工作吗？
A: 可以，会以2.0工作。反之，亦然。

- Q: 我把x16的显卡插在主板上最长的x16插槽中，可是benchmark下来却说跑在x8下，怎么回事?！
A: 主板插槽x16不见得就连在支持x16的root port上，最好详细看看主板说明书，有些主板实际上是x8。有个主板原理图就更方便了。

- Q: 我新买的SSD是Mini PCIe的，Mini PCIe是什么鬼？
A: Mini PCIe接口常见于笔记本中，为54pin的插槽。多用于连接wifi网卡和SSD，注意不要和mSATA弄混了，两者完全可以互插，但大多数情况下不能混用（除了少数主板做了特殊处理），主板设计中的防呆设计到哪里去了！请仔细阅读主板说明书。另外也要小心不要和m.2(NGFF)搞混了，好在卡槽大小不一样。

# 常用命令
```c
1> 查看以太网设备
lspci | grpe -i eth
lspci -nn | grep -i eth

2> 查看设备的供应商、设备码，以及设备的类别
lspci -nn

3> 查看设备的具体信息
lspci -s PCIE-ID -vv

4> 树形结构查看机器上的Pcie设备
lspci -t -v

5> 查看设备的config space 的信息（16进制）
lspci -s PCIe-ID -xxx
```
![](attachments/Pasted%20image%2020230806180423.png)

# 参考
```c
1> pcie 系列：
http://r12f.com/posts/pcie-1-basics/

2> 系列2：
http://blog.chinaaet.com/justlxy/p/5100053481
```