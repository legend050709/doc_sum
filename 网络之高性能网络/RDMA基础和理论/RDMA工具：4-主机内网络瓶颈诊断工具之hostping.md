```table-of-contents
```

# 背景
近几年来，云业务数据量激增，RDMA网卡速率也随之迅速提升，100G/200G RDMA网卡已在数据中心内得到了广泛应用。
在RDMA网络中，主机内网络通常被认为是稳健的，很少受到关注。然而，随着RNIC速率迅速增加到200G/400G，主机内网络成为网络应用的潜在性能瓶颈。包括主机内带宽降低和主机内延迟增加等，这会严重影响云业务性能。现有的主机内性能分析工具无法有效诊断主机内网络瓶颈，需耗时数小时甚至数天时间来定位故障。

# hostping 
## Hostping核心思想
Hostping的核心思想是在RNIC和主机内的端点（GPU、内存）之间进行环回(loopback)测试，以测量主机内的延迟和带宽，并利用测量的数据推断主机内的瓶颈。
如果在环回测试中没有发现异常，推断瓶颈发生在网络中。

## 框架图
![](attachments/Pasted%20image%2020250316131600.png)


# 其他
紫金山实验室未来网络研究中心研究员张娇与字节跳动高速网络团队共同合作完成的论文《Hostping: Diagnosing Intra-host Performance Bottlenecks in RDMA Servers》被NSDI 2023会议录用，这是紫金山实验室在该会议发表的首篇论文。

# 参考
```bash

# hostping：主机内网络瓶颈诊断方法 [NSDI2023]
https://mp.weixin.qq.com/s/-jPfC77NhDqnAFF3OW1r5g

# 【NSDI'23】Hostping：诊断RDMA服务器中的主机内性能瓶颈
https://mp.weixin.qq.com/s/NWaz_vX_qiyB5qUE0XkP3A
```