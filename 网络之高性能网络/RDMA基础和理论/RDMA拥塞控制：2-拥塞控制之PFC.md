```table-of-contents
```
# FC
# QOS
## SL（Service Level）
## VL（Virtual Lanes）
# PFC
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
# RoCEv2 高性能传输协议与 Lossless 无损网络
https://is-cloud.blog.csdn.net/article/details/145773002

# RDMA(6)流控：让数据流动如沐春风，为数据传输保驾护航
https://mp.weixin.qq.com/s/BjV5U7URuRQbQP_q0-j_Ng
```