```table-of-contents
```
# 概述
`librte_pdump` 库是在 DPDK `16.07` 版本引入的一个 `DPDK` 数据包捕获开发框架，
`dpdk-pdump` Tool 就是基于`librte_pdump` 库开发出来的`DPDK App` 抓包工具。

DPDK自带有两个抓包工具，分别是`dpdk-pdump`和`dpdk-dumpcap`；
==`dpdk-pdump`是旧的抓包工具==，功能太简单，不支持按条件过滤报文。

因为 `DPDK App` 是完全内核旁路（`Kernel-bypass`）的用户态网络协议栈，所以无法使用 `tcpdump` 工具来进行抓包。使用 `dpdk-pdump` 可以用于抓取被 `DPDK App` 接管的指定接口、队列的数据包。

# dpdk 的 librte_pdump 库
# dpdk-dumpcap 工具
## 介绍
功能：作为备程序，抓取`dpdk`主程序进入，出去的流量，写入到文件中。  
前提条件：`dpdk`主程序中存在初始化包抓包框架，已知`testpmd`初始了该框架，其他的`dpdk`程序没有初始化。
```bash

```

# dpdk-pdump 工具


# 参考
```bash
# DPDK — PDUMP 抓包工具
https://blog.51cto.com/u_15301988/5181156
https://wenqingwu.github.io/2018/12/08/pdump-DPDK%E6%8A%93%E5%8C%85%E5%B7%A5%E5%85%B7/

# 官方文档
https://doc.dpdk.org/guides/howto/packet_capture_framework.html
# 官方pdump工具
https://doc.dpdk.org/guides/tools/pdump.html
# 官方dumpcap工具
https://doc.dpdk.org/guides/tools/dumpcap.html

# dpdk-dumpcap 存在的问题
https://www.cnblogs.com/CQzhangyu/p/18406754
```