```table-of-contents
```
# 介绍
`Shared Memory Communication over RDMA` (SMC-R) 是一种基于 RDMA 技术、兼容 socket 接口的内核网络协议，由 IBM 提出并在 2017 年贡献至 Linux 内核。**SMC-R 能够帮助 TCP 网络应用程序透明使用 RDMA**，获得高带宽、低时延的网络通信服务。

# 参考
```bash
# 阿里巴巴的 SMC-R的高性能文章【系列】
https://openanolis.cn/sig/high-perf-network

# 系列解读 SMC-R：透明无感提升云上 TCP 应用网络性能（一）
https://www.infoq.cn/article/l2chlcblsb1kczu1m9o2

# 系列解读 SMC-R：融合 TCP 与 RDMA 的 SMC-R 通信（二）
https://www.infoq.cn/article/S825k3R50ZAKPlkFOwYZ

# 性能提升 57% ，SMC-R 透明加速 TCP 实战解析
https://www.infoq.cn/article/0v4hIYUxAH6QQXB8LkBh

# 性能透明提升 50%，详解 SMC + ERDMA 云上超大规模高性能网络协议栈
https://www.infoq.cn/article/dx0r5ahoom0fjb3ftuzr

#【实践】SMC-R透明加速TCP技术，在Redis场景下的应用实践
https://www.ieisystem.com/keyarchos/news/12927.html

# alibaba: 共享内存通信（SMC）适用性说明
https://www.alibabacloud.com/help/zh/alinux/user-guide/smc-applicability


# 基于 RDMA 的共享内存通信 (SMC-R) 【图不错；】
https://www.ibm.com/docs/zh/aix/7.3.0?topic=access-shared-memory-communications-over-rdma-smc-r
https://www.ibm.com/docs/en/aix/7.2.0?topic=access-shared-memory-communications-over-rdma-smc-r
```