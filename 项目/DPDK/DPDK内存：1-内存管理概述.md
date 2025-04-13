```table-of-contents
```
# 概述
DPDK 考虑了 NUMA 以及多内存通道的访问效率，会在系统运行前要求配置Linux 的 HugePage，初始化时申请其内存池，用于 DPDK 运行的主要内存资源。Linux 大页机制利用了处理器核上的的 TLB 的 HugePage 表项，这可以减少内存地址转换的开销。

# 内存多通道的使用
现代的内存控制器都支持内存多通道，比如 Intel 的 E5-2600V3 系列处理器，能支持 4个通道，可以同时读和取数据。依赖于内存控制器及其配置，内存分布在这些通道上。每一个通道都有一个带宽上限，如果所有的内存访问都只发生在第一个通道上，这将成为一个潜在的性能瓶颈。

因此，DPDK 的 mempool 库缺省是把所有的对象分配在不同的内存通道上，保证了在系统极端情况下需要大量内存访问时，尽可能地将内存访问任务均匀平滑。

# 内存拷贝
很多 libc 的 API 都没有考虑性能，因此，不要在高性能数据平面上用 libc 提供的 API，比如，memcpy()或 strcpy()。虽然 DPDK 也用了很多 libc 的 API，但均只是在软件配置方面用于方便程序移植和开发。

DPDK 提供了一个优化版本的 rte_memcpy() API，它充分利用了 Intel 的 SIMD 指令集，也考虑了数据的对齐特性和 cache 操作友好性。

# 内存分配
在某些情况下，应用程序使用 libc 提供的动态内存分配机制是必要的，如 malloc()函数，它是一种灵活的内存分配和释放方式。但是，因为管理零散的堆内存代价昂贵，并且这种内存分配器对于并行的请求分配性能不佳，所以不建议在数据平面处理上使用类似malloc()的函数进行内存分配。

在有动态分配的需求下，建议使用 DPDK 提供的 rte_malloc() API，该 API 可以在后台保证从本 NUMA 节点内存的 HugePage 里分配内存，并实现 cache line 对齐以及无锁方式访问对象等功能。

注：其实**对于数据面转发，其实更加建议的是，使用 mempool，提前申请好固定大小的内存空间**。

# NUMA考虑
NUMA（Non Uniform Memory Access Architecture）与 SMP（Symmetric Multi Processing）是两种典型的处理器对内存的访问架构。随着处理器进入多核时代，对于内存吞吐量和延迟性能有了更高的要求，NUMA 架构已广泛用于最新的英特尔处理器中，为每个处理器提供分离的内存和内存控制器，以避免 SMP 架构中多个处理器同时访问同一个存储器产生的性能损失。

在双路服务器平台上，NUMA 架构存在本地内存与远端内存的差异。本地和远端是个相对概念，是指内存相对于具体运行程序的处理器而言，如图所示。
![](attachments/Pasted%20image%2020240313215446.png)

在 NUMA 体系架构中，CPU 进行内存访问时，本地内存的要比访问远端内存更快。因为访问远端内存时，需要跨越 QPI 总线，在应用软件设计中应该尽量避免。在高速报文处理中，这个访问延迟会大幅降低系统性能。

DPDK 提供了一套在指定 NUMA 节点上创建 memzone、ring, rte_malloc 以及 mempool 的API，可以避免远端内存访问这类问题。在一个 NUMA 节点端，对于内存变量进行读取不会存在性能问题，因为该变量会在 CPU cache 里。但对于跨 NUMA 架构的内存变量读取，会存在性能问题，可以采取复制一份该变量的副本到本地节点（内存）的方法来提高性能。


# 预留内存
## 没有预留内存的问题
现在新版本的 DPDK默认都是按需申请大页，比如系统启动时，一共分配了100个大页，但是DPDK程序启动的时候，不会将100个大页全部占用，而是基于实际需要多少个大页，就从系统中申请多少个大页。

如果DPDK程序存在频繁的配置变更，每次配置下发都需要进行内存申请，然后进行释放。
正常是先从DPDK管理的内存中申请，如果DPDK管理的内存不足，则从系统中申请一个大页；使用完毕之后，DPDK应用将内存还给DPDK管理，DPDK管理时发现管理的内存超过一个大页且连续，可能又将大页还给了系统，即DPDK是按需管理内存。

如果存在大量的配置变更，DPDK需要频繁的从系统中申请大页，以及将大页还给系统。这个是比较耗时的。

在老版本的DPDK，系统申请100个大页，在DPDK系统启动的时候，直接从系统中申请100个大页全部给占用了，实际运行中，不存在频繁的从系统中申请大页以及释放大页给系统。

## DPDK中的预留内存
```bash
`--socket-mem`: Memory to allocate from hugepages on specific sockets. In dynamic memory mode, this memory will also be pinned (i.e. not released back to the system until application closes).
```

# DPDK内存整体架构
参考: [# DPDK内存管理概述](https://zhuanlan.zhihu.com/p/658824633)



# 参考
```bash

# DPDK内存管理概述
https://zhuanlan.zhihu.com/p/658824633

# DPDK 22.11内存管理变化解析
http://blog.chinaunix.net/uid-28541347-id-5877488.html

# DPDK内存管理——全网最全篇
https://zhuanlan.zhihu.com/p/702445686

# DPDK内存管理分析
https://blog.csdn.net/weixin_48329334/article/details/139059952

# 一文读懂DPDK rte_mempool创建、使用及信息查询
https://zhuanlan.zhihu.com/p/695112706

--socket-mem 参数说明
https://doc.dpdk.org/guides/linux_gsg/linux_eal_parameters.html
https://doc.dpdk.org/guides/linux_gsg/build_sample_apps.html

# DPDK性能影响因素分析
https://zhuanlan.zhihu.com/p/557294705
```