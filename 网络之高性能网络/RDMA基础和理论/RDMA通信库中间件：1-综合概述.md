```table-of-contents
```
# 概述
# 市面上的统一多通信库
## rsocket
## mellanox的VMA
如果能使用我们熟悉的 socket 编程接口进行 RDMA 数据传输多好呀!

Mellanox 的 VMA（Voltaire Messaging Accelerator）就是这么一种用户空间的高性能网络库，旨在加速基于以太网和 InfiniBand 的网络通信。VMA 主要用于低延迟、高吞吐量的应用场景，特别是在金融服务（如高频交易）、电信和大规模分布式系统中。它通过绕过内核网络协议栈，直接与网卡硬件交互，从而减少网络延迟并提高应用程序性能。**VMA 支持标准的 BSD 套接字 API**，使现有应用程序能够无缝迁移，无需对代码进行大规模修改。VMA 与 Mellanox 网卡（如 ConnectX 系列）深度集成，充分利用网卡的硬件加速功能。

只需要在正常的加上`LD_PRELOAD=libvma.so`即可让你的程序支持 RDMA:
```bash
# 服务端  
LD_PRELOAD=libvma.so VMA_STATS_FD_NUM=500 ./src/redis-server --bind 192.168.3.11 --port 6379 --protected-mode no --save  
# 客户端  
LD_PRELOAD=libvma.so VMA_STATS_FD_NUM=500 ./src/redis-benchmark -h 192.168.3.11 -p 6379 -n 1000000 -t set -c 100 -d 3000
```

VMA 通过 `LD_PRELOAD` 的方式拦截应用程序中的标准 `socket` 调用，将其重定向到用户空间库中处理。这种设计允许 VMA 优化部分或全部网络通信，而无需对应用程序代码进行显著修改。

不过 vma 主要和 Mellanox 网卡集成，对 Intel 或其他厂商的网卡的支持非常有限，通常不作为官方支持的目标。

## UCX
## 阿里的X-RDMA


## 其他

# 参考
```bash

```