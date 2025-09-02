```table-of-contents
```
# FC
# QOS
## SL（Service Level）
## VL（Virtual Lanes）

# PFC特点
## RoCE v2用PFC做链路层流控
### 链路层

计算机网络通常按**OSI七层模型**来划分：

|层级|名称|主要功能|
|---|---|---|
|7|应用层|应用程序之间的数据交互|
|6|表示层|数据格式转换、加密等|
|5|会话层|建立、管理会话连接|
|4|传输层|端到端数据传输、流控和差错控制（如TCP）|
|3|网络层|路由选择与IP寻址|
|**2**|**链路层**|**局域网内数据帧的传输，错误检测与流控**|
|1|物理层|物理信号传输（电缆、光纤等）|

**链路层**主要负责相邻网络设备之间的数据帧传输，通常是局域网内部的通信，比如以太网交换机和网卡之间。它包含了以太网MAC地址、帧格式、错误检测和局部流控等。

**链路层流控** 是在两个直接相连的设备（例如交换机端口和服务器网卡）之间，控制数据传输速度的一种机制。
PFC通过发送特殊的以太网PAUSE帧，告诉对端“请暂停发送某个优先级的数据”，以避免对端继续发送导致接收端缓冲区溢出。
这种机制只在相邻链路（链路层连接）上生效，**它不跨越多跳路由**，只对直接链路有效。


### 流控(flow control)非拥塞控制(cc)

### RoceV2和PFC
**RoCE v2** 是在以太网UDP基础上实现的RDMA协议，利用UDP封装RDMA数据包，使其可以跨越多跳IP网络。
UDP本身不可靠，也不提供流控或拥塞控制。如果交换机端口缓冲区满了，UDP包就会丢失，严重影响RoCE性能。
因此，RoCE v2依赖底层链路层机制——PFC来实现精细的流控，阻止发送端继续发包，保证端口不丢包。

在传统以太网中，Ethernet流控机制是基于PAUSE帧实现的，但PAUSE是全局暂停，会影响所有流。
PFC（Priority Flow Control）则是基于`IEEE 802.1Qbb`标准，实现按优先级（Priority）单独流控，避免影响所有流。

### 链路层流控和传输层流控


## 有了PFC为什么还要拥塞控制


# PFC分类
# 基于VLAN的PFC
# 基于DSCP的PFC
## 背景
基于VLAN的PFC的拥塞控制是逐跳工作的，源和目的服务器之间可能有多跳，如果有持续的网络拥塞，PFC暂停帧会从阻塞点传播并返回到源，这就会导致诸如`unfairness`和`victim flow`的问题。因此作者提出了基于`DSCP`的优先级流量控制机制PFC，替换掉PCP和VID来确保大规模部署。

# PFC的问题
## PFC unfairness 不公平问题
## PFC HOL 对头阻塞问题
![](attachments/Pasted%20image%2020250425022046.png)

 PFC基于优先级的流量控制协议可能如上图所示，当Port4流量达到阈值后，向Port1和Port2发送Pause Frame，但导致Port1与Port3的数据传输暂停，从而影响整体。
因此，由于RoCE缺乏类似TCP的流量控制和拥塞控制算法，在丢包情况下，性能表现极差。

## PFC DeadLock 死锁问题
## PFC Storm 风暴问题

# 参考
```bash
# 智能无损网络技术白皮书-6W104
https://www.h3c.com/cn/Service/Document_Software/Document_Center/Home/Public/00-Public/Learn_Technologies/White_Paper/WP-17074/

# RoCEv2 高性能传输协议与 Lossless 无损网络
https://is-cloud.blog.csdn.net/article/details/145773002

# RDMA(6)流控：让数据流动如沐春风，为数据传输保驾护航
https://mp.weixin.qq.com/s/BjV5U7URuRQbQP_q0-j_Ng
```