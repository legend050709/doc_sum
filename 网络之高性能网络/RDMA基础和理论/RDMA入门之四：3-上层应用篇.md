```table-of-contents
```

# RDMA学习
## 开源项目

- [RDMA-core项目](https://github.com/linux-rdma/rdma-core)：提供很多实际RDMA应用的基础结构。
- [StarryVae/RDMA-tutorial](https://github.com/StarryVae/RDMA-tutorial)：包括学习笔记、示例代码和相关文档。
- [HaifengSun-Kira/RDMA-Tutorial](https://github.com/HaifengSun-Kira/RDMA-Tutorial)：提供RDMA详细学习资料和示例代码。

## 使用库

mellanox的OFED库特别是`libibverbs`和`librdmacm`是入门者的必备工具。

## 小结
==对于简单项目，可以看下 `rdma-core` 中的 `example` 下的范例，比如 `ud_pingping 和 rc_pingping`。 然后就是测试工具: `perftest`== .


# X-RDMA

## 介绍
阿里RDMA通信库`X-RDMA`；X-RDMA是一款通信中间件，部署并大量用于阿里的云存储和数据库系统。

X-RDMA很多功能是业务需求推动的，比如稳健性、高效的资源管理以及用于调试和性能调整的便捷工具。

## X-RDMA收益
X-RDMA已经在阿里大量使用，几乎所有的基于RDMA的应用都在使用X-RDMA。X-RDMA收益如下：
- 网络吞吐提升24%
- 网络延迟降低5%（相比于ucx-am-rc）


# UCX
## 其他

### UCX和NCCL
NCCL：面向GPU智算提供集合通讯库能力；
UCX：面向CPU通算提供通用通讯库能力；


### 统一通信框架(UCF)
统一通信框架（UCF，Unified Communication Framework）是为了促进行业、实验室和学术界之间的合作，为高性能应用程序创建生产级通信框架和开放标准。

#### UCF组织成员
UCF组织成员：
![](attachments/Pasted%20image%2020250314164629.png)

#### 主要项目
目前只看到6个头部玩家的logo。

统一通信框架（UCF）包含以下几个主要项目：

**1）Unified Communication X (UCX)**：这是一个开源框架，支持MPI和SHMEM/PGAS编程模型。

**2）UCX for Apache Spark**：一个开源框架，支持使用UCX加速Apache Spark网络。

**3）Open RDMA**：支持RDMA Core开发和基于RDMA的开发。

**4）OpenSNAPI**：这是一个开源、标准的应用程序接口，用于在网络上访问计算引擎或智能网卡。

**5）UCC (Unified Collective Communication)**：这是一个开源项目，提供集合通信操作的API和库实现。

**6）HPCA Benchmark**：旨在创建一个新的指标，用于评价现代高性能计算（HPC）和人工智能系统的性能。

以上这些项目共同构成了UCF的生态系统，旨在推动高性能通信和数据处理技术的发展。


## 参考
```bash
# 【网络】UCX（Unified Communication X )|统一抽象通信接口
https://blog.csdn.net/bandaoyu/article/details/125207112

# 统一通信框架(UCF)
https://mp.weixin.qq.com/s/rSqpxMO04UhRORAi8_AMvA
```

# MPI
[# MPI，OpenMPI 与深度学习](https://zhuanlan.zhihu.com/p/158584571)

```bash
# 分布式入门（一）- 通信原语和通信库
https://zhuanlan.zhihu.com/p/679523886

# 分布式入门（二）- MPI 的进程组与进程拓扑
https://zhuanlan.zhihu.com/p/703695114
```

# brpc-rdma


# IPoIB
## 介绍
IPoIB（Internet Protocol over InfiniBand）是指利用IB物理网络设备(包括IB网卡、IB线缆、IB交换机等)，完美支持基于TCP/IP协议编写的应用程序无需作出任何修改可以在IB链路上直接进行数据通信。

## IPoIB体系结构

IPoIB的体系架构如图1所示。在Linux操作系统中，IPoIB协议是作为标准Linux网络驱动程序的一部分实现的，任何基于 TCP/IP 协议栈的应用程序或内核驱动程序无需修改即可使用InfiniBand 传输。
使用IPoIB网络接口发送数据，需要经过内核的网络协议栈，无法从InfiniBand 设备的内核旁路、零拷贝等功能中受益，因此IPoIB性能比RDMA通信方式性能要低。

![](attachments/Pasted%20image%2020250313104008.png)

# 参考
```bash
# 0. 《RDMA杂谈》专栏索引
https://zhuanlan.zhihu.com/p/164908617

# RDMA 在典型场景下的技术应用分析与探索https://my.oschina.net/u/4273516/blog/10094826

# 系列解读 SMC-R：透明无感提升云上 TCP 应用网络性能（一）
https://www.infoq.cn/article/l2chlcblsb1kczu1m9o2

# 性能提升 57% ，高性能网络协议 SMC-R 透明加速 TCP 实战解析
https://linux.cn/article-14612-1.html

# SMC-R 透明加速 TCP 技术，在 Redis 场景下的应用实践
https://openanolis.cn/blog/detail/1110588606592388285

# 存储大师班 | RDMA简介与编程基础
https://zhuanlan.zhihu.com/p/387549948
```