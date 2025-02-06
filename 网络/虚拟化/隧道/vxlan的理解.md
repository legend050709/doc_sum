```table-of-contents
```
# 配置和查看
## 查看
```bash
# ip -d link show
27: vxlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UNKNOWN mode DEFAULT qlen 1000
    link/ether f2:fb:dd:9f:36:82 brd ff:ff:ff:ff:ff:ff promiscuity 0
    vxlan id 1 local 192.20.28.1 srcport 0 0 dstport 4789 ageing 300 addrgenmode eui64
```
# 应用场景
## 场景一：某个设备建立一对多的隧道，基于内层五元组选择具体的隧道
### 思路
（1）多个对端设备发布相同的Vtep，即anycast Vtep。那么本端机器，其实就是一对一的vxlan隧道。
基于外层隧道的五元组hash决定将流量导给哪个对端设备。

（2）多个对端设备发布不同的vtep。本端机器，建立多个vxlan隧道。

### 方法
# 参考
```bash

```