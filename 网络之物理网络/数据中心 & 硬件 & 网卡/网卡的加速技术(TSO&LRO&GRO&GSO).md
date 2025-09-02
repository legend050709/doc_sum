```table-of-contents
```
# NAPI
# Jumbo Frames
# offload技术
## TSO
### 背景
通常，以太网的MTU是1500B，除去TCP/IP的协议首部，TCP的MSS(Max Segment Size)大小是1460B。一般情况下，协议栈会对超过1460B的TCP payload进行切片，保证生成的IP包不超过MTU的大小，但对于支持TSO的网卡，协议栈的TCP层不进行segment, 我们可以把最多64KB大小的TCP payload直接往下传给协议栈，此时IP层也不会进行fragment，一直会传给网卡驱动，支持TSO的网卡会自己进行TCP切片，这样可以减少很多协议栈上的数据包处理数量。切片、checksum计算等原本靠CPU来做的工作都转移给了网卡，从而提高网络性能。相对应的，LRO(Large Receive Offload)是在接收方向上，通过将接收到的多个TCP数据聚合成一个大的数据包，然后传递给网络协议栈处理，以减少上层协议栈处理的开销。当然，如果都是小包，那么功能基本就作用不大了。

TCP Segmentation Offload （TSO），UDP Fragmentation Offload （UFO）和Large Receive Offload （LRO）技术基于网卡特性，可在网卡上进行包合并和拆分，减轻CPU负荷。其中，TSO是针对TCP的拆包，UFO是针对UDP的拆包，LRO是针对TCP的合包。然而，LRO、TSO和UFO只能处理TCP和UDP包，而且并非所有的网卡都支持这些特性。**而软件包合并（Generic Receive Offload，GRO）和包拆分（Generic Segmentation Offload，GSO）在网卡不支持分片、重组offload能力（如TSO、UFO、LRO）的情况下，GSO推迟数据分片直至数据发送到网卡驱动之前进行分片后再发往网卡，GRO将大量的小报文合并为少量的大报文，再将合并后的大报文提交给OS协议栈处理。同时GSO、GRO不仅支持TCP和UDP包，还可支持vxlan和gre。**

## LRO
## GRO
## GSO
## checksum offload

# CQE compress
# RSS
# FDIR
# Scatter/Gather
# DMA


# 参考
```bash
# 常见网络加速技术浅谈（一）
https://zhuanlan.zhihu.com/p/44635205
https://zhuanlan.zhihu.com/p/44683790
# 系列文章都不错 （++++++++++++）
https://www.zhihu.com/column/software-defined-network


```
