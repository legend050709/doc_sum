```table-of-contents
```
# 硬件架构
先来看一下常见的服务器架构中几个硬件组件之间的关系，下图中的CPU通过系统总线(system Bus)和主机内存(memory)、以及PCIe RC相连接，PCIe RC通过PCIe总线（Pcie Bus）直接连接了GPU和PCIe Switch，Switch又连接了多个外设（比如：网卡，DPU等）。

![](attachments/Pasted%20image%2020250322153022.png)

## 各个组件
### CPU

CPU不用解释，需要注意图中的CPU内部集成了内存控制器Memory Controller，内存控制器主要负责处理CPU核发出的内存访问请求，对内存进行读写。

### PCIe RC

RC全称为Root Complex，它是PCIe总线树状结构的“根”节点，主要负责处理CPU对PCIe设备的控制请求、完成地址转换等工作。

在 PCI Express（PCIe）架构中，Root Complex 是连接 CPU 和 PCIe 设备的部分。它负责将 CPU 的请求转发到 PCIe 设备，并处理来自这些设备的中断和数据。Root Complex 通常包括 IOMMU，以便在处理设备请求时提供内存地址转换和保护功能。

### 系统总线(system bus)

CPU、内存控制器和PCIe RC是由高速总线连接到一起的，一般称这个总线为系统总线。

> 注：请注意它们三者(CPU、内存控制器和PCIe RC)的关系也不一定跟上图完全一样。PCIe规范没有对RC具体怎么实现做出规定，因此RC的概念比较模糊。有的芯片架构中，PCIe RC被集成到了CPU内部，而且内存控制器也可以被划分到PCIe RC中。

### PCIe Switch

虽然PCIe RC上面可以直接连接终端设备（PCIe中称为EP，End Point），但是数量是有限的。所以一般情况下只有GPU直接连接到PCIe RC上，其他的外设通过PCIe Switch进行连接。PCIe Switch跟网络中的交换机的作用差不多，下面也可以增加更多层级，从而连接更多的PCIe EP到CPU上。

### PCIe EP
终端设备（PCIe中称为EP，End Point），就是通过Pcie连接的外设设备，比如网卡等。

PCIe总线上可能会连接可种各样的设备，这里我们只列出了NIC和HCA。

#### NIC
Network Interface Controller，网络接口控制器，也就是我们常说的网卡，插上网线并进行配置之后就可以接入网络了。

#### HCA(RNIC)

它就是我们关注的重点，即支持RDMA技术的网卡。
在`Infiniband/RoCE`规范中，将RDMA网卡称为HCA，全称为`Host Channel Adapter`，即主机通道适配器；
而在`iWARP`协议族中，将RDMA网卡称为RNIC，全称为`RDMA enabled Network Interface Controller`，即支持RDMA的网络接口控制器。


# 地址概述
既然RDMA技术的核心是内存，自然离不开如何对内存进行寻址、也就是如何访问的问题。RDMA技术中的**内存既要供软件（或者说CPU）访问，也要供网卡访问（HCA/RNIC）**，这其中涉及多种地址，我们将在本节中讲解。

## 物流地址(PA)
**PA (physial address 物理地址)**
内存中的数据是按照字节连续排布的，每个字节都可以有一个索引，这个索引就是内存的地址。
![](attachments/Pasted%20image%2020250322155013.png)

如果我们的CPU想要访问内存，最朴素的想法就是CPU直接指定一个内存的地址就可以了，这个地址就是物理地址，即`Physical Address`，简称PA。

![](attachments/Pasted%20image%2020250322155022.png)

## 虚拟地址(VA)
**VA (virtual address 虚拟地址)**

### 程序访问物理地址存在的问题
直接使用物理地址虽然方便，但是在操作系统上直接用物理地址访问内存产生了一些问题，比如：

**地址之间不隔离**：
难以避免一个程序恶意写入另一个程序所使用的内存。

**内存容易不够用**：
当同时运行的程序比较多时，内存很容易就不够用了。

**内存使用效率低**：
即使当一个程序执行完毕后释放了自己的内存，它留下的“内存空洞”不太可能完全匹配另一个程序所需要的内存大小，可能会产生一些难以利用的“内存碎片”。

### 虚拟地址介绍
后来，人们设计出了虚拟地址来解决这些问题。虚拟地址和物理地址之间经过了一层转换，软件或者说CPU通过虚拟地址来访问内存，并不能看到真实的地址，而在中间起到转换作用的是一个专用的模块——**MMU（Memory Management Unit）**。

### 使用虚拟地址访问的流程

当CPU发起对一个虚拟地址的访问时，MMU会去查询页表，将虚拟地址转换为物理地址，这个过程如下图所示：
![](attachments/Pasted%20image%2020250322155208.png)

**一般情况下MMU都是被集成在CPU内部的**，所以以后的图中我们会把MMU放到CPU中。

**页表本身也储存在内存当中，每个进程都有一份，也就是每个进程都有一份虚拟-物理地址的映射关系**。当不同的进程访问相同的地址时，最终对应的内存的物理地址是不同的。

进程初始化的时候，页表的基地址会被储存在特殊的寄存器中，这个基地址是物理地址还是虚拟地址？——自然是物理地址，要不然CPU怎么在没有页表的情况下找到页表的基地址呢？

## MMU

![](attachments/Pasted%20image%2020250322155558.png)

MMU(Memory Management Unit)，即内存管理单元，是现代CPU架构中不可或缺的一部分

### 页表(page table)
MMU 可以通过页表（Page Table）实现虚拟内存管理。
页表是一种数据结构，记录了每个虚拟页和其对应的物理页之间的映射关系。
在使用虚拟内存的系统中，每个进程都有自己的虚拟地址空间，而这些虚拟地址空间被分割成许多物理内存页（通常大小为4KB或更大），而不是一整块连续的物理内存。因此，当进程需要访问某个虚拟地址时，需要将其翻译成对应的物理地址，翻译过程就是通过页表来完成的。

当CPU发出一个虚拟地址时，MMU会通过页表查找并将其转换为对应的物理地址。此外，MMU还可以通过页表实现内存保护和共享等功能，从而提高系统的安全性和效率。

![](attachments/Pasted%20image%2020250323213945.png)

#### 基本原理

页表的基本原理是将虚拟地址划分成一个**页号**和一个**偏移量**。页号用于在页表中查找对应的物理页帧号，而偏移量则用于计算该虚拟地址在物理页帧中的偏移量。通过这种方式，就可以将虚拟地址映射到物理地址，使得进程可以访问对应的内存区域。



### 缺页中断（Page Fault）
Linux 利用虚拟内存极大的扩展了程序地址空间，使得原来物理内存不能容下的程序也可以通过内存和硬盘之间的不断交换（把暂时不用的内存页交换到硬盘，把需要的内存页从硬盘读到内存）来赢得更多的内存，**看起来就像物理内存被扩大了一样**。

事实上这个过程对程序是完全透明的，程序完全不用理会自己哪一部分、什么时候被交换进内存，一切都有内核的虚拟内存管理来完成。当程序启动的时候，Linux内核首先检查CPU的缓存和物理内存，如果数据已经在内存里就忽略，如果数据不在内存里就引起一个缺页中断（Page Fault），然后从硬盘读取缺页，并把缺页缓存到物理内存里。

#### 分类
缺页中断可分为主缺页中断（Major Page Fault）、次缺页中断（Minor Page Fault）：
- 要从磁盘读取数据而产生的中断是主缺页中断；
- 数据已经被读入内存并被缓存起来，从内存缓存区中而不是直接从硬盘中读取数据而产生的中断是次缺页中断。

#### 缺页中断的几种情况
缺页异常被触发通常有以下几种情况：

（1）程序设计的不当导致访问了进程的非法地址区域，SIGSEGV 信号杀死进程；
    
（2）访问的地址是合法的，但是虚拟内存地址 address 还未被映射过，该地址还未分配物理页框，其在页表中对应的各级页目录项以及页表项都还是空的，进程首次访问时触发（接下来重点要分析这种情况）；
    
（3）内存不足的状态下，即虚拟内存地址 address 之前被映射过，但是映射的这块物理内存（进程的匿名页/文件页）被内核 swap out 到磁盘上；
    
（4）虚拟内存地址 address 背后映射的物理内存还在，只是由于访问权限不够引起的缺页中断，比如：写时复制（COW）机制就属于这一种。


### MMU的功能
MMU主要包含以下几个功能：

#### 虚实地址翻译

在用户访问内存时，将用户访问的虚拟地址翻译为实际的物理地址，以便CPU对实际的物理地址进行访问。

#### 访问权限控制
可以对一些虚拟地址进行访问权限控制，以便于对用户程序的访问权限和范围进行管理，如代码段一般设置为只读，如果有用户程序对代码段进行写操作，系统会触发异常。

#### 物理内存管理
对系统的物理内存资源进行管理，为用户程序提供物理内存的申请、释放等操作接口。

### MMU带来的好处
#### 提升物理内存的利用率
##### 物理内存按需申请
面对众多用户同时发起的复杂任务请求，MMU采用灵活的内存管理策略，如按需分页，结合 MMU 的地址转换机制，确保**只有当前正在使用的内存页面被加载到物理内存中**，大大提高了内存利用率。

如代码段的内存在执行时进行映射和转换，进程fork后，t通过写时复制(Copy-On-Write)进行真正的物理内存分配。

##### 解决内存管理碎片化的问题
即在系统运行一段时间后，频繁的内存申请和释放会导致内存碎片化，无法申请到一块足够大的地址连续的内存。

#### 对内存地址的访问进行控制
如上述代码段只读权限控制，多线程的栈内存之间的空洞页隔离可以防止栈溢出后改写其他线程的栈内存，不同进程之间的地址隔离等等。

#### 将进程的地址空间隔离
不同进程之间可以使用相同的虚拟内存地址空间，而进程间的物理内存又可以做到隔离，这保证了进程的独立性同时，又简化了地址的访问方式，如在早期32位CPU上，为了支持4G以上的物理内存，一般物理地址有36-bit(如PowerPC-604系列)，但是用户的虚地址仍然使用32-bit，做法就是将用户的不同进程的32-bit虚地址在MMU转换时，转换为36-bit的物理地址，这样每个进程仍然能访问0-3G虚地址范围，将多个进程的3G空间映射到36-bit的物理内存空间中去。




## 地址空间
所有能够被索引的地址，就构成了一个地址空间。
所以我们可以说，所有CPU能够访问的虚拟地址构成了虚拟地址空间，所有物理地址构成了物理地址空间。
对于一个64位的操作系统来说，理论上CPU最大能够访问0~2^64 Bytes的地址空间，但是目前一般是只实现48位，也就是能够访问2^48=256TB的空间。

需要注意的是，地址空间里并不是物理地址都会给主机内存使用。比如物理地址空间中，有一部分物理地址会映射给BIOS，有一部分物理地址会映射给PCIe设备。

BIOS大家应该都知道，就是硬件上电之后先执行的一小段程序，负责对包括内存在内的一些硬件进行初始化，之后会引导操作系统启动；


## MMIO
MMIO（Memory-Mapped I/O，内存映射I/O）。
MMIO是一种用于进行输入输出操作的技术。它通过将外围设备的寄存器或者设备内存映射到计算机的内存地址空间，使得CPU可以通过访问内存，使用和访问这些外设的寄存器或者或者设备内存。
在MMIO中，设备的控制和数据寄存器被分配一段连续的内存地址，并且可以通过读取和写入这些内存地址来与外部设备进行通信。

比如：**CPU通过MMIO向RNIC发送消息来启动网络操作（这个过程即是Doorbell机制）**。

前文中说的把一部分物理地址映射给PCIe设备，指的就是当CPU发起对MMIO地址的读写操作时，会被PCIe控制器接管，然后转化为对PCIe总线上连接的设备的访问请求。而最终转化对设备寄存器的读写，还是对设备内部存储空间的读写，是由设备注册时的配置决定的。

### MMIO和IOMMU
MMIO是让CPU通过内存地址访问设备寄存器，进而给设备下发命令或者执行操作。
而IOMMU是让设备在访问内存时通过地址转换得到设备的物理地址来进行访问。
换句话说，**MMIO是CPU到设备的访问，而IOMMU是设备到内存的访问**。
它们可能在某些情况下一起使用，比如当一个设备需要通过MMIO被CPU访问，同时该设备执行DMA时又需要IOMMU来转换地址。

![](attachments/Pasted%20image%2020250319104327.png)

## Pcie总线地址(Pcie bus address)
CPU访问MMIO空间之后经过PCIe RC转化的地址，就是（PCIe）总线地址。外设也可以通过PCIe总线地址访问其他挂在PCIe总线上的设备，所有这些总线地址构成了总线地址空间。

CPU要访问总线地址空间中的地址，是需要PCIe RC将物理地址转换为总线地址的，画图来表示就是：

![](attachments/Pasted%20image%2020250323123221.png)
```bash
如上，CPU通过虚拟地址访问寄存器的总线地址
```

## IOMMU
### 背景
虽然**外设可以通过物理地址直接访问内存**，但是也产生了跟CPU通过物理地址访存类似的问题，其中对于外设最重要的一点便是：**有些设备可能会需要大量地址连续的物理内存**，比如HCA/RNIC就会耗费许多内存来放置软硬件进行交互的队列，以及队列的属性等等。

虽然可以通过自己设计**多级寻址**的方式来解决这个问题，但是这大大增加了软硬件的复杂度。那么能不能把MMU的设计思想也用在外设上呢？当然可以，于是就产生了IOMMU(x86平台)/SMMU(ARM平台)这种专门用于给外设进行地址翻译的设备。

### 介绍
**IOMMU (Input and Output Memory Management Unit： I/O内存管理单元)**，其名称显然是由MMU衍生而来。

![](attachments/Pasted%20image%2020250323124113.png)

IOMMU是DMA（直接内存访问，即设备与内存直接通信，而无需经过CPU）过程中的一个环节。更多时候会**把IOMMU看作一种机制**，从这个角度，我们也可以说：**IOMMU是DMA的一种实现方式**。

IOMMU 是**专为硬件设备设计的“内存翻译器”**；给需要「地址转换」才能访问某段 memory 的 I/O device 用的。
这个 memory 可能是 CPU 侧的 system memory，对于 GPU 设备来说， 还可能是自带的「显存」（一般叫 graphic memory 或 video memory）。

### IOMMU和MMU

- 传统MMU（内存管理单元）负责将**CPU**的**虚拟地址**转换为物理地址，让CPU能正确访问内存。
- IOMMU的作用类似，但服务对象是**显卡、网卡等硬件设备**。当这些设备需要直接访问系统内存时（如通过DMA），IOMMU会将**设备**使用的“虚拟地址”翻译成真实的物理地址。

![](attachments/Pasted%20image%2020250323130540.png)

由于 IOMMU 被多个 device 共享，所以相比被单个 CPU 独享的 MMU，需要有更高的并行处理能力。


### IOMMU和DMA的关系

**IOMMU与DMA协同工作**。
DMA允许设备直接访问内存，而IOMMU负责在此过程中进行地址翻译和权限控制。

### 其他
#### **（1）IOMMU物理硬件位于哪个位置？**
A：在CPU package的uncore部分，或者说我们常说的RC（Root Complex）中；
> 在 PCI Express（PCIe）架构中，Root Complex 是连接 CPU 和 PCIe 设备的部分。它负责将 CPU 的请求转发到 PCIe 设备，并处理来自这些设备的中断和数据。Root Complex 通常包括 IOMMU，以便在处理设备请求时提供内存地址转换和保护功能。

![](attachments/Pasted%20image%2020250326115222.png)

#### **（2）一个CPU有几个IOMMU单元？**
A：有多个，这个与物理root port是1：1的关系，即每个CPU的物理PCI口都有一个独立的IOMMU单元

#### **（3）一个IOMMU单元与PCI设备的管理关系是什么？**

A：每个IOMMU单元负责root port下挂载的所有设备，包括switch，PF，以及PF生成的所有VF。

![](attachments/Pasted%20image%2020250323134229.png)


### 为什么需要IOMMU
#### 虚拟化与安全性
- 在虚拟化场景中，**设备可能被分配到虚拟机使用**。IOMMU能确保设备只能访问虚拟机对应的物理内存，而非整个主机的内存。

- 某些硬件协议（如Firewire）存在安全风险，IOMMU可限制其直接访问全部物理内存。或者通过配置，禁止设备访问其他设备或者进程的地址。

- IOMMU使得设备无法直接访问物理地址，大大增加了设备进行DMA攻击的难度。

#### 提升性能
- 设备可能需要“连续的虚拟内存地址”，但物理内存可能是碎片化的。IOMMU能动态映射，让设备“看到”连续的地址空间。

#### 兼容性

- 例如，让32位设备访问64位系统的内存。


## IO虚拟地址(IOVA)

有了IOMMU/SMMU之后，外设跟使用MMU的CPU一样，看到的是一整片连续的虚拟地址。区别于CPU的VA，我们称这些地址为IO虚拟地址，即IOVA（Input/Output Virtual Address）。

IOMMU/SMMU记录着地址的转换关系，外设发出的IOVA，会被它翻译为PA，根据SMMU/IOMMU的位置不同，可能有两种情况：它有可能在设备内，也有可能挂在PCIe总线上，为多个EP提供地址翻译功能，这两种情况分别如下：

![](attachments/Pasted%20image%2020250323132735.png)
```bash
外设通过IOVA访问内存（1）
```

![](attachments/Pasted%20image%2020250323132744.png)
```bash
外设通过IOVA访问内存（2）
```

### IOVA和 Pcie Bus Address

大家可能对IOVA和Pcie Bus Address的关系表示困惑，这两个地址可以认为是相等的。
概念的差别在于：
当我们强调SMMU/IOMMU的输入时，也就是被转换前的地址时，一般将地址称为IOVA；
当我们强调在总线上的地址时，比如PCIe，我们一般将使用的地址称为Bus Address。
不必纠结它们的关系，我们RDMA领域更常使用IOVA的概念。


## DMA地址

![](attachments/Pasted%20image%2020250323231504.png)

外设（PCIe EP）通过DMA访问内存时，发出的地址就是DMA地址。
结合上面的描述我们可以知道，**当设备使用了IOMMU/SMMU时，DMA地址是IO虚拟地址IOVA。当未使用IOMMU/SMMU时，DMA地址是物理地址PA。**



# DMA
## 背景
外设如何访问内存呢？

有些设备不具备直接访问内存的能力，这个时候就需要CPU来帮助实现。下图中的例子中，如果想要把内存中的数据写入硬件内部的存储空间，需要CPU先把内存中的数据读入寄存器，再写入硬件。

![](attachments/Pasted%20image%2020250323123505.png)
```bash
如上，在没有DMA的情况下外设访问内存
```
但是毕竟CPU的主要功能是计算，不能总让它做这种搬运数据的苦力活。所以后来诞生了DMA（Direct Memory Access），即直接存储器访问。DMA是一个数据搬运工，硬件可以通过它读写内存，DMA一般会被集成到设备当中。

![](attachments/Pasted%20image%2020250323123536.png)
```bash
如上，在有DMA的情况下外设访问内存
```
外设发出DMA访存请求时，会在PCIe总线上发出总线地址，而一般情况下这个总线地址的值等于物理地址，我们可以认为外设发出的就是物理地址。这个请求被PCIe RC收到之后，它会通过系统总线来访问内存。

![](attachments/Pasted%20image%2020250323123619.png)
```bash
如上，外设通过物理地址访问内存
```



## 什么是DMA？

DMA （Direct Memory Access：直接内存访问）技术是一种不需要CPU参与，就能实现host外设比如pcie设备对host memory的访问。

有了DMA机制之后，外设跟主存之间的数据交互基本都是由dma controller完成，从而避免了cpu的参与，降低cpu使用率。

![](attachments/Pasted%20image%2020250315230136.png)

DMA就是我们在主板上放一块独立的芯片。在进行内存和 I/O 设备的数据传输的时候，我们不再通过 CPU 来控制数据传输，而直接通过 DMA 控制器（DMA Controller，简称 DMAC）。这块芯片，我们可以认为它其实就是一个协处理器（Co-Processor）。

## DMA有什么用？

DMAC最有价值的地方体现在，当我们**要传输的数据特别大、速度特别快**；或者**传输的数据特别小、速度特别慢**的时候。

例如，我们用千兆网卡或者硬盘传输大量数据的时候，如果都用 CPU 来搬运的话，肯定忙不过来，所以可以选择 DMAC。而当数据传输很慢的时候，DMAC 可以等数据到齐了，再发送信号，给到 CPU 去处理，而不是让 CPU 在那里忙等待。

在 DMAC 控制数据传输的过程中，我们还是需要 CPU 的进行控制，但是具体数据的拷贝不再由 CPU 来完成。

### 没有DMA的情况
原本，计算机所有组件之间的数据拷贝（流动）必须经过 CPU，如下图所示：

![](attachments/Pasted%20image%2020250313204956.png)

### 存在DMA的情况
DMA 代替 CPU 负责内存与磁盘以及内存与网卡之间的数据搬运，CPU 作为 DMA 的控制者，如下图所示：
![](attachments/Pasted%20image%2020250313205014.png)

## DMA的实现简述

在实现DMA传输时，是由DMA控制器直接掌管总线，因此，存在着一个总线控制权转移问题。即DMA传输前，CPU要把总线控制权交给DMA控制器，而在结束DMA传输后，DMA控制器应立即把总线控制权再交回给CPU。
CPU只是在每个数据块传输的开始和结束实施控制，在数据块的传输过程中则由DMA控制器控制。

DMA控制方式传送数据的过程可以分为三个阶段：预处理、数据传送、传送后处理。
### 预处理阶段

DMA控制器的命令/状态寄存器（CR）接收到CPU发出的I/O命令，将内存起始地址（输入数据到内存）或内存源地址（内存输出数据）写入DMA控制器的内存地址寄存器（MAR），同时将要传送的字（字节）数存入DMA控制器的字计数器（DC）。随后将外设中的源地址（或目标地址）直接送入 DMA控制器的I/O控制逻辑，然后启动DMA控制器进行数据传送。


### 数据传送阶段

CPU将总线控制权让给DMA控制器，并且在数据传送期间不再使用总线。DMA控制器按照内存地址寄存器（MAR）的指示，不断在设备与内存之间进行数据传输，并随时修改内存地址寄存器（MAR）和字计数器（DC）的值。

当一个数据块传输完毕或数据计数器的值减少到0时（所有数据都已传输完毕），传输停止并且向CPU发出中断信号。

### 传送后处理阶段

CPU响应DMA控制器的中断请求。如果数据传送完成，则转向相应的中断处理程序进行后续处理；如果还有数据需要传送，则按照相同方法重新启动剩余数据的传送。

在DMA控制方式中，CPU只是在每个数据块传输的开始和结束实施控制，在数据块的传输过程中则由DMA控制器控制。


##  DMA的scatter/gather(SG-DMA)

Scatter：离散
Gather：聚合

Linux 在内核 2.4 版本里引入了 DMA 的 scatter/gather – 分散/收集功能，只要确保 Linux 版本高于 2.4 即可。

### 场景1：Scatter Transfer
**场景1：将一片连续内存数据搬运到一片不连续的的内存空间（且间隔是相等的）**
源内存：连续
目的内存：离散
这个时候就可以用到`DMA`的 `Scatter Transfer`

![](attachments/Pasted%20image%2020250313202630.png)


### 场景2：Gather transfer
**场景2：将一片内存区域中等间隔的多段数据拷贝到一段连续内存中。（常见的2D矩形抠图就是这种场景的典型应用）**

源内存：分散
目的内存：聚合
这种场景就可以使用DMA的`Gather transfer`

![](attachments/Pasted%20image%2020250313202822.png)


### scatter/gather DMA 与 block DMA

在DMA传输数据的过程中，要求源物理地址和目标物理地址必须是连续的。但是在某些计算机体系中，如IA架构，连续的存储器地址在物理上不一定是连续的（CPU以虚拟地址寻址），所以DMA传输要分成多次完成。

#### block DMA
DMA控制器在传输完一块物理上连续的数据后引起一次中断，然后再由CPU触发进行下一块物理存储空间上的数据传输，这种方式被称为 Block DMA。

传统的block DMA像这样：
![](attachments/Pasted%20image%2020250313224146.png)

#### scatter/gather DMA
**Scatter-gather DMA(分散聚集式直接内存访问)** 方式则不同，将物理上分散的多个不连续存储空间以链表（struct scatterlist，链表或数组实现）形式组织起来，然后把链表首地址告诉`DMA master`。`DMA master`在传输完一块物理连续的数据后，不用发起中断，而是根据链表来传输下一块物理上连续的数据，直到传输完毕后再发起一次中断。

先进的scatter-gather DMA像这样：
![](attachments/Pasted%20image%2020250313224150.png)

即：`scatter-gather DMA`允许一次传输多个物理上不连续的块，完成传输后只发起一次中断。这样做的好处是大大减少了中断的次数，提高了数据传输的效率。



### scatter/gather DMA的应用

![](attachments/Pasted%20image%2020250313224208.png)
```c
.txmode = { 
    ...                                                  
    .offloads = DEV_TX_OFFLOAD_MULTI_SEGS,
    ...
}
```

Scatter-Gather DMA允许网卡从多个不连续的内存区域收集数据，合并成一个数据包发送出去；或者反过来，接收的数据分散到不同内存位置。在发送数据包时，如果数据分散在多个缓冲区，SG-DMA能够有效地处理这些分散的数据，提升性能。

> dpdk在ip分片的实现中，采用了一种称作零拷贝的技术。而这种实现方式的底层，正是由`scatter/gather DMA`支撑的。dpdk的多个分片包采用了链式管理，同一个数据包的多个分片，分散存储在不连续的块中（mbuf结构）。这就要求DMA一次操作，需要从不连续的多个块中搬移数据到网卡，然后发送出去。附上e1000驱动发包部分代码：
```c
uint16_t
eth_em_xmit_pkts(void *tx_queue, struct rte_mbuf **tx_pkts,
        uint16_t nb_pkts)
{
	...
	txq = tx_queue;
	sw_ring = txq->sw_ring;
	txr = txq->tx_ring;
	tx_id = txq->tx_tail;
	txe = &sw_ring[tx_id];
	...
    //e1000驱动部分代码
    for (nb_tx = 0; nb_tx < nb_pkts; nb_tx++) {
	    ...
	    tx_pkt = *tx_pkts++;
	    ...
	    m_seg = tx_pkt;
	    do {
	        txd = &txr[tx_id];
	        txn = &sw_ring[txe->next_id];
	 
	        if (txe->mbuf != NULL)
	            rte_pktmbuf_free_seg(txe->mbuf);
	        txe->mbuf = m_seg;
	 
	        /*
	        * DMA 映射：将数据包的物理内存地址映射到网卡的 DMA 引擎可访问的地址空间（IOVA或DMA地址）。
	        * Set up Transmit Data Descriptor.
	        * 需要配置描述符（Descriptor），
	        * 这些描述符告诉DMA引擎每个内存块的位置和长度。
	        */
	        slen = m_seg->data_len;
	        buf_dma_addr = rte_mbuf_data_iova(m_seg); //获取dma地址(iova地址)
	 
	        txd->buffer_addr = rte_cpu_to_le_64(buf_dma_addr);
	        txd->lower.data = rte_cpu_to_le_32(cmd_type_len | slen);
	        txd->upper.data = rte_cpu_to_le_32(popts_spec);
	 
	        txe->last_id = tx_last;
	        tx_id = txe->next_id;
	        txe = txn;
	        m_seg = m_seg->next;
	    } while (m_seg != NULL);
	 
	    /*
	    * The last packet data descriptor needs End Of Packet (EOP)
	    */
	    cmd_type_len |= E1000_TXD_CMD_EOP;
	    txq->nb_tx_used = (uint16_t)(txq->nb_tx_used + nb_used);
	    txq->nb_tx_free = (uint16_t)(txq->nb_tx_free - nb_used);
	    ...
    }
}
```

#### NIC是否支持SG-DMA

```bash
ethtool -k enp6s0
Features for enp6s0:
rx-checksumming: off [fixed]
tx-checksumming: off
        tx-checksum-ipv4: off [fixed]
        tx-checksum-ip-generic: off [fixed]
        tx-checksum-ipv6: off [fixed]
        tx-checksum-fcoe-crc: off [fixed]
        tx-checksum-sctp: off [fixed]
scatter-gather: on
        tx-scatter-gather: on
        tx-scatter-gather-fraglist: off [fixed]
tcp-segmentation-offload: off
        tx-tcp-segmentation: off [fixed]
        tx-tcp-ecn-segmentation: off [fixed]
        tx-tcp-mangleid-segmentation: off [fixed]
        tx-tcp6-segmentation: off [fixed]
udp-fragmentation-offload: off
generic-segmentation-offload: on
generic-receive-offload: on
large-receive-offload: off [fixed]
rx-vlan-offload: off [fixed]
tx-vlan-offload: off [fixed]
ntuple-filters: off [fixed]
receive-hashing: off [fixed]
highdma: on
rx-vlan-filter: on [fixed]
vlan-challenged: off [fixed]
tx-lockless: off [fixed]
netns-local: off [fixed]
tx-gso-robust: off [fixed]
tx-fcoe-segmentation: off [fixed]
tx-gre-segmentation: off [fixed]
tx-gre-csum-segmentation: off [fixed]
tx-ipxip4-segmentation: off [fixed]
tx-ipxip6-segmentation: off [fixed]
tx-udp_tnl-segmentation: off [fixed]
tx-udp_tnl-csum-segmentation: off [fixed]
tx-gso-partial: off [fixed]
tx-sctp-segmentation: off [fixed]
tx-esp-segmentation: off [fixed]
fcoe-mtu: off [fixed]
tx-nocache-copy: off
loopback: off [fixed]
rx-fcs: off [fixed]
rx-all: off [fixed]
tx-vlan-stag-hw-insert: off [fixed]
rx-vlan-stag-hw-parse: off [fixed]
rx-vlan-stag-filter: off [fixed]
l2-fwd-offload: off [fixed]
hw-tc-offload: off [fixed]
esp-hw-offload: off [fixed]
esp-tx-csum-hw-offload: off [fixed]
rx-udp_tunnel-port-offload: off [fixed]
```

# 参考
```bash
# RDMA之内存地址基础知识
https://zhuanlan.zhihu.com/p/463199854

# RDMA为什么要Pin内存？
https://mp.weixin.qq.com/s/6ESmh8RKAAdW5nm3J2a9VQ
```