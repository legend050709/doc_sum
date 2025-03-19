```table-of-contents
```
# FC
# PFC
# 基于VLAN的PFC
# 基于DSCP的PFC
## 背景
基于VLAN的PFC的拥塞控制是逐跳工作的，源和目的服务器之间可能有多跳，如果有持续的网络拥塞，PFC暂停帧会从阻塞点传播并返回到源，这就会导致诸如`unfairness`和`victim flow`的问题。因此作者提出了基于`DSCP`的优先级流量控制机制PFC，替换掉PCP和VID来确保大规模部署。

# PFC的问题
## PFC unfairness 不公平问题
## PFC HOL 对头阻塞问题
## PFC DeadLock 死锁问题
## PFC Storm 风暴问题

# 参考
```bash
# RoCEv2 高性能传输协议与 Lossless 无损网络
https://is-cloud.blog.csdn.net/article/details/145773002
```