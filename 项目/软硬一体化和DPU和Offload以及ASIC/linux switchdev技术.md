```table-of-contents
```
# 背景
## Linux上的路由与交换
Linux诞生于网络，天生对网络拥有全面且强大的支持，即便再复杂的协议，再封闭的技术，几乎都可以找到对应的Linux实现。然而这并不是说Linux网络就天下无敌了，它存在很多不合理的地方。

Linux拥有对**路由**的强大支持，在数据平面，你可以很轻松地实现一种路由查找算法，在控制平面，你也可以在用户态实现任何已有的或者你自己设计的路由协议，然而，这一切都是软的，也就是说，都是CPU来完成的。

 当我们知道**路由和交换**的区别之后，就会发现，Linux一直以来都没有实现真正的交换，起码在通用接口层面上没有一个合理的解决方案。Linux的bridge模块？算了吧，它只是实现了一个软网桥，和真正交换机不沾边的。


## 传统的交换机功能开发

通常情况下，交换芯片厂商会提供用户态软件开发工具包（SDK）来实现与硬件的接口。要用交换芯片设计一个交换机或路由器产品，设备商需要开发一个网络操作系统（NOS）或者移植像SONiC一样的开源产品。
虽然这种模式支撑传统业务很多年，但它并不能够有效地满足不断变化的市场需求。为了满足网络扩展和应用需求的增长，硬件设备商需要不断开发新技术和新协议，这就导致变更的成本高昂，大部分情况下客户都会抱怨响应速度太慢。


# 交换offload在主机侧的实现
## Switchdev 的介绍
 Switchdev的出现就帮助厂商解决了这一难题，因为它利用大家熟知的Linux开源框架，用户**利用Linux环境和工具**就可以打破厂商的禁锢，这种灵活性和自由度可以更好地满足客户的需求。
 
 Swtichdev的内核模式不是全新造轮子，而是对现有的Linux网络工具和应用进行改造。这种方法对服务器端的SmartNIC和路由器/交换机同样有效，因此**终端用户能够消除对厂商特定API的依赖**。

如下图所示，Switchdev位于Linux内核层，它可以**将内核的数据转发平面卸载到交换机的ASIC芯片上**。

![](attachments/Pasted%20image%2020240709120701.png)

通过这种方式，就可以**用标准开放的Linux接口取代专有的SDK和NOS接口**。
Switchdev还规划了一个统一的接口，简化了集成、配置和支持的过程。与SONiC等NOS相比，Switchdev驱动的网络系统更加轻量级。

## Linux 4.0+内核对硬件交换模块的支持(HW Switch Offload)

Linux 4.0引入了一个switchdev框架，它代表一类拥有“交换”能力芯片的多网口设备的抽象。其中每一个网口就是一个port，在switchdev框架中被注册成一个net_device。除此之外，内核中自带了一个rocker driver，演示了一个实际的设备驱动的实现。

switchdev起源于Open vSwitch项目，由Jiři Pirko在2014年9月首次提出。在2015年2月的Netdev 0.1会议上，网络开发人员决定扩展并采用switchdev作为硬件交换机芯片的通用解决方案。
**switchdev驱动模型出现之前，Linux需要交换机厂商的专门工具套件操作交换机，而在switchdev驱动模型之后，通用接口被实现，交换机正式纳入Linux网络设备体系，Linux可以用标准接口实现交换机的控制面和管理面**。




## 架构

![](attachments/Pasted%20image%2020240709120004.png)

注意，理想化的实现中，OpenFlow控制器可以直接将流表注入到设备中，从而指导设备直接进行数据包交换。流表的内容超级复杂，不是本文的目标，但是相信在后一个内核版本中会出现相关的Document。

利用Switchdev，除了常见的Linux内核数据面能够卸载到硬件，也可以直接将流表注入到设备中，从而指导设备直接进行数据包交换，如mellanox的一些智能网卡的做法。
采用了硬件交换模块的Linux BOX和原来的截然不同了，它更像是一个高端的专业网络设备，类似Cisco那样的。它看起来就是下面的样子：

![](attachments/Pasted%20image%2020240709120117.png)


在switchdev驱动框架下，硬件交换机设备上的每个物理端口都在内核中注册为一个net_device，就像对现有的网络接口卡(nic)所做的那样。可以使用现有的工具(如桥接、ip和iproute2)将端口绑定或桥接、隧道化或划分vlan。switchdev驱动程序的优点是这样的交换结构可以被卸载到交换机硬件上。
因此，驱动程序将转发数据库(FDB)中的每个条目镜像到硬件，并监视其更改情况。

 内核中switch如下：
![](attachments/Pasted%20image%2020240709141418.png)


参考：[Ethernet switch device driver model (switchdev)](https://www.kernel.org/doc/Documentation/networking/switchdev.txt)

# DENT操作系统

  Switchdev项目由Linux内核（具体是**netdev**）托管，这是Linux基金会旗下的一个开源项目社区。
  随着Linux内核的变化，无论是增加新的功能还是关键问题的修复，通过社区就可以确保Switchdev基础架构持续向前演进。
   Linux有众多的应用程序来实现各种网络协议，主要的愿景就是**让这些协议能够利用到交换机的硬件能力，或者说利用到交换芯片来卸载流量处理**，这样就不会浪费通用CPU资源。为了将基础级的Switchdev转化为白盒设备所需的全面商用级NOS，Linux基金会正在围绕Switchdev建立一个**应用生态系统DENT**，创始成员包括亚马逊和一些网络硬件厂商。
  
![](attachments/Pasted%20image%2020240709120938.png)

DENT是基于Ubuntu的Linux发行版，它封装了交换机硬件（风扇、温度传感器、ASIC等）的驱动程序以及开源的FRRouting路由协议套件，其中包括BGP、IS-IS、LDP、OSPF、PIM和RIP的协议守护进程。FRRouting软件使用Linux netlink API对Linux内核的数据包转发进行编程，在硬件交换机平台上，数据包转发由switchdev驱动卸载到ASIC进行线速转发。 

DENT采用Linux作为网络操作系统的一个主要优势是，**用于配置、管理和监控Linux服务器的工具也可以用来管理网络交换机**。此外，DENT虚拟机的运行方式与在物理交换机上运行的DENT完全相同，这样就可以保证配置在进入生产网络之前在虚拟环境中得到充分验证。


# 其他
## Switchdev成功的关键

目前高端网络设备市场已经开始普及白盒产品，中低端商用交换机和路由器等细分市场如果想要打开白盒交换机的想象空间，就必须有一套开放、灵活和标准的解决方案，Switchdev的目标就是为客户提供一个全新的、简化的网络设备开发环境。

通过采用Switchdev作为开源基础架构框架，就可以利用到开放的Linux工具，设备厂商**省去开发专有NOS或网管系统的商务成本，同时减少整合商业交换芯片所需的繁琐开发流程**。**统一的Linux界面**也降低了客户的使用成本，简化了学习曲线。
比如，智能网卡采用Switchdev可以方便地与数据中心或网络编排系统（OpenStack、Kubernetes、XCloud等）集成。

开发Switchdev需要包括Linux内核、驱动和应用方面的相关知识，软件工程师要有底层网络和嵌入式软件开发能力，包括对所选交换芯片SDK的了解和平台集成经验。
芯片和设备厂商在为其产品开发Switchdev驱动时，在遵循开源社区的开发原则同时，建议将相应产品的Switchdev驱动向Linux内核提交。因为Switchdev尚在发展中，设备厂商在为产品适配Switchdev时，相关平台代码以及移植问题上传到开源社区，就可以确保在未来的Linux发行版中得到支持。

## rocker dirver
内核中自带了一个rocker driver，演示了一个实际的设备驱动的实现，最先支持 switchdev 的就是是 QEMU 的 Rocker 软件交换机。后来 Mellanox 和 Broadcom 等公司均提供了支持 switchdev 的交换机器。它就是一个pci dirver，对接了switchdev框架将kernel数据面下发的模拟的硬件。

```bash
static struct pci_driver rocker_pci_driver = {

    .name = rocker_driver_name,

    .id_table = rocker_pci_id_table,

    .probe = rocker_probe,

    .remove = rocker_remove,

};
```


 Rocker 是一个模拟网络交换机平台，旨在加速内核网络交换机驱动程序模型的开发。 Rocker 有两个部分：一个带有 PCI 主机接口的 62 端口交换机芯片的 Qemu 仿真和一个 Linux 设备驱动程序。 目标是模拟数据中心/企业中使用的当代网络交换机 ASIC 的功能，以便社区可以在内核中开发交换机设备驱动程序接口。 最初的目标功能是 L2 桥接功能卸载和 L3 路由功能卸载。 在这两种情况下，转发（数据）平面都被卸载到交换机设备，但控制和管理平面仍保留在 Linux 中。 L2overL3 隧道、L2 绑定、ACL 支持和基于流的网络等其他功能正在计划中或正在进行中。


 根据官方的说法，Rocker 背后的动机是加速开发用于网络交换机的 Linux 内核设备驱动程序模型，在没有供应商提供的开源驱动程序的情况下，Rocker 被创建为网络交换机设备的仿真，其功能集接近于现实世界的供应商交换机 ASIC。使用 Rocker 设备，我们可以创建设备驱动程序来开发和测试 switchdev 驱动程序模型，而无需依赖供应商的 SDK。期望一旦 switchdev 达到一定的成熟度，供应商或社区提供的用于现实世界 ASIC 的设备驱动程序将会出现，并且对 Rocker 的需求将随着时间的推移而减少。也就是说是一个演示性质的实现，为大家开发支持switchdev的设备驱动提供参考。

# 参考  
```bash
# Linux 4.0+内核对硬件交换模块的支持(HW Switch Offload)
https://blog.csdn.net/dog250/article/details/45788449

第四章云网络4.9.6节——linux switchdev技术
https://cloud.tencent.com/developer/article/2108936
```