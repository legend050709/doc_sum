```table-of-contents
```
# 概述
网卡bond，即网卡绑定。网卡绑定有多种叫法：Port Trunking, Channel Bonding, Link Aggregation, NIC teaming等等。主要是将多个物理网卡绑定到一个逻辑网卡上。通过绑定可以达到链路冗余、带宽扩容、负载均衡等目的。网卡bond一般主要用于网络吞吐量很大，以及对于网络稳定性要求较高的场景，是生产场景中提高性能和可靠性的一种常用技术。

Linux内置了网卡绑定的驱动程序，可以将多个物理网卡分别捆绑成多个不同的逻辑网卡（例如把eth0、eth1捆绑成bond0，把eth2、eth3捆绑成bond1）。

# 模式
在 Linux 中，网络接口的 Bonding 模式允许将多个物理网络接口聚合成一个逻辑接口，以提高带宽和冗余性。不同的 Bonding 模式提供了不同的负载均衡和故障转移策略。

对应于不同的负载均衡和容错特性需求，Linux网卡bond的模式共有bond0－bond6共7种。
## mode=0（balance-rr）

表示负载分担round-robin，并且是轮询的方式，顺序在每个slave接口上面发送数据包，直到数据包发送完毕。提供负载均衡和容错的能力，总带宽是两个接口的带宽总和。实际使用中需要接入交换机做端口聚合。

## mode=1（active-backup）

表示主备模式，即同一时间时只有一个slave被激活，只有活动的slave接口失效时，其他的slave才会被激活。这种模式可以提供较高的冗余性，但是链路利用率低，不论几块网卡只有1块在工作。


## mode=2(balance-xor)(平衡策略)
## mode=3(broadcast)(广播策略)
## mode=4(802.3ad)(IEEE 802.3ad 动态链接聚合)

表示支持802.3ad协议，和交换机的聚合LACP方式配合（需要xmit_hash_policy）。协议标准要求所有设备在聚合操作时，要在同样的速率和双工模式。通过创建一个聚合组，共享同样的速率和双工设定。根据802.3ad规范将多个slave工作在同一个激活的聚合体下。外出流量的slave选举是基于传输hash策略，该策略可以通过xmit_hash_policy选项从缺省的XOR策略改变到其他策略。


## mode=5(balance-tlb)(适配器传输负载均衡)

这种模式是根据每个slave的负载情况选择slave进行发送，接收时使用当前轮到的slave。它不需要任何特别的switch(交换机)支持的通道bonding。在每个slave上根据当前的负载（根据速度计算）分配外出流量。如果正在接受数据的slave出故障了，另一个slave接管失败的slave的MAC地址。


## mode=6(balance-alb)(适配器适应性负载均衡)

在5的tlb基础上增加了rlb(接收负载均衡receiveload balance).不需要任何switch(交换机)的支持。接收负载均衡是通过ARP协商实现的。总带宽是每个网口的带宽综合。

## 总结
mode 1、mode 5和mode 6不需要交换机端的设置，网卡能自动聚合。mode 4需要支持802.3ad。mode 0，mode 2和mode 3理论上需要静态聚合方式。  

bond模式的选择要根据应用中的需要进行选择，在实际环境中，mode 0，1，4，6使用较多。
如果交换机及网卡都确认支持802.3ad，则实现负载均衡时尽量使用mode 4以提高系统性能。

# 特性
## bond口和成员口具有相同的mac

![](attachments/Pasted%20image%2020240627104444.png)



# 配置
## 临时配置
需要创建一个 bonding 接口，通常通过以下步骤完成：
```bash
# 安装 bonding模块
modprobe bonding

# 创建 bond 接口
ip link add bond0 type bond mode 802.3ad

# 添加物理接口到 bond 接口
ip link set eth0 master bond0
ip link set eth1 master bond0

# 启动 bond 接口和物理接口
ip link set bond0 up
ip link set eth0 up
ip link set eth1 up
```

## 开机生效配置

![](attachments/Pasted%20image%2020240904112945.png)

# 查看
## bond0的生效信息
```bash
# cat /proc/net/bonding/eth02
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: IEEE 802.3ad Dynamic link aggregation
Transmit Hash Policy: layer3+4 (1)
MII Status: up
MII Polling Interval (ms): 100
Up Delay (ms): 0
Down Delay (ms): 0

802.3ad info
LACP rate: slow
Min links: 0
Aggregator selection policy (ad_select): stable
System priority: 65535
System MAC address: 5c:6f:69:c8:32:c0
Active Aggregator Info:
	Aggregator ID: 1
	Number of ports: 2
	Actor Key: 21
	Partner Key: 3265
	Partner Mac Address: 3c:c7:86:43:ac:f1

Slave Interface: lan03
MII Status: up
Speed: 25000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 5c:6f:69:c8:32:c0
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 5c:6f:69:c8:32:c0
    port key: 21
    port priority: 255
    port number: 1
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 3c:c7:86:43:ac:f1
    oper key: 3265
    port priority: 32768
    port number: 11
    port state: 61

Slave Interface: lan04
MII Status: up
Speed: 25000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 5c:6f:69:c8:32:c1
Slave queue ID: 0
Aggregator ID: 1
Actor Churn State: none
Partner Churn State: none
Actor Churned Count: 0
Partner Churned Count: 0
details actor lacp pdu:
    system priority: 65535
    system mac address: 5c:6f:69:c8:32:c0
    port key: 21
    port priority: 255
    port number: 2
    port state: 61
details partner lacp pdu:
    system priority: 32768
    system mac address: 3c:c7:86:43:ac:f1
    oper key: 3265
    port priority: 32768
    port number: 12
    port state: 61
```
## 查看当前速率
```bash
ethtool bond0
```


# dpdk 接口bond

DPDK支持网卡的bond，通过调用一系列的API完成bond接口的创建，初始化和报文收发动作。bond口和普通dpdk口通过统一的API实现网卡的初始化、报文的收发等操作。

## 简介
参考：[# 链路绑定PMD](https://dpdk-docs.readthedocs.io/en/latest/prog_guide/link_bonding_poll_mode_drv_lib.html)

除了用于物理和虚拟硬件的轮询模式驱动程序（PMD）之外，DPDK还包括一个纯软件库，可将多个物理PMD绑定在一起以创建单个逻辑PMD。

![](attachments/Pasted%20image%2020240904113547.png)

Link Bonding PMD库（librte_pmd_bond）支持绑定相同速度和双工的rte_eth_dev端口组，以提供类似于Linux绑定驱动程序中的功能，以允许将多个（从属）NIC聚合到服务器和交换机中的单个逻辑接口。然后，新的聚合的PMD将根据指定的操作模式处理这些接口，以支持冗余链路，容错和/或负载均衡等功能。

librte_pmd_bond库导出一个C语言API，包括用于创建绑定设备的API，以及配置和管理绑定设备及其从属设备的API。

## DPDK支持的bond模式

![](attachments/Pasted%20image%2020240904105710.png)

### 主动备份（模式1）

![](attachments/Pasted%20image%2020240904113843.png)

 在此模式下，在任何时间只有一个从设备处于活动状态，当且仅当当前活跃从设备发生故障时，不同的从设备才会激活，从而为故障设备提供容错。单个逻辑绑定接口的MAC地址只能在一个NIC（端口）上外部可见，以避免交换机混淆( to avoid confusing the network switch.)。



## bond口创建的一般流程

Step 1、创建slave口
Step 2、slave口配置网卡队列、网口启动
Step 3、创建bond口
Step 4、bond口添加slave口
Step 5、bond口配置网卡队列、网口启动
Step 6、通过bond口id进行收发包

DPDK提供了一系列的库函数实现bond功能。

![](attachments/Pasted%20image%2020240904113215.png)

### 创建bond口
rte_eth_bond_create，通过指定网口名称、bond模式、numa节点id，以确定资源的分配，函数原型如下：

bond口和普通dpdk口的id不会冲突，如dpdk接管了2个网口，id分别是0和1，这两个口绑定的bond口，id为2.

![](attachments/Pasted%20image%2020240904105835.png)

### 负载均衡策略设置
rte_eth_bond_xmit_policy_set，bond口运行在mode 2时，支持多种形式的负载均衡策略，通过rte_eth_bond_xmit_policy_set指定使用哪种策略：

![](attachments/Pasted%20image%2020240904105911.png)

可以设置的负载均衡策略包含以下几种：
```bash
BALANCE_XMIT_POLICY_LAYER2 :根据MAC地址负载均衡
BALANCE_XMIT_POLICY_LAYER23 :根据MAC+IP地址负载均衡
BALANCE_XMIT_POLICY_LAYER34 :根据IP+端口负载均衡
```

### 添加bond成员口
rte_eth_bond_slave_add，指定bond口由哪些dpdk口聚合而成。函数原型如下：

![](attachments/Pasted%20image%2020240904110000.png)


### 获取指定bond口的成员口数目

rte_eth_bond_active_slaves_get，获取指定bond口的状态正常的slave口数目，返回正常slave口的数目和port id列表。函数原型如下：

![](attachments/Pasted%20image%2020240904110040.png)

### 释放bond口
rte_eth_bond_free，释放一个已经创建的bond口。函数原型如下：

![](attachments/Pasted%20image%2020240904110102.png)


## 范例
参考：[# f-stack dpdk bond 功能](https://blog.csdn.net/shaoyunzhe/article/details/72902251)



# 参考
```bash

```