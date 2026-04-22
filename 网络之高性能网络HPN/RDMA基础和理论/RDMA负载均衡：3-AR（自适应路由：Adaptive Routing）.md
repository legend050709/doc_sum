```table-of-contents
```
# Spectrum-X 关键技术
提起**Adaptive Routing(AR：自适应路由)**，就不得不提NVIDIA Spectrum-X平台（类似技术Broadcom 也支持，只不过叫Dynamic Load Balancing，简称DLB）。

为支持和加速AI工作负载，Spectrum-X 从端侧网卡（DPU/SuperNIC）到交换机、电缆/光学器件、网络和加速软件，进行了一系列优化，包括：

- **交换机侧AR**: Spectrum-4上的NVIDIA RoCE自适应路由AR（Adaptive Routing）
- **端侧DDP**: BlueField-3上的NVIDIA直接数据放置（Direct Data Placement, DDP）
- Spectrum-4和BlueField-3上的NVIDIA RoCE拥塞控制
- NVIDIA AI加速软件
- 端到端AI网络可见性

# AR

## 背景
大模型训练中的 all-reduce、all-to-all、parameter synchronization 和 expert dispatch 会形成大规模、持续性、同步化的 elephant flows；
基于 **传统ECMP 的静态散列**无法稳定地把这些流均匀分布到所有可用链路上，因此容易形成**局部拥塞、路径失衡和显著尾时延**。

NVIDIA 为解决这一问题，在 Spectrum-X 中引入RoCE Adaptive Routing，使交换机能够依据**实时遥测**结果，以比传统 ECMP **更细粒度**的方式把流量动态分散到不同路径。

但只要**路径选择从固定散列走向动态、多路径、细粒度分流**，就不可避免地引入一个伴随问题：同一逻辑消息的不同数据单元可能**乱序到达**接收端。

对于 AI 集群而言，乱序本身不是问题的本质；真正的问题在于，若接收端不能在线速条件下处理乱序，网络侧通过自适应路由获得的拥塞收益，会被接收端的重组、缓存和软件参与开销抵消。

## 介绍

NVIDIA Spectrum-X 的Adaptive Routing是一种精细化负载均衡技术，通过动态调整RDMA数据路由以避免拥塞，结合`BlueField 3（DPU/SuperNIC）`的DDP技术，提供最佳负载均衡和实现更高效的数据带宽。

Spectrum-X 网络平台是第一个专为提高Ethernet-based AI云的性能和效率而设计的以太网平台。这项突破性技术在类似LLM的大规模AI工作负载中，提升了1.7倍AI性能、能效，以及保证在多租户环境中的一致、可预测性。

Spectrum-X**基于Spectrum-4以太网交换机与NVIDIA BlueField-3 （DPU/SuperNIC）网卡**构建，针对AI工作负载进行了端到端优化。
> 即：Adaptive Routing的实现是需要NVDIA的Spectrum交换机（如Spectrum-4系列交换机）和其（DPU/SuperNIC）网卡（Bluefield-3 DPU网卡）配合完成的。

## AR和ECMP
### 对比
（1）静态和动态
ECMP是静态的，不管网络链路的拥塞情况
AR是动态的，实时感知网络链路的拥塞情况，选择

（2）流粒度和包粒度：
ECMP是基于流粒度进行分发；
AR是基于包粒度进行分发。

# DDP
在 NVIDIA Spectrum-X 架构中，DDP（Direct Data Placement）是支撑 RoCE Adaptive Routing 的关键**接收侧机制**。它的作用是在多路径、自适应路由、乱序到达的条件下，保证接收端仍能以线速将数据直接放置到目标内存位置，从而使交换网络能够采用更积极的负载分散策略。

Direct Data Placement 的核心语义是：发送端为传输数据附加足够的放置信息，使接收端网络接口能够在数据到达时，不经过额外中间缓冲或软件拷贝，直接将其写入预先指定的目标内存区域。主要应用在**流量负载均衡和乱序接收问题**。

## DDP & DMA & RDMA
DMA 描述的是设备直接访问内存的搬运机制，RDMA 描述的是远程内存访问语义，而 **DDP 关注的是接收数据如何在到达瞬间被直接放置到正确位置**。

在 Spectrum-X 中，DDP 的重点进一步从“零拷贝”扩展到**乱序接收**和 **与交换网络路径分散协同**。

传统 RDMA 已经减少了 CPU 参与和协议栈开销，但其收益主要建立在稳定的远程内存访问语义之上；Spectrum-X 所面对的问题是，在路径自适应、链路级实时负载分担和可能乱序到达的前提下，如何维持这些收益。

## DDP 和 AR
在 Spectrum-X 中，AR（Adaptive Routing）与DDP不是两个彼此独立的优化点，而是一个统一系统中的**发送侧、网络侧与接收侧协同机制**。

![](attachments/Pasted%20image%2020260418170743.png)

**AR 负责动态选路，DDP 负责在乱序条件下完成正确的数据落点。**

如果只有 AR，没有DDP，接收端就必须承担更高的乱序重组代价，网络侧通过动态分流获得的收益会被接收路径开销抵消；
如果没有 AR，则多路径链路无法被充分利用，拥塞热点难以缓解。


传统以太网的一个长期弱点是：其路径负载分担与接收端处理模型并非为**大规模同步大象流**设计，导致**集体通信**下的**尾时延和抖动**较难控制。
通过 AR + DDP 的组合，Spectrum-X 试图把“路径动态化”与“接收放置确定性”同时建立起来，从而提高网络在 AI 训练中的可预测性。


# Spectrum-X平台对于TCP和RDMA流量的不同处理
具体来说就是Spectrum交换机可以针对RDMA和TCP采用不同的策略，如**TCP使用flowlet，RDMA（RoCE）采用逐包的load balance**。
Spectrum可以**通过网络侧交换机和端侧（DPU/SuperNIC）的紧耦合联动**，做到**实时动态监控ECMP各个链路的物理带宽和端口出口拥塞情况，来做到基于每个报文的动态负载分担**。

Spectrum-4交换机负责选择每个数据包基于最低拥塞端口，均匀分配数据传输。当同一流的不同数据包通过网络的不同路径传输时，它们可能以无序的方式到达目的地。

==BlueField-3 （DPU/SuperNIC）通过DDP处理无序数据，以提供连续的数据透明给应用程序==。

# 原理

![](attachments/Pasted%20image%2020260324233101.png)

端侧设备需要给**报文打标**，来告诉交换机这些流量可以进行`Adaptive Routing`，否则导致了乱序问题目的端可能无法处理，另外可能NVIDIA为了实现`Adaptive Routing`需要在报文中添加一些标记，这些也需要在目的端网卡去除。

要实现`per packet`的打散，必然要面临乱序的问题；而**对RDMA来说，在网卡上实现报文的重排序代价是比较大的，因为缓存报文需要大量的buffer**。为了解决这个问题，BF3支持了DDP（Direct Data Placement），即根据报文中的目的内存地址直接将报文放入目的host的内存中，而不需要在网卡上buffer。

如下图所示，报文按照1,2,3,4的顺序发送，如果报文4先到达可直接根据报文的地址放入host内存。

![](attachments/Pasted%20image%2020260324233207.png)

# 优缺点
## 优点

通过**adaptive routing（自适应路由）和DDP**两种硬件加速技术的结合，解决了**传统以太网ECMP带宽不均**和报文乱序问题，也消除了一些应用由于处理乱序造成的长**尾延迟**问题。

## 缺点
首先是必须绑定NVIDIA的一套硬件，包括交换机和网卡，这是NVIDIA的一贯风格，尽可能的绑定。

# 参考
```bash
# NIC DDP 与 Spectrum-X AR的关系
https://mp.weixin.qq.com/s/KipkpQ7Qx5GX8wX4loC_JQ
```