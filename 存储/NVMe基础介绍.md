```table-of-contents
```
# NVMe
## 介绍
**Non-Volatile Memory Express (NVMe) 非易失性存储器高速接口协议**： 专为高性能 SSD 设计的本地通信协议（通过 PCIe 总线）。

NVMe协议是为实现对FLASH介质的高性能访问而推出的标准协议，从最初用于访问PCIe FLASH SSD，逐渐演进到基于其他网络的远程访问。

# NVMe SSD
**SSD** = 固态硬盘（Solid State Drive），**NVMe** = Non-Volatile Memory Express（一种协议）
**NVMe SSD** = 使用 NVMe 协议的 SSD，它不是一种“新型存储介质”，而是一种**访问协议 + 总线架构方式**。

## SATA SSD 和 NVMe SSD

 **SATA SSD特点**：
```bash
- 单队列
- 中断驱动
- 内核参与
- 调度路径长
```

 **NVMe SD特点**：
```bash
- 多队列 per core
- 用户态 polling
- doorbell 通知
- 减少锁
- 减少中断
```

```bash
SATA (AHCI):
- 更偏 Reactor
- 中断驱动
- 内核调度


NVMe:
- 更偏 Proactor
- 提前提交大量 IO
- 硬件批量执行
- 以队列为中心
```


### 为什么 NVMe 会比 SATA 快很多
```bash
### 1️⃣ 队列并行度大幅提升
SATA：1 queue × 32 depth  
NVMe：64K × 64K

### 2️⃣ 锁消失
每核一个 queue，不需要全局锁

### 3️⃣ 减少中断
支持 polling

### 4️⃣ PCIe 直连
带宽从：
- SATA3 ≈ 600MB/s
- PCIe 4.0 x4 ≈ 8GB/s
```


## NVMe 和 SSD 的关系
```bash
SSD = 存储介质（NAND Flash）
NVMe = 访问这个存储介质的协议

SSD 是存储介质  
NVMe 是访问存储的“高性能多队列协议”  
SATA SSD 是老架构  
NVMe SSD 是为多核 + 并行 + 低延迟时代设计的架构
```

|类比|说明|
|---|---|
|SSD|像网卡|
|NVMe|像 RDMA 协议|
|SATA|像 TCP over kernel|



# NVMe-oF-Fabrics
在当前大数据节点的背景下，存储节点不是独立存在的，海量的存储数据读写，需要高性能存储网络的支撑，由此产生了NVMe-oF。

NVMe over Fabrics（NVMe-oF）是一种网络存储协议，它允许NVMe（Non-Volatile Memory Express）存储协议通过网络传输，而不是仅限于本地PCI Express（PCIe）总线。

NVMe-oF的目标是将NVMe的高性能和低延迟特性扩展到网络存储环境中，使得远程主机能够像访问本地NVMe存储设备一样访问远程存储资源。

##  介绍
**Non-Volatile Memory Express (NVMe)**：非易失性存储器高速接口协议
专为高性能 SSD 设计的本地通信协议（通过 PCIe 总线）。

**over Fabrics (oF)**：
- **基于网络架构扩展**。"Fabrics" 指高速网络结构（如以太网、InfiniBand、光纤通道等），允许 NVMe 命令和数据通过**网络传输**，而非仅限于本地 PCIe 总线。

**核心作用与意义：**
- **将 NVMe 的高性能扩展到网络**： 使远程存储设备（如全闪存阵列）能像本地 NVMe SSD 一样被访问，**保持超低延迟、高吞吐量和高并发特性**（传统网络存储协议如 iSCSI 或 NFS 无法做到）。
- **解耦计算与存储**： 服务器（主机）可通过网络直接访问远程 NVMe 存储资源，实现**高性能分布式存储架构**。

**NVMe-oF = NVMe（高性能本地协议） + over Fabrics（网络扩展）** 它彻底改变了数据中心存储架构，使**网络存储性能逼近本地 NVMe SSD**，成为现代高性能存储网络的基石技术。


## NVMe-oF 传输
NVMe over Fabrics 要求底层 NVMe 传输提供可靠的 NVMe 命令和数据传递。NVMe 传输是一个抽象的协议层，独立于任何物理互连属性。

![](attachments/Pasted%20image%2020260621134557.png)

上图展示了 NVMe 传输的分类及其示例。

NVMe 传输可能展现出一个内存模型、一个消息模型，或两者的结合。
（1）内存模型是指通过执行显式的内存读写操作在网络节点之间传输命令、响应和数据。
（2）消息模型则是指仅通过发送包含command capsules、response capsules和数据的消息在网络节点之间进行传输。
（3）消息/内存模型则结合了消息和显式的内存读写操作，以在网络节点之间传输command capsules、response capsules和数据。数据可以选择性地包含在command capsules和response capsules和响应中。

## NVMe-oF的传输协议

在NVMe over Fabrics（NVMe-oF）中，不同的传输协议（见下图）在数据面中扮演着关键角色，它们各自有不同的工作原理和特点。以下是针对这几种传输协议的详细讲解：

![](attachments/Pasted%20image%2020260621134754.png)

### TCP （NVMe-oF-TCP： NVMe-oT）

TCP是互联网协议族中的一个核心协议，提供可靠的、面向连接的数据传输服务。在NVMe-oF中，NVMe over TCP（NVMe-oT）使用TCP作为传输层协议，通过标准的以太网传输NVMe命令和数据。

NVMe-oF-TCP的优势在于其广泛的兼容性和易于部署，因为它可以在现有的TCP/IP网络基础设施上运行，无需特殊的硬件支持。
然而，TCP的性能通常不如RDMA等专用协议，因为它在传输过程中需要更多的CPU参与，增加了延迟和处理开销。

### RDMA（NVMe-oF-RDMA）

RDMA是一种网络协议，允许计算机在网络中直接访问另一台计算机的内存，而无需远程计算机的操作系统参与。这减少了CPU的负担，降低了延迟，并提高了数据传输效率。

在NVMe-oF中，RDMA可以通过多种网络技术实现，如InfiniBand、RDMA over Converged Ethernet（RoCE）和Internet Wide Area RDMA Protocol（iWARP）。

RDMA的优势在于其极低的延迟和高吞吐量，适合高性能计算和数据中心环境。使用RDMA的NVMe-oF需要支持RDMA的网络硬件和适配器，这可能需要额外的投资和配置。

### FC（Fiber Channel，光纤通道）


## 技术实现关键
- **协议映射**： NVMe 命令集（用于本地 PCIe 通信）被封装到网络传输协议中，例如：
    - **NVMe over RDMA**（如 RoCE v2、InfiniBand）：利用 RDMA 技术绕过 CPU 和内核协议栈，实现零拷贝、低延迟。
    - **NVMe over TCP**：基于标准以太网（TCP/IP），兼容性强，但延迟略高于 RDMA。
    - **NVMe over Fibre Channel**（FC-NVMe）：适用于传统 FC 存储网络。

- **端到端架构**：
    - **Host（Initiator）**：发起 NVMe 命令的客户端。
    - **Target**：提供 NVMe 存储服务的远程设备（如存储阵列）。





# NVMe和SPDK

# 参考
```bash
# 利用SPDK改善NVMe存储I/O性能
https://zhuanlan.zhihu.com/p/702765269


https://www.cnblogs.com/vlhn/p/7727141.html
```