```table-of-contents
```
# PCI介绍
一个远古的时代，Intel就推出了一种**共享总线**协议，用于当时的CPU的I/O扩展，最初版本是PCI 1.0 （全称是Peripheral Component Interconnect：外设部件互连）。从这个英文全称不难看出一二，这个总线的定位就是外设的互联。

PCI 1.0总线频率仅为33MHz，位宽有32bit，根据小学算术可得PCI 1.0的带宽为133M Byte/s
从PCI1.0到PCI2.0，带宽就实现了翻倍，带宽翻倍这个词一直是PCI协议发展的灵魂，延续到现在。直到PCI-X 2.0 当时的带宽已经达到2133MByte/s。
```c
- PCI1.0, 133MB/sec，32bit, 32MHz
- PCI2.0, 533MB/sec，64bit, 66MHz
- PCI-X 1.0, 1066MB/sec，64bit, 133MHz
- PCI-X 2.0, 2133MB/sec，64bit, 266MHz
```
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731192427.png)
## PCI架构图
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801233210.png)

## PCI总线
PCI总线是一种树型结构，并且独立于CPU总线，可以和CPU总线并行操作。PCI总线上可以挂接PCI设备和PCI桥，PCI总线上只允许有一个PCI主设备（同一时刻），其他的均为PCI 从设备，而且读写操作只能在主从设备之间进行，从设备之间的数据交换需要通过主设备中转。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731205139.png)
一个典型的33MHz的PCI总线系统如上图所示，处理器通过FSB与北桥相连接，北桥上挂载着图形加速器（显卡）、SDRAM（内存）和PCI总线。PCI总线上挂载着南桥、以太网、SCSI总线（一种老式的小型机总线）和若干个PCI插槽。CD和硬盘则通过IDE连接至南桥，音频设备以及打印机、鼠标和键盘等也连接至南桥，此外南桥还提供若干的USB接口。

PCI总线是一种**共享总线**，所以需要特定的仲裁器（Arbiter）来决定当前时刻的总线的控制权。一般该仲裁器位于北桥中，而仲裁器（主机）则通过一对引脚，REQ#（request） 和GNT# （grant）来与各个从机连接。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731205450.png)

PCI Spec规定了每个PCI总线上最多可以连接多达32个PCI设备，但是实际上却远远达不到32个，33MHz的32位PCI总线一般只能连接10到12个负载。
>注：如果使用插槽连接，则一个连接算两个PCI设备，插槽和PCI卡分别算作一个PCI设备。也就是说一个33MHz的PCI总线最多只能连接4到5个插槽即PCI卡。
如果需要连接更多的PCI设备，则需要借助PCI-to-PCI桥，每个桥的内部都有隔离，这保证了每个桥又可以连接额外的10~12个负载。一个包含PCI-to-PCI桥的33MHz PCI总线系统的架构图如下所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731212521.png)

## PCI总线的三种传输模式
PCI Spec规定的三种数据传输模型：Programmed I/O（PIO），Peer-to-Peer和DMA。
三种数据传输模型的示意图如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731212657.png)
**Programmed I/O（PIO）**
PIO在早期的PC中被广泛使用，因外当时的处理器的速度要远远大于任何其他外设的速度，所以PIO足以胜任所有的任务。举一个例子，比如说某一个PCI设备需要向内存（SDRAM）中写入一些数据，该PCI设备会向CPU请求一个中断，然后CPU首先先通过PCI总线把该PCI设备的数据读取到CPU内部的寄存器中，然后再把数据从内部寄存器写入到内存（SDRAM）中。现在看来，这种传输方式的效率还是很低的。首先，每次CPU和PCI设备以及SDRAM通信都需要额外的时钟周期（相对于DMA）；其次，这种传输方式还需要长时间地占用CPU，影响CPU的使用率。试想一下，你在用PC在线观看一个1080p60的高清视频，这需要以太网连续地向内存（SDRAM）中写入数据，如果使用PIO的方式的话，将难以保证数据的写入速度。随着目前的PCI外设速度越来越高，PIO已经逐渐被DMA传输方式所取代，但是为了兼容早期的一些设备，PCI Spec依然保留了PIO。

**DMA，即Direct Memory Access** 
DMA是一种在传输过程中，几乎不需要CPU进行干预的数据传输方式。如上面的图片所示，以太网可以直接向内存（SDRAM）中写入数据，而几乎不需要CPU的干预。实际上，DMA不仅仅应用于PCI总线系统中，它是一种更为广泛应用的数据传输方式。目前，几乎所有的CPU，甚至是MCU都支持DMA。

**Peer-to-Peer**
前面的文章中，我们介绍过PCI总线系统中的主机身份并不是固定不变的，而是可以切换的（借助仲裁器），但是同一时刻只能存在一个主机。完成Peer-to-Peer这一传输方式的前提是，PCI总线系统中至少存在一个有能力成为主机的设备。在仲裁器的控制下，完成主机身份的切换，进而获得PCI总线的控制权，然后与总线上的其他PCI设备进行通信。不过，需要注意的是，在实际的系统中，Peer-to-Peer这一传输方式却很少被使用，这是因为获得主机身份的PCI设备（Initiator）和另一个PCI设备（Target）通常采用不同的数据格式，除非他们是同一个厂家的设备。


# PCIe协议的三层
和很多的串行传输协议一样，一个完整的PCIe体系结构包括应用层、事务层（Transaction Layer）、数据链路层（Data Link Layer）和物理层（Physical Layer）。
其中，应用层并不是PCIe Spec所规定的内容，完全由用户根据自己的需求进行设计，另外三层都是PCIe Spec明确规范的，并要求设计者严格遵循的。

PCIe协议分为3层
- Transaction Layer （事务层）
- Data Link Layer （数据链路层）
- Physical Layer （物理层）
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731205829.png)
如上图所示，其中**Device Core and interface to Transaction Layer**就是我们常说的**应用层**或者**软件层**。这一层决定了PCIe设备的类型和基础功能，可以由硬件（如FPGA）或者软硬件协同实现。如果该设备为Endpoint，则其最多可拥有8项功能（Function），且每项功能都有一个对应的配置空间（Configuration Space）。如果该设备为Switch，则应用层需要实现包路由（Packet Routing）等相关逻辑。如果该设备为Root，则应用层需要实现虚拟的PCIe总线0（Virtual PCIe Bus 0），并代表整个PCIe总线系统与CPU通信。

![](../限速%20&%20流控/attachments/Pasted%20image%2020230731210621.png)
每一层都有不同的责任，以发送方向简单来说
- Transaction Layer
负责数据（Payload) 和控制指令的组包，该层的数据包叫TLP（Transaction Layer Packet，TLP），组包完成后发给Data Link Layer。接收端的事务层负责事务层包（Transaction Layer Packet，TLP）的解码与校检，发送端的事务层负责TLP的创建。此外，事务层还有QoS（Quality of Service）和流量控制（Flow Control）以及Transaction Ordering等功能。

- Data Link Layer
接收来自Transaction Layer的TLP，负责数据的可靠传输；同时数据链路层负责数据链路层包（Data Link Layer Packet，DLLP）的创建，解码和校检。同时，本层还实现了Ack/Nak的应答机制。

- Physical Layer
物理层负责Ordered-Set Packet的创建和解码。同时负责发送与接收所有类型的包（TLPs、DLLPs和Ordered-Sets）。当前在发送之前，还需要对包进行一些列的处理，如Byte Striping、Scramble（扰码）和Encoder（8b/10b for Gen1&Gen2, 128b/130b for Gen3& Gen4）。对应的，在接收端就需要进行相反的处理。此外，物理层还实现了链路训练（Link Training）和链路初始化（Link Initialization）的功能，这一般是通过链路训练状态机（Link Training and Status State Machine，LTSSM）来完成的。

需要注意的是，在PCIe体系结构中，事务层，数据链路层和物理层存在于每一个端口（Port）中，也就是说Switch中必然存在一个以上的这样的结构（包括事务层，数据链路层和物理层的）。一个简化的模型如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731210436.png)

## PCIe总线的通信机制
假设某个设备要对另一个设备进行读取数据的操作，首先这个设备（称之为Requester）需要向另一个设备发送一个Request，然后另一个设备（称之为Completer）通过Completion Packet返回数据或者错误信息。

### 请求分类
在PCIe Spec中，规定了四种类型的请求（Request）：Memory、IO、Configuration和Messages。其中，前三种都是从PCI/PCI-X总线中继承过来的，第四种Messages是PCIe新增加的类型。详细的信息如下表所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731211112.png)

只有Memory Write和Message是Posted类型的，其他的都是Non-Posted类型的。
- 所谓Non-posted，就是Requester发送了一个包含Request的包之后，必须要得到一个包含Completion的包的应答，这次传输才算结束，否则会进行等待。
- 所谓Posted，就是Requester的请求并不需要Completer通过发送包含  Completion 的包进行应答，当然也就不需要进行等待了。很显然，Posted类型的操作对总线的利用率（效率）要远高于Non-Posted型。

那么为什么要分为Non-Posted和Posted两种类型呢？对于Memory Writes来说，对效率要求较高，因此采用了Posted的方式。但是这并不意味着Posted类型的操作完全不需要Completer进行应答，Completer仍然可采用另一种应答机制——Ack/Nak的机制（在数据链路层实现的）。**即Non-Posted和Posted是从事务层来看，是否需要发送事务层的Completion 包；而DDL层的ack/Nack都是存在的，保证可靠**

### TLP传输
TLP传输的示意图如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731211401.png)
TLP在整个PCIe包结构的位置如以下两张图所示：（第一张为发送端，第二张为接收端）
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731211438.png)
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731211501.png)

其中，TLP包的结构图如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731211541.png)
图中的TLP Digest即ECRC（End-to-End CRC），是可选项。此外，TLP的长度（包括其中的Header、Data和ECRC）是以DW（双字：double word，即四个字节）为单位的。


# 事务层
## TLP格式
对于任意的一个TLP来说，除了Data Payload，还有物理层添加的包头（1 Byte）、数据链路层添加的包编号（2 Bytes）、事务层添加的包头（12 or 16 Bytes）、事务层添加的ECRC（4Bytes，可选的）、数据链路层添加的LCRC（4Bytes）和物理层添加的包尾（1 Byte）。

具体如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731171630.png)

## MPS对于传输的影响
如果以3DW的事务层包头来计算，且不添加ECRC，则该TLP至少含有20 Bytes的额外数据（除了Data Payload之外的）。我们姑且称之为TLP Overhead。

**如果只从TLP Overhead考虑的话，显然每个TLP包含的Data Payload的量越大，真实有效速率越高。**
然而，实际应用中却并非如此，因为链路上的其他因素也在影响实际的真实有效速率。PCIe Spec规定，任何TLP都不允许被Order Sets或者DLLP打断。也就是说，Skip Order Sets和FC DLLP、Ack/Nak DLLP都只能在两个TLP之间发送。
换一句话说，Data Payload越大，TLP的也就越长，为了保证正常通信，两个TLP之间的时间间隔也就越大。这就是为什么Data Payload越大，但真实有效速率却未必会越高的原因。  

除了TLP Overhead之外，前面文章介绍的Ack/Nak机制和Flow Control机制都是需要花费时间的。这里我们分别称其所消耗的时间为Link Protocol Overhead和Flow Control Protocol Overhead。

**注：**显然，更低频率的Flow Control Update，会一定程度上提高真实有效速率，但这需要更大的Rx Buffer，从而带来更高的硬件成本开销。一般情况下，PCIe设备都应当遵循Spec所定义的FC Update周期计算方式，具体可可参考前面的文章：[http://blog.chinaaet.com/justlxy/p/5100053465](http://blog.chinaaet.com/justlxy/p/5100053465)

### MPS的协商
虽然PCIe Spec规定，TLP的Data Payload最高可达4096 Bytes，但同时也规定了PCIe体系结构中，所有的设备，都必须使用相同的Max_Payload_Size值。
换一句话说，整个总线的Max_Payload_Size值必须使用总线体系结构中所有设备所支持的最小的Max_Payload_Size值。具体如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731172200.png)
**注：**每个设备所支持的Max_Payload_Size最大值信息，存在于==Device Capability Register==中。

Max_Payload_Size值的设置是在PCIe总线枚举和配置的过程中完成的，软件确定了Max_Payload_Size的值后，会将该值写入到每个设备的==Device Control Register==中。

在不考虑Ack/Nak机制和Flow Control机制等因素的情况下，真实有效速率可以这样计算：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731172607.png)

则有：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731172614.png)

### PCIe的枚举Enumeration
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801123349.png)


## MRRS对于传输的影响
Maximun Read Request Size（即最大读请求），在PCIe总线配置的过程中，该值会被写入到每个设备的Device Control Register中。
PCIe Spec规定，Maximun Read Request Size的值可以超过Max_Payload_Size，例如，可以向Max_Payload_Size为128 Bytes的设备，一次请求读512 Bytes的数据。此时，一次请求会对应多个返回的Completion。

### MRRS的设置
Maximun Read Request Size的值也并非越大越好，该值设置的过大，会导致某个PCIe设备独占整个系统带宽的时间过长。
但是如果Maximun Read Request Size设置的过小，则需要发起多个读请求操作。

## RCB
RCB位在Link Control寄存器中定义。RCB位决定了RCB参数的值，在PCIe总线中，RCB参数的大小为64B或者128B，如果一个PCIe设备没有设置RCB的大小，则RC的RCB参数缺省值为64B，而其他PCIe设备的RCB参数的缺省值为128B。PCIe总线规定RC的RCB参数的值为64B或者128B，其他PCIe设备的RCB参数为128B。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801124934.png)
Read Completion Boundary（RCB）确定了针对读请求返回的每个Completion的Data Payload的最大值，一般为64 Bytes或者128 Bytes（由系统或者软件设置）。当然，Completion的Data Payload值，是可以小于RCB的。

以64 Bytes 的RCB和一次读256 Bytes的请求为例，可能的情况如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731173326.png)

**注：**目前，大部分的Root都固定地使用64 Bytes的RCB，尽管Max_Payload_Size的值可能是128或者更大。


在PCIe总线中，一个存储器读请求TLP可能收到目标设备发出的多个完成报文后，才能完成一次存储器读操作。因为在PCIe总线中，一个存储器读请求最多可以请求4KB大小的数据报文，而目标设备可能会使用多个存储器读完成TLP才能将数据传递完毕。
当一个EP向RC或者其他EP读取数据时，这个EP首先向RC或者其他EP发送存储器读请求TLP；之后由RC或者其他EP发送存储器读完成TLP，将数据传递给这个EP。
如果存储器读完成报文所传递数据的地址范围没有跨越RCB参数的边界，那么数据发送端只能使用一个存储器完成报文将数据传递给请求方，否则可以使用多个存储器读完成TLP。

- 范例
假定一个EP向地址范围为0xFFFF-0000~0xFFFF-0010这段区域进行DMA读操作，RC收到这个存储器读请求TLP后，将组织存储器读完成TLP，由于这段区域并没有跨越RCB边界，因此RC只能使用一个存储器读完成TLP完成数据传递。

如果存储器读完成报文所传递数据的地址范围跨越了RCB边界，那么数据发送端(目标设备)可以使用一个或者多个完成报文进行数据传递。数据发送端使用多个存储器读完成报文完成数据传递时，需要遵循以下原则。

- 第一个完成报文所传送的数据，其起始地址与要求的起始地址相同。其结束地址或者为要求的结束地址(使用一个完成报文传递所有数据)，或者为RCB参数的整数倍(使用多个完成报文传递数据)。
- 最后一个完成报文的起始地址或者为要求的起始地址(使用一个完成报文传递所有数据)，或者为RCB参数的整数倍(使用多个完成报文传递数据)。其结束地址必须为要求的结束地址。
- 中间的完成报文的起始地址和结束地址必须为RCB参数的整数倍。

当RC或者EP需要使用多个存储器读完成报文将0xFFFE-FFF0~0xFFFF-00C7之间的数据发送给数据请求方时，可以将这些完成报文按照表5‑9方式组织。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801153521.png)

表提供的方式仅供参考，目标设备还可以使用其他拆分方法发送存储器读完成TLP。PCIe总线使用多个完成报文实现一次数据读请求的主要原因是考虑Cache行长度和流量控制。在多数x86处理器系统中，存储器读完成报文的数据长度为一个Cache行，即一次传送64B。除此之外，较短的数据完成报文占用流量控制的资源较少，而且可以有效避免数据拥塞。

## 可靠传输
可靠传输，可以分为两个方面

- 确保PCIe Link两端的数据传输不会因为SerDes的误码率而出错，主要利用CRC校验和重传（Replay）的机制。
- 在数据传送正确的基础上，确保Link两端在发送TLP时候，确保对方有能力接收数据，防止对方因为没有空间来接收数据导致数据的丢失。这就是流量控制（Flow Control）

从这个分层结构可以看出，Flow Control的目的是为了确保对方有能力接收TLP。TLP在Transaction Layer, TLP的发送首先达到Data Link Layer，**所以Flow Control的机制存在于Data Link Layer, 是直接用来管控TLP流向Data Link Layer的“红绿灯”**

## Flow Control/FC
流量控制的机制最初来自于互联网中，在一个网络中主要包含两类资源，数据通路和数据缓冲。数据通路是最珍贵的资源，它决定了网络的最大带宽。数据缓冲区也很重要，当数据在网络上传输时，它是从一个节点传到另一个节点时中间需要经过若干节点才能到达目的地。每个节点中都含有缓冲区，暂存这个节点中没有处理完的数据。网络设备使用这些缓冲区可以搭建数据传送的流水线从而提高数据传输性能。最初在网络节点中只为一条链路提供了一个缓冲区，后来有引入了多个多虚拟通路（VC： virtual channel）缓冲区，不同的报文可以使用不同的通路进行传递，从而提高了传输效率。

目前的流量控制的实现主要是基于多通道技术，它的主要作用是合理的利用物理链路，避免因接收端缓冲区不足导致数据包的丢弃和重发，从而有效的使用网络带宽。它的核心原理都是通过根据接收端缓冲区的容量，向发送端提供反馈，发送端根据该反馈决定发送多少数据。

### 目的
PCIe总线采用Flow Control的目的是，保证发送端的PCIe设备永远不会发送接收端的PCIe设备不能接收的TLP（事务层包）。也就是说，发送端在发送前可以通过Flow Control机制知道接收端能否接收即将发送的TLP。

### 引入FC的背景
对于大部分的串行传输协议而言，发送方能够有效地将数据发送至接收方的前提是，接收方有足够的接收Buffer来接收数据。在PCI总线中，发送方在发送前并不知道接收法是否有足够的Buffer来接收数据（即接收方是否就绪），因此经常需要一些Disconnects和Retries的操作，这将会严重地影响到总线的传输效率（性能）。

PCIe总线为了解决这一问题，提出了Flow Control的概念，如下图所示。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220244.png)
>注：采用Flow Control机制的PCIe总线，相对于PCI总线获得了更高的总线利用率。虽然增加了Flow Control DLLP，但是这些DLLP对带宽的占用极小，几乎对总线利用率没有什么影响。

PCIe Spec规定，PCIe设备的每一个端口（Ports）都必须支持Flow Control机制，在发送TLP之前，Flow Control必须先检查接收端口是否有足够的Buffer空间来接收这个TLP。当PCIe设备支持多个VC（Virtual Channel）时，Flow Control机制可以显著地提高总线的传输效率。

PCIe Spec规定，每个PCIe端口最多支持8个VC，并且每个VC的Flow Control Buffer是完全独立的。也就是说，某一个VC的Flow Control Buffer满了，并不会影响其他的VC的通信。

**注：**一般Endpoint只有一个端口，Root有一个或者多个端口，Switch有一个Upstream端口和多个Downstream端口。

### 原理
Flow Control机制是通过相邻两个端口（Ports）的数据链路层之间发送DLLP（Flow Control DLLPs）来实现的。也就是说Flow Control是一种点到点（Point to Point）的方式，而非端到端（End to End）。
> 两个PCIe设备的通信是 End to End, 但是具体的Flow control则是 Point to Point ， 怎么理解？
> 我的理解是：
> FC是一个发送队列到另外一个接受队列，可以理解为socket的发送缓冲区、接受缓冲区。
> DDL层相对于是TCP层，TLP层相对于是socket层。

在进行初始化的时候，接收端需要向发送端报告（reports）其Buffer的大小，在正常运行状态（Run-time）时，会周期性地通过Flow Control DLLPs来告知发送端，接收端的各个Buffer的大小。

需要注意的是，虽然Flow Control DLLP只在相邻的数据链路层之间传输，但是相关的Buffer和计数器（FC Counter）确是在事务层（Transaction Layer）的，即事务层参与了Flow Control机制的管理。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731183442.png)
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731183524.png)

### DDLP
Flow control过程需要两个层次的参与，Transaction Layer包含Counter和Data Link Layer产生FC DLLP给对端。

![](../限速%20&%20流控/attachments/Pasted%20image%2020230731183838.png)
在PCIe总线中，当一个发送端发送数据报文给接收端时，它需要保证接收端有足够的缓冲区。为了能够知道接收端还有多少可用的缓冲区，接收端需要通过DLLP (Data Link Layer Packet）随时汇报可用的缓冲区。

#### 发送DDLP
Flow Control的对话在Data Link Layer定期发生：
- 对方：到点啦，现在9点钟，我看看我这边的Buffer，数数看还有10个Data Credit和3和Header Credit，于是Data Link 层就组建个Flow Control DLLP，带着10个Data Credit和3和Header Credit信息，发出去，biu~
- 自己：收到DLLP，拆开一看，发现对方还有10个Data Credit和3个Header Credit，记到自己的账本上
- 自己：呀！Transaction Layer需要发送一个TLP，需要占用1个Header Credit和4个Data Credit
- 自己：Transaction Layer看看Data Link Layer刚刚收到的Credit信息，再看看账本，那对方空间足够呀，绿灯亮了！发起来！发完后，还要悄悄的记一笔帐，对方还剩下6个Data Credit和2个Header Credit
- 对方：收到了新Trasaction Layer数据包啦，先放进Buffer里，此时Buffer里也只剩下6个Data Credit和2个Header Credit了，但是看看表，还没到10点啊，我现在还不能发，只能干着急
- 自己：呀！Transaction Layer需要发送一个长的TLP，需要占用1个Header Credit和10个Data Credit，再看看账本，Credit不够了啊，红灯亮了，那等着吧
- 对方：Buffer里面的数据都处理完啦，看看此时Buffer里剩下20个Data Credit和5个Header Credit，再看看表，到10点了啊，于是Data Link 层又组建个Flow Control DLLP，带着最新的Credit信息，发出去，biu~
- 自己：又收到DLLP，拆开一看，发现对方现在已经有20个Data Credit和5个Header Credit啦，那更新下账本，对方还剩下20个Data Credit和5个Header Credit （实际上记账方式是利用两边计数差的方式，为了便于理解不详细展开了）
- 自己：刚刚不是有个TLP堵住了嘛！现在好了，绿灯可以亮了！发起来！发完后，还要悄悄的记一笔帐，对方还剩下个10Data Credit和4个Header Credit
- .就这样每到整点，对方都会更新自己的Credit信息，这样周而复始，度过了一天又一天

#### 发送DDLP的间隔/Flow Control Update Latency
从上面的过程中可以看出，这个定时发送Credit信息的间隔至关重要。隔得时间太长了呢，本来Buffer都空了，结果还不能发新的TLP；隔得时间太短了呢，DLLP本身也会消耗链路的带宽，影响带宽利用率。于是乎，PCIe给出了一个计算公式，感兴趣的童鞋可以查阅PCIe Spec的最后的章节“Flow Control Update Latency”

![](../限速%20&%20流控/attachments/Pasted%20image%2020230731190841.png)

TLPOverhead为28B，包含STP(1B), Sequence(2B),TLP header(16B),ECRC(4B),TLP Prefix(4B), END(1B)

UF为UpdateFactor：
Gen1 （2.5GT/s）如下表所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731191118.png)
Gen2（5GT/s）如下表所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731191133.png)
Gen3 (8GT/s）如下表所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731191144.png)
根据公式计算出的Max UpdateFC Latency计算结果如下:
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731191327.png)

## 流控算法
Flow Control Buffer：是事务层虚拟通道的即Receiver Buffer，其存储单元（Unit）被称作Flow Control Credits。

Buffer的内容释放代表着对方Transaction Layer把TLP发送给相关应用处理完成；这个Buffer的内容增加代表对方Transaction Layer收到了新的TLP。PCIe规定了一种计算Buffer剩余空间的单位：Credit。1个Credit代表4DW的TLP Payload 或者1个TLP Header，分别叫Header Credit和Data Credit。
> DW: 即double word,即4Byte. 4DW即4B*4=16B.

PCIe总线使用的流控算法叫做基于信用的流量控制机制（Credit-based Mechanism）。接收端的可用缓冲数量使用信用积分来表示，接收端通过不断的发送DLLP告知接收端VC buffer的信用数，当缓冲区用尽时发送端会停止发送TLP以防数据包丢弃和重发。

>注：发送端也会有Buffer，缓存一定数量的TLP。一般会根据P, NP, C分为三个buffer，然后根据Credit和PCIe的Ordering Rule，来仲裁选择下一个可以发送的TLP。

### TLP
PCIe设备发送的数据是以TLP形式发出的。
#### TLP分类
TLP一共有三大类：**Posted Transactions**（包括Memory Writes和Messages）、**Non-Posted Transactions**（包括Memory Reads、Configuration Reads and Writes、IO Reads and Writes）以及**Completions**（包括Read and Write Completion）。

> 注：
> 我理解的：
> 收包时：网卡的DMA将数据包从网卡的缓存发送到内存中，应该是memory write.
> 发包时：网卡的DMA将数据包从内存中移到网卡的缓存中，应该是Memory Reads。
> 另外，收包时，网卡的DMA从ring buffer中获取描述符，应该是 Memory Reads。

#### TLP组成
PCIe设备发送的数据是以TLP形式发出的，TLP通过数据链路层，而到达数据缓存时被分解为Header和Data两个部分，分别存放到不同的接收缓冲队列中。
Flow Control为了获得更高的数据传输效率，将这三类TLP分开存放，同时将Header与Data部分也分开存放。因此，一共存在六种不同的Flow Control Buffer类型，如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731184201.png)

1. PH缓存存放Memory Writes和Messages请求的TLP 头
2. PD缓存存放Memory Writes和Messages请求的TLP 数据；
3. NPH缓存存放Non-Posted请求的TLP 头；
4. NPD缓存存放Non-Posted请求的TLP 数据；
5. CPLH缓存存放Read/Wirte Completions请求的TLP 头；
6. CPLD缓存存放Read/Wirte Completions请求的TLP 数据；
>PCIe总线将Header和Data缓存分离的主要原因是，通常TLP的Header的大小是固定的但是Data的大小并不固定，Data的长度通常由TLP的Length字段确定。将Header和Data缓存分离有利于合理利用Data缓存。

PCIe对于每种不同类型的Header和Data使用不同的Credit值，PCIe总线规范并没有规定如何设置Credit，但是规定了初始化之后的最小值。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731184530.png)

# 数据链路层
数据链路层（Data Link Layer）主要进行链路管理（Link Management）、TLP错误检测，Flow Control和Link功耗管理。
数据链路层不仅可以转发来自事务层的包（TLP），还可以直接向另一个相邻设备的数据链路层直接发送DLLP，比如应用于Flow Control和Ack/Nak的DLLP。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220521.png)

数据链路层还实现了一种自动的错误校正功能，即Ack/Nak机制。如下图所示，发送方会对每一个TLP在Replay Buffer中做备份，直到其接收到来自接收方的Ack DLLP，确认该DLP已经成功的被接受，才会删除这个备份。如果接收方发现TLP存在错误，则会向发送发发送Nak DLLP，然后发送方会从Replay Buffer中取出数据，重新发送该TLP。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220653.png)

两种DLLP（转发TLP的DLLP，用于Flow Control或Ack/Nak等的DLLP）的结构图分别如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220714.png)
一个Non-Posted传输中，Ack/Nak的执行过程如下图所示：
![](../限速%20&%20流控/attachments/Pasted%20image%2020230731220808.png)

# PCIe性能
## 性能的定义
PCIe GEN1最大传输速率是2.5Gbps,PCIe GEN2最大传输速率是5Gbps，PCIe GEN3最大传输速率是8Gbps。这些速率只的是raw bit传输速率，并不是系统的有效data传输速率。有效的数据传输速率因为overhead要比此低。

各种类型的数据，例如内存、I/O 或配置数据，通过 PCI Express 系统传输。大多数设计都专注于尽可能有效地传输数据 。就本白皮书而言，_性能_定义为 系统的有效数据传输。 此后， data统称有效数据。

## 数据传输overhead
通过 PCI Express 系统的任何数据移动都包含一定数量的数据overhead开销。 PCIe系统影响包括 symbol encoding， 传输层数据包 （TLP） overhead， 和 traffic overhead。

### Symbol Encoding
PCIe GEN1 和 GEN2协议使用 8B/10B 编码方案传输；PCIe GEN3传输速率设置为 8.0 Gb/s。为了将第二代速率 5.0 Gb/s 的有效带宽加倍，决定使用具有 2% 损耗特性的 128B/130B 编码，而不是 8B/10B（20% 损耗特性）。

尽管第一代传输线可以以 2.5 Gb/s 的速度运行，但 8B/10B 编码将有效带宽降低至每个通道每个方向 2.0 Gb/s。  同样，第三代可以使用 128B/130B 编码以 8.0 Gb/s 的速度运行，这将理论带宽降低到 7.9。由于数据包和流量开销，实际系统带宽略低，总体系统效率为每通道每方向 7.8 Gb/s。

### Transaction Layer Packet Overhead
PCI Express 系统在 TLP 的有效负载中传输数据。内存数据在内存写入 TLP 和完成数据 TLP 中传输，这是对内存读取操作的响应。图 1 和图 3 分别显示了 Gen 2 和 Gen 3 的典型 TLP。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801145221.png)
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801145244.png)

事务层、数据链路层 (DLL) 和物理层 (PHY) 增加了每个TLP的开销，从而降低有效数据传输率。
>事务层  增加加数据包头和可选的端到端循环冗余校验（ECRC）。 DLL 将序列号和链路层 CRC (LCRC) 添加到  数据包以保证通过链路的成功传输。 PHY 增加了标记数据包开始和结束的信息。

大量数据的传输需要多个TLP，每个TLP都有自己的overhead。尽管每个 TLP 包含给定量的开销，但更大的倍数 TLP 传输提高了链路效率。**最大有效负载大小 (MPS) 设置，分配给通信设备，确定最大 TLP 大小。  增加MPS并不一定会导致链接相应增加效率，因为随着单个 TLP 变得更大，其他因素例如流量开销开始影响链路性能。**

### Traffic Overhead
PHY 和 DLL 都会引入流量开销。当链路达到正常运行状态 (L0) 时，PHY 会插入skip-ordered集以补偿两个通信端口之间的比特率差异。对于 Gen 1 和 Gen 2 速度，skip-ordered集的长度为四个字节。对于 Gen 3 速度，skip-ordered集的长度通常为 16 个字节。
DLL 的目的是维持两个链接伙伴之间的可靠数据传输。为了实现这一目标，PCI Express 规范定义了源自 DLL 层并由 DLL 层使用的数据链路层数据包 (DLLP)。各种类型的 DLLP 用于链接管理。确认 (ACK) DLLP、非确认 (NAK) DLLP 和流量控制 (FC) DLLP 对性能影响最大。
### Link Protocol Overhead
对于从一个设备发送到另一设备的每个 TLP，必须向 TLP 的发送方返回 ACK 或 NAK，以指示数据包的接收成功或不成功。发送设备必须将 TLP 保存在其重播缓冲区中，直到收到该 TLP 的 ACK。如果在原始传输过程中出现问题，则可以再次发送数据包，确保不会丢失数据，并实现高度可靠的链路。鉴于协议的 ACK/NAK 性质，当发送大量 TLP 时，还会生成大量 ACK/NACK DLLP，从而减少链路带宽。
该协议允许将等待传输的相同类型的多个 DLLP 折叠为单个 DLLP。在链路上折叠 ACK 既有优点也有缺点。折叠 ACK 或 NAK 可减少链路上的流量开销。然而，如果 ACK 发送得不够频繁，则发送器可能会在数据包等待从重播缓冲区中清除时限制数据包传输。确定何时或何时不将多个 DLLP 折叠为一个的机制是单个设备数据链路层设计的一部分。如果数据包确认速度不够快，传输设备的重放缓冲区就会填满，并且在确认旧数据包之前不允许发送新数据包。这会停止线路上的数据传输。

### Flow Control Protocol Overhead
基于信用的流量控制消除了由于接收缓冲区溢出而导致的数据包丢弃。虽然流量控制是必要的，并且比传统 PCI 中较旧的“重试”模型更有利，但它仍然具有降低链路效率的潜在后果。每个链路伙伴发送的 FC DLLP 不断更新设备接收器的状态，以便发送设备仅在知道接收器有足够的缓冲区空间时才发送数据包。发送器保存链路伙伴接收器上可用信用数量的运行计数，并在每次发送数据包时递减该计数。

接收器处理数据包并释放缓冲区空间后，它会发送 FC 更新 DLLP，通知发送设备可用空间。设备处理和传输 FC 更新 DLLP 的效率会影响链路的整体性能。与设备处理折叠 ACK/NAK 的方式类似，设备用于处理和传输流量控制更新的机制取决于设备的设计，并且在设备之间可能有所不同。同样，根据发送流量控制更新的频率，有利有弊。不太频繁的流量控制更新可减少链路管理流量开销。然而，接收器的缓冲区更大以维持合理的性能。

## System Parameters Affecting Performance
除了协议和流量开销之外，还有其他因素会影响 PCI Express 系统的性能。其中三个是最大有效负载大小、最大读取请求大小和请求类型。

### Maximum Payload Size
尽管 PCI Express 规范允许高达 4,096 字节的有效负载，但规范说：“软件必须注意确保每个数据包不超过  数据包路径上任何系统元素的 Max_Payload_Size 参数。”这意味着层次结构中的每个设备必须使用相同的 MPS 设置，并且设置不得超出层次结构中任何设备的能力。所以，具有高 MPS 功能的设备需要在较低 MPS 设置下运行，以容纳具有最低 MPS 能力的设备。例如，下图中的 PCI Express 系统被编程为 128 字节以容纳端点 3。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801150226.png)

系统的 MPS 设置是在枚举和配置过程中确定的过程。层次结构中的每个设备都会在其设备中通告其 MPS 能力寄存器（位于设备的配置空间中）。软件探测每个设备以确定其 MPS 能力、确定 MPS 设置，以及通过将 MPS 设置写入其设备控制寄存器来对每个设备进行编程。

MPS 设置确定传输给定数据量所需的 TLP 数量。随着 MPS 的增加，传输同一块数据所需的 TLP 数量会减少。公式 2 定义了数据包效率。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801150714.png)
表 3 显示了四种 MPS 设置的数据包效率计算。这些示例假设 TLP 具有 3 DW 或 12 字节标头并且没有 ECRC，从而产生总共 20 字节的开销。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801150751.png)

当MPS从128字节增加到256字节时，数据包效率增加6%。当MPS从512字节增加到1024字节时，数据包效率仅增加4%。对于当前可用的大多数系统，MPS 往往设置为 128 字节或 256 字节。无论给定 MPS 的传输大小如何，数据包效率都是固定的。例如，使用 128 字节 MPS 设置，任何大小的传输的数据包效率均为 86%。

### Maximum Read Request Size
在配置过程中，软件还将最大读取请求大小编程到每个设备的控制寄存器中。该参数设置内存读取请求的最大大小，最大可设置为 4096 字节，以 128 字节为增量。
最大读取请求大小可以大于 MPS。例如，可以向具有 128 字节 MPS 的设备发出 512 字节读取请求。返回读取请求数据的设备将 Completion with Data TLP 的大小限制为 128 字节或更少，这需要一次读取多次完成。  ***系统使用最大读取请求大小来平衡整个拓扑中的带宽分配。限制设备在一次传输中可以读取的最大数据量可以防止其独占系统带宽。***

最大读请求大小也会影响性能，因为它决定了需要多少个读请求才能获取数据，读请求为100%开销（overhead），因为它们不包含任何有效负载。使用最大读取请求大小 128 字节读取 64 KB 数据需要 512 个内存读取 TLP（64 KB / 128 字节 = 512）来从内存请求数据。为了提高移动大数据块时的效率，读取请求的大小应尽可能接近最大读取请求大小，以减少必须传输的读取次数。

### Read Completion Boundary
数据完成 TLP 返回数据以响应读取请求。读取完成边界 (RCB) 允许来自单个读取请求的数据由多个完成提供服务。数据分为 64 字节或 128 字节补全，自然地与地址边界对齐。这意味着数据量某些完成返回的值可能小于 RCB 设置，具体取决于下一个地址边界的位置。通常，大多数根复合体将 RCB 设置为 64 字节，并以 64 字节完成形式返回数据，而不是 MPS 允许的数据。
图 9 显示了端点从地址 0x00010028 读取的 256 (0x100) 字节如何由 RCB 设置为 64 字节的根联合体返回。
![](../限速%20&%20流控/attachments/Pasted%20image%2020230801151401.png)
许多根联合体使用 64 字节 RCB，即使 MPS 设置为 128 字节或更高，这意味着读取完成不仅会受到传输请求所花费的时间的影响，而且还会受到完成被划分为RCB 设置允许的最小封装。
# 参考
```c
PCIe的架构图：
http://r12f.com/posts/pcie-1-basics/
http://r12f.com/posts/pcie-2-config/
http://r12f.com/posts/pcie-3-tl-dll/

# flow control:
http://blog.chinaaet.com/justlxy/p/5100053464
http://blog.chinaaet.com/justlxy/p/5100053465
https://zhuanlan.zhihu.com/p/352533416

```