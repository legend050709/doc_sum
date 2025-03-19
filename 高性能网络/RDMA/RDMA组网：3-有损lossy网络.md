```table-of-contents
```
# 背景
RDMA 在高性能计算（HPC）社区中长期以来一直被用于使用基于credit的流量控制的专用 Infiniband 集群，以确保网络无丢包(lossless)。由于在这类集群中数据包丢失很少，因此 RDMA Infiniband 传输（在 NIC 上实现）并未设计为高效地从数据包丢失中恢复。

当接收方收到乱序数据包时，它会简单地丢弃该数据包并向发送方发送负确认（NACK）。当发送方看到 NACK 时，它会重新传输所有在{BANNED}最佳后一个被确认数据包之后发送的包（即，go-back-N）。

为了利用以太网在数据中心的广泛使用，RoCE 被引入以实现 RDMA 在以太网上的应用。RoCE 采用了相同的 Infiniband 传输设计（包括回退 N 丢包恢复），并通过 PFC 实现了网络无丢包。

## PFC的问题
优先级流量控制（PFC） 是以太网的流量控制机制，其中交换机会在队列超过某个配置阈值时向上游实体（一个交换机或 NIC）发送暂停（或 X-OFF）帧。当队列降低到该阈值以下时，会发送 X-ON 帧以恢复传输。当配置正确时，PFC 使网络无丢包（只要所有网络元素保持正常工作）。
然而，这种对拥堵的粗略反应对导致拥堵的流量是无差别的，这导致了近年来在多篇论文中记录的各种性能问题 ，例如队头阻塞、拥塞传播和偶发的死锁等。


# 参考
```bash
# 大规模 RDMA AI 组网技术创新：算法和可编程硬件的深度融合 【系列文章+++】
https://blog.csdn.net/Jmilk/article/details/145792856

# Lossy网络的RoCE方案——IRN
http://m.blog.chinaunix.net/uid-28541347-id-5887217.html

# RDMA有损网络（Lossy）和选择性重传
https://zhuanlan.zhihu.com/p/699271616


```