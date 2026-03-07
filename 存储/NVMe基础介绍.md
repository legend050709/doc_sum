```table-of-contents
```
# NVMe
## 介绍
**Non-Volatile Memory Express (NVMe) 非易失性存储器高速接口协议**： 专为高性能 SSD 设计的本地通信协议（通过 PCIe 总线）。

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


为什么 NVMe 会比 SATA 快很多：
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



# NVMe-oF
##  介绍
**Non-Volatile Memory Express (NVMe)**：非易失性存储器高速接口协议
专为高性能 SSD 设计的本地通信协议（通过 PCIe 总线）。

**over Fabrics (oF)**：
- **基于网络架构扩展**。"Fabrics" 指高速网络结构（如以太网、InfiniBand、光纤通道等），允许 NVMe 命令和数据通过**网络传输**，而非仅限于本地 PCIe 总线。

**核心作用与意义：**
- **将 NVMe 的高性能扩展到网络**： 使远程存储设备（如全闪存阵列）能像本地 NVMe SSD 一样被访问，**保持超低延迟、高吞吐量和高并发特性**（传统网络存储协议如 iSCSI 或 NFS 无法做到）。
- **解耦计算与存储**： 服务器（主机）可通过网络直接访问远程 NVMe 存储资源，实现**高性能分布式存储架构**。

**NVMe-oF = NVMe（高性能本地协议） + over Fabrics（网络扩展）** 它彻底改变了数据中心存储架构，使**网络存储性能逼近本地 NVMe SSD**，成为现代高性能存储网络的基石技术。

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