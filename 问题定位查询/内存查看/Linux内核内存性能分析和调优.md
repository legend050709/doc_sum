```table-of-contents
```
# 前言
在工作生活中，我们时常会遇到一些性能问题：比如手机用久了，在滑动窗口或点击 APP 时会出现页面反应慢、卡顿等情况；比如运行在某台服务器上进程的某些性能指标（影响用户体验的 PCT99 指标等）不达预期，产生告警等；造成性能问题的原因多种多样，可能是网络延迟高、磁盘 IO 慢、调度延迟高、内存回收等，这些最终都可能影响到用户态进程，进而被用户感知。

# 概述
在 Linux 服务器场景中，内存是影响性能的主要因素之一，本文从内存管理的角度，总结归纳了一些常见的影响因素（比如**内存回收、Page Fault 增多、跨 NUMA 内存访问**等），并介绍其对应的调优方法。

# 内存回收

# Huge Page

# mmap_lock 锁

# 跨 numa 内存访问

# 总结

# 参考
```bash
# Linux内核内存性能调优的一些笔记
https://mp.weixin.qq.com/s/S0sc2aysc6aZ5kZCcpMVTw

# 内存满了，会发生什么？
https://xiaolincoding.com/os/3_memory/mem_reclaim.html#%E5%B0%BD%E6%97%A9%E8%A7%A6%E5%8F%91-kswapd-%E5%86%85%E6%A0%B8%E7%BA%BF%E7%A8%8B%E5%BC%82%E6%AD%A5%E5%9B%9E%E6%94%B6%E5%86%85%E5%AD%98
```