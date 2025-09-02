```table-of-contents
```
# NVMe
## 介绍

# NVMe-oF
##  介绍
**Non-Volatile Memory Express (NVMe)**：
- **非易失性存储器高速接口协议**，专为高性能 SSD 设计的本地通信协议（通过 PCIe 总线）。

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