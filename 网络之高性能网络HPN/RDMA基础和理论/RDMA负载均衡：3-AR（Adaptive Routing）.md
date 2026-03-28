```table-of-contents
```
# 介绍
提起Adaptive Routing(自适应路由)，就不得不提NVIDIA Spectrum-X平台（类似技术Broadcom 也支持，只不过叫Dynamic Load Balancing，简称DLB）。
NVIDIA Spectrum-X 的Adaptive Routing是一种精细化负载均衡技术，通过动态调整RDMA数据路由以避免拥塞，结合BlueField 3的DDP技术，提供最佳负载均衡和实现更高效的数据带宽。

Spectrum-X 网络平台是第一个专为提高Ethernet-based AI云的性能和效率而设计的以太网平台。这项突破性技术在类似LLM的大规模AI工作负载中，提升了1.7倍AI性能、能效，以及保证在多租户环境中的一致、可预测性。

Spectrum-X**基于Spectrum-4以太网交换机与NVIDIA BlueField-3 DPU网卡**构建，针对AI工作负载进行了端到端优化。
> 即：Adaptive Routing的实现是需要NVDIA的Spectrum交换机（如Spectrum-4系列交换机）和其网卡（Bluefield-3 DPU网卡）配合完成的。

# Spectrum-X 关键技术

为支持和加速AI工作负载，Spectrum-X 从DPUs到交换机、电缆/光学器件、网络和加速软件，进行了一系列优化，包括：

- Spectrum-4上的NVIDIA RoCE自适应路由AR（Adaptive Routing）
- BlueField-3上的NVIDIA直接数据放置（Direct Data Placement, DDP）
- Spectrum-4和BlueField-3上的NVIDIA RoCE拥塞控制
- NVIDIA AI加速软件
- 端到端AI网络可见性

# Spectrum-X平台对于TCP和RDMA流量的不同处理
具体来说就是Spectrum交换机可以针对RDMA和TCP采用不同的策略，如TCP使用flowlet，RDMA（RoCE）采用逐包的load balance。Spectrum可以通过网络侧交换机和端侧DPU的紧耦合联动，做到实时动态监控ECMP各个链路的物理带宽和端口出口拥塞情况，来做到基于每个报文的动态负载分担。

Spectrum-4交换机负责选择每个数据包基于最低拥塞端口，均匀分配数据传输。当同一流的不同数据包通过网络的不同路径传输时，它们可能以无序的方式到达目的地。

BlueField-3 DPU通过DDP处理无序数据，以提供连续的数据透明给应用程序。


# 原理

![](attachments/Pasted%20image%2020260324233101.png)

端侧设备需要给报文打标，来告诉交换机这些流量可以进行Adaptive Routing，否则导致了乱序问题目的端可能无法处理，另外可能NVIDIA为了实现Adaptive Routing需要在报文中添加一些标记，这些也需要在目的端网卡去除。

要实现per packet的打散，必然要面临乱序的问题，而对RDMA来说在网卡上实现报文的重排序代价是比较大的，因为缓存报文需要大量的buffer。为了解决这个问题，BF3支持了DDP（Direct Data Placement），即根据报文中的目的内存地址直接将报文放入目的host的内存中，而不需要在网卡上buffer。如下图所示，报文按照1,2,3,4的顺序发送，如果报文4先到达可直接根据报文的地址放入host内存。

![](attachments/Pasted%20image%2020260324233207.png)

# 优缺点
## 优点

通过**adaptive routing和DDP**两种硬件加速技术的结合，解决了**传统以太网ECMP带宽不均**和报文乱序问题，也消除了一些应用由于处理乱序造成的长**尾延迟**问题。

## 缺点
首先是必须绑定NVIDIA的一套硬件，包括交换机和网卡，这是NVIDIA的一贯风格，尽可能的绑定。

# 参考
```bash

```