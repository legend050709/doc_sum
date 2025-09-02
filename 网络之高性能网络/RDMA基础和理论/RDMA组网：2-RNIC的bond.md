```table-of-contents
```
# 前置知识

## 区分以太网设备(ethernet device)和IB设备(ib device)
```bash
# ibdev2netdev
mlx5_bond_0 port 1 ==> bond0 (Up)

如上所示：
	mlx5_bond_0 是 ib device；
	bond0 是 以太网设备；
```

## RDMA相关知识
![](attachments/Pasted%20image%2020250710193532.png)

# RDMA RoCE Bonding

参考：[rdma bond](https://www.openfabrics.org/images/eventpresos/workshops2014/DevWorkshop/presos/Tuesday/pdf/13_rdma_bonding.pdf)

参考：[rdma and user space ethernet bonding](https://www.openfabrics.org/images/eventpresos/2016presentations/303RDMAUserSpc.pdf)

## 传统ethernet 以太网 的bond
我们知道操作系统里面，可以将2个实际的物理网卡，合体形成一个“逻辑网卡”，从而达到如主备/提升带宽等目的。

![](attachments/Pasted%20image%2020250314195703.png)


## Roce 的bond

### 限制
但是RoCE网卡，是否也跟普通网卡一样，支持Bond能力呢？
答案是的，RoCE也可以组Bond，只是比普通网卡多了一些约束。

![](attachments/Pasted%20image%2020250710170442.png)


参考：[# How to Configure RoCE over LAG (ConnectX-4/ConnectX-5-/ConnectX-6)](https://enterprise-support.nvidia.com/s/article/How-to-Configure-RoCE-over-LAG-ConnectX-4-ConnectX-5-ConnectX-6)

（1）`RoCE LAG` 是一种用于模拟 IB 设备的以太网绑定的功能，**仅适用于双端口卡(即：单卡双口)**。
（2）仅仅支持三种bond mode（mode1, mode2, mode4）;



### RoceV2 的bond 的实现

#### 先看传统以太网在链路层进行bonding

![](attachments/Pasted%20image%2020250710142327.png)

如上所示，是传统的 ethernet 的 bonding的实现, 特点如下：
```bash
（1）bonding的实现 在内核协议栈的三四层和物理口之间。
（2）物理接口无状态的转发数据包。
（3）对于上层来说，bonding是透明的，即应用层其实是不感知的。
（4）传输层实在内核协议栈实现的。
```

对于RDMA而言，进行bonding就存在一些挑战：
对于RDMA流量（比如：RoceV2）而言，物理网卡是有状态的(网卡中需要资源，保存状态。比如QP信息)，因为`RDMA`是`bypass kernel`的, `RoceV2`流量在`RNIC`网卡上进行`UDP/IP/Ethernet`层的封装和解封装，上层驱动和应用不感知`L2/L3/L4`层的信息。

![](attachments/Pasted%20image%2020250710194626.png)


#### 传输层的bonding
![](attachments/Pasted%20image%2020250710213604.png)

HAL: `hardware abstract layer`, 硬件抽象层。
bonding driver: 硬件独立的`bonding`驱动；

![](attachments/Pasted%20image%2020250710220621.png)
![](attachments/Pasted%20image%2020250710220111.png)




### 问题
#### 单卡双口的bond和多卡的bond
猜测：
"单卡双口的两个物理口进行bonding，如果某个成员口down掉，由于两个口属于同一个卡，此时流量到达另外一个口，应该也是可以查找到down掉的口的QP信息的。
但是如果是多个卡，那么一个卡的口down掉了，另外一个卡的口上是没有down掉卡的QP信息的，无法无损的进行failover。因为对于RDMA流量而言，RNIC是带有状态的"。
#### QP的归属
创建一个QP，这个QP只会在一个口上创建，不会在2个成员口中都存在。

#### 成员口的选择
传统的以太网的`bonding`，选择成员口的时候，比如 `bond mode4`, 可以选择`hash`策略为： `layer3+4`；
`RoceV2` 是 基于以太网`ethernet + IP/UDP(dport: 4791)`， 但是应该无法基于  `layer3+4` 进行`hash`来选择具体的哪个成员口。
因为 `Ethernet/IP/UDP`的封装是在 `RNIC`上进行的，应用层通过 `ibverbs` API 接口下发数据时，此时是没有五元组信息的，应该只有QP信息。

##### 能够基于QP信息得到五元组信息么？

##### 如何确定QP创建在哪个 slave 成员口？
**我的理解**：
由 `Linux bonding 驱动`和`底层 RDMA 驱动`在 QP 创建时（或首次使用时），根据配置的 `bonding` 模式（如 active-backup 的当前 active 口，或 load balancing 模式的哈希结果）隐式、透明地选择一个`活动的 slave 成员口`进行绑定。应用不直接指定。

##### 后续流量路径
**我的理解**：

 **核心原则**：绝大多数情况下，一旦一个 QP 被绑定到一个 slave 成员口后，这个 QP 的所有流量（发送和接收）都会固定通过这个指定的 slave 成员口。
这是为了**保证数据包的顺序性 (Ordering)**，这是 RDMA 可靠传输（RC, RD）和 RoCEv2 协议的一个重要要求。乱序到达的数据包会导致协议错误或性能下降。

原因，如下所示：
**（1）QP 状态关联：** 
QP 的状态机（SQ, RQ, CQ）与底层硬件队列深度绑定。这些硬件队列存在于特定的物理网卡（slave 成员口）上。将 QP 的流量分散到不同物理口的不同硬件队列上会破坏状态一致性。

**（2）顺序性保证：**
同一个 QP 发出的数据包序列必须保持发送顺序。如果它们走了不同的物理路径（即使负载均衡模式开启了），由于链路延迟、拥塞情况的差异，到达对端的顺序极有可能被打乱，违反协议要求。

**（3）连接上下文：**
QP 代表了一个通信端点对端点（或一对多）的连接上下文。这个上下文（包括序列号、确认号等）在特定物理口的硬件/驱动中维护。

##### 负载均衡单位
**我的理解**：
负载均衡（在支持的模式下）是在 **QP 粒度**上进行的（即不同的 QP 走不同的 slave 口），而不是在单个 QP 的流量内部进行分流。

##### 应用透明性
**我的理解**：
整个过程对 RDMA 应用是透明的（除了需要处理路径失效事件外）。应用只与 `bonding` 设备 (比如：`mlx5_bond_0`) 交互。

# RDMA多路径
## 方法：基于IP层的ECMP

- 原理：每个物理端口配置不同的IP地址，在主机上配置多个路由路径（通过多个下一跳）。然后利用IP层的多路径将流量分散到不同的端口。
- 流量策略：RDMA流量根据路由表选择路径，可以通过流哈希（如Flow Label）来维持同一个连接在一条路径上（避免乱序）。
- 注意点：
    - 需要交换机支持ECMP。
    - 需要主机配置多个IP地址和路由。
    - 需要RDMA应用支持多路径（或者通过RDMA CM管理多路径）

## 网卡的硬件bonding对比
RDMA多路径不是传统意义上的bonding，而是利用IP层的多路径。RDMA流量可以通过不同的路径传输，但需要交换机的支持。
一些RDMA网卡（如Mellanox）支持将多个物理端口绑定为一个逻辑端口（类似于LACP），这种绑定在网卡硬件层面完成，对上层透明。

# 参考
```bash

```