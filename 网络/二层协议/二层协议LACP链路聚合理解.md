```table-of-contents
```
# 介绍
链路聚合控制协议（LACP）是一种用于将多个物理链路聚合为一个逻辑链路的协议，以提高带宽和冗余。

# 基本概念
## 聚合组（Bonding Group）
将多个物理接口聚合为一个逻辑接口。

# LACP角色
- **主机（Actor）**：发起聚合的设备。
- **对端（Partner）**：响应聚合请求的设备。

# LACP 端口状态
- **活动（Active）**：端口处于活动状态，参与聚合。
- **聚合（Aggregated）**：端口成功聚合。
- **同步（Synchronized）**：端口与对端的状态同步。

# 配置LACP
## 前提条件
- **配置一致**：确保所有参与聚合的端口配置一致，包括速率、双工模式和 VLAN 设置。
- **链路正常**：确保所有物理链路都处于良好状态，没有故障或不良连接。
- **设备兼容性**：确保对端设备也支持 LACP，并且配置正确。

## 负载均衡

# LACP协商异常的分析
## LACP异常情况的范例
异常一：
![](attachments/Pasted%20image%2020241219144030.png)

异常二：
![](attachments/Pasted%20image%2020241219144429.png)

### LACP正常情况下的分析

![](attachments/Pasted%20image%2020241219144228.png)

## 抓取LACP报文
```bash
tcpdump -nni ethx not ip and not ip6 and not arp and ether proto not 0x88cc -vvv -e 
```

# 参考
```bash

```