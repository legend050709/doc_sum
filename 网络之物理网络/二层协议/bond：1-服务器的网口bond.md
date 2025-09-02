```table-of-contents
```
# 概述
网卡bond，即网卡绑定。网卡绑定有多种叫法：Port Trunking, Channel Bonding, Link Aggregation, NIC teaming等等。主要是将多个物理网卡绑定到一个逻辑网卡上。通过绑定可以达到链路冗余、带宽扩容、负载均衡等目的。网卡bond一般主要用于网络吞吐量很大，以及对于网络稳定性要求较高的场景，是生产场景中提高性能和可靠性的一种常用技术。

Linux内置了网卡绑定的驱动程序，可以将多个物理网卡分别捆绑成多个不同的逻辑网卡（例如把eth0、eth1捆绑成bond0，把eth2、eth3捆绑成bond1）。

# 模式
在 Linux 中，网络接口的 Bonding 模式允许将多个物理网络接口聚合成一个逻辑接口，以提高带宽和冗余性。不同的 Bonding 模式提供了不同的负载均衡和故障转移策略。

对应于不同的负载均衡和容错特性需求，Linux网卡bond的模式共有bond0－bond6共7种。
## mode=0（balance-rr）

表示负载分担`round-robin`，并且是轮询的方式，顺序在每个slave接口上面发送数据包，直到数据包发送完毕。提供负载均衡和容错的能力，总带宽是两个接口的带宽总和。实际使用中需要接入交换机做端口聚合。

### 特点
传输数据包顺序是依次传输（即：第1个包走eth0，下一个包就走eth1….一 直循环下去，直到最后一个传输完毕），此模式提供负载平衡和容错能力；

但是我们知道如果一个连接或者会话的数据包从不同的接口发出的话，中途再经过不同的 链路，在客户端很有可能会出现数据包无序到达的问题，而无序到达的数据包需要重新要求被发送，这样网络的吞吐量就会下降。

## mode=1（active-backup）

表示主备模式，即同一时间时只有一个slave被激活，只有活动的slave接口失效时，其他的slave才会被激活。这种模式可以提供较高的冗余性，但是链路利用率低，不论几块网卡只有1块在工作。

### 特点
（1）只有一个设备处于活动状态，当一个宕掉另一个马上由备份转换为主设备。
（2）mac地址是外部可见得，从外面看来，bond的MAC地址是唯一的，以避免switch(交换机)发生混乱。

此模式只提供了容错能力；由此可见此算法的优点是可以提供高网络连接的可用性，但是它的资源利用率较低，只有一个接口处于工作状态，在有 N 个网络接口的情况下，资源利用率为1/N

## mode=2(balance-xor)(平衡策略)
### 特点
基于指定的传输HASH策略传输数据包。缺省的策略是：(源MAC地址 XOR 目标MAC地址)% slave数量。
其他的传输策略可以通过`xmit_hash_policy`选项指定，此模式提供负载平衡和容错能力。

## mode=3(broadcast)(广播策略)
## mode=4(802.3ad)(IEEE 802.3ad 动态链接聚合)

表示支持802.3ad协议（802.3ad Dynamic link aggregation），和交换机的聚合LACP方式配合（需要`xmit_hash_policy`）。

### 特点
协议标准要求所有设备在聚合操作时，**要在同样的速率和双工模式**。
通过创建一个聚合组，共享同样的速率和双工设定。根据802.3ad规范将多个slave工作在同一个激活的聚合体下。
外出流量的slave选举是基于传输`hash`策略，该策略可以通过`xmit_hash_policy`选项从缺省的`XOR`策略改变到其他策略。



### 常用 `xmit_hash_policy` 选项

|策略值|名称|哈希计算依据|适用场景|
|---|---|---|---|
|`layer2`|Layer 2|源/目标 MAC 地址|默认策略，简单但负载不均|
|`layer2+3`|Layer 2+3|MAC + IP 地址|基础 IP 负载均衡|
|**`layer3+4`**|Layer 3+4|IP 地址 + 端口号|**推荐策略** (TCP/UDP 流量)|
|`encap2+3`|Encapsulation 2+3|外层 MAC + IP|VLAN/隧道环境|
|`encap3+4`|Encapsulation 3+4|外层 IP + 端口|高级隧道环境|


对于大多数 IP 网络（尤其是 TCP/UDP 流量），**`layer3+4` 是最佳选择**

#### 配置以及验证
```bash
配置：
echo layer3+4 > /sys/class/net/bond0/bonding/xmit_hash_policy

验证：
cat /sys/class/net/bond0/bonding/xmit_hash_policy
```


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

# 设置发送的hash策略，默认是l2(简单但负载不均);
echo layer3+4 > /sys/class/net/bond0/bonding/xmit_hash_policy

# 添加物理接口到 bond 接口
ip link set eth0 master bond0
ip link set eth1 master bond0

#将物理口从bond口中删除
ip link set eth0 nomaster
ip link set eth1 nomaster

# 启动 bond 接口和物理接口
ip link set bond0 up
ip link set eth0 up
ip link set eth1 up

# 删除bond口
ip link del bond0
```


或者
```bash
# 加载bonding模块时指定参数 
modprobe bonding mode=802.3ad xmit_hash_policy=layer3+4 

# 创建 bond 
ip link add bond0 type bond

注：此方法会影响所有后续创建的 Bond
```



## 开机生效配置

![](attachments/Pasted%20image%2020240904112945.png)

## 检查
### 正常的bond 配置
![](attachments/Pasted%20image%2020250704113252.png)

如上所示，如果bond 模式使用 `mode=4(802.3ad)(IEEE 802.3ad 动态链接聚合), xmit_hash_policy 为 layer3+4`; 
`bond`的2个成员口的 `state 是 ACTIVE`； 具有相同的 ad_aggregator_id， 表示在一个组内，然后 `ad_actor_oper_port_state` 和  `ad_partner_oper_port_state` 都应该是 61。

**参数**：
`ad_actor_oper_port_state`：本机端口的状态，
`ad_partner_oper_port_state`：对端（交换机）端口的状态。

**状态值**：
```c
// linux/drivers/net/bonding/bond_3ad.c

/* Port state definitions (43.4.2.2 in the 802.3ad standard) */
#define AD_STATE_LACP_ACTIVITY   0x1
#define AD_STATE_LACP_TIMEOUT    0x2
#define AD_STATE_AGGREGATION     0x4
#define AD_STATE_SYNCHRONIZATION 0x8
#define AD_STATE_COLLECTING      0x10
#define AD_STATE_DISTRIBUTING    0x20
#define AD_STATE_DEFAULTED       0x40
#define AD_STATE_EXPIRED         0x80

状态为 61 的解析：
61的二进制是00111101 = AD_STATE_LACP_ACTIVITY + AD_STATE_AGGREGATION + AD_STATE_SYNCHRONIZATION + AD_STATE_COLLECTING + AD_STATE_DISTRIBUTING；

63的二进制是00111111 = 61 + AD_STATE_LACP_TIMEOUT;
```

### 异常的bond 配置

![](attachments/Pasted%20image%2020250704112223.png)

如上所示，bond1的2个成员口，eth05和eth06的状态就不对。

#### 影响
有可能通过 bond1 的流量，时通时不通。比如：`tcp telnet` 时通时不通。

# 查看
## bond0的生效信息
```bash
# cat /proc/net/bonding/bond0
Ethernet Channel Bonding Driver: v3.7.1 (April 27, 2011)

Bonding Mode: IEEE 802.3ad Dynamic link aggregation // bond mode
Transmit Hash Policy: layer3+4 (1)  // hash策略
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

Slave Interface: lan03  //成员口的信息
MII Status: up
Speed: 25000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 5c:6f:69:c8:32:c0
Slave queue ID: 0
Aggregator ID: 1 // 聚合的id
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
    port state: 61 // 成员口的状态
details partner lacp pdu:
    system priority: 32768
    system mac address: 3c:c7:86:43:ac:f1
    oper key: 3265
    port priority: 32768
    port number: 11
    port state: 61 // 成员口互联设备的的状态

Slave Interface: lan04
MII Status: up
Speed: 25000 Mbps
Duplex: full
Link Failure Count: 0
Permanent HW addr: 5c:6f:69:c8:32:c1
Slave queue ID: 0
Aggregator ID: 1 // 聚合的id
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

#  LAG和LACP
## LAG

### 介绍
LAG (Link Aggregation Group - 链路聚合组) 不是一个协议，而是一个**逻辑概念**或**逻辑接口**。它指的是将多个物理以太网端口（成员端口）捆绑在一起，形成一个单一的、逻辑上的高带宽链路。

**核心目标：** 
增加带宽、提供链路冗余（当一条成员链路故障时，流量自动切换到其他正常链路）、有时也能实现负载均衡（将流量分散到不同成员链路上）。

### 实现方式
LAG 可以通过两种主要方式创建：
 **（1）静态 LAG (Static LAG / Manual LAG)：** 
管理员在两端的交换机（或交换机与服务器）上**手动配置**哪些端口属于同一个聚合组。**不需要任何控制协议**。双方只是简单地按照配置把端口捆绑起来工作。这种方式简单，但缺乏链路状态检测和协商机制。

**（2）动态 LAG (Dynamic LAG)：** 
使用 `LACP` 协议来自动协商、建立和管理聚合组。这是更推荐的方式。

## LACP
LACP (Link Aggregation Control Protocol - 链路聚合控制协议) 是一个**协议标准**（由 IEEE 802.1AX / 802.3ad 定义）。它运行在参与聚合的**设备之间**（比如两台交换机之间，或者交换机与支持 LACP 的服务器网卡之间）。
  
### 核心功能
- **发现与协商：** 
设备通过发送 LACPDU (LACP Data Units) 报文来发现对端是否也支持 LACP，以及可以聚合的端口。

- **链路状态监控：** 
持续监控成员链路的物理状态（是否 up/down）。
	
- **一致性检查：** 
确保聚合两端端口的配置（如速度、双工模式、VLAN 成员资格、MTU 等）是兼容的。只有配置一致的端口才能成功加入活动聚合组。
	
- **动态管理：** 
自动确定哪些符合条件的端口可以成为**活动端口**（Active Ports，实际转发流量的端口），哪些可以作为**备用端口**（Standby Ports，在活动端口故障时接替）。这提供了更好的冗余能力。
	
- **防止环路和错误配置：** 
通过协商机制，避免单端配置导致的环路或黑洞问题。

### LAG 和 LACP的关系

|特性|LAG (链路聚合组)|LACP (链路聚合控制协议)|
|---|---|---|
|**本质**|**逻辑接口/概念**：捆绑后的结果。|**协议**：用于建立和管理 LAG 的规则和通信机制。|
|**全称**|Link Aggregation Group|Link Aggregation Control Protocol|
|**做什么**|定义了一个逻辑通道，由多个物理链路组成。|提供自动协商、链路监控、一致性检查和动态管理功能。|
|**如何工作**|本身不工作。是配置（静态）或协议（LACP）的结果。|通过在设备间交换 LACPDU 报文来工作。|
|**依赖性**|可以静态存在（无需协议），也可以通过 LACP 动态建立。|依赖于 LAG 的概念来应用其功能。没有 LAG，LACP 无用武之地。|
|**配置类型**|可以是静态配置或动态配置（使用 LACP）。|是启用动态 LAG 的配置选项。|
|**主要优势**|提供高带宽和冗余的链路。|提供自动化、错误检查、动态故障切换和更可靠的聚合。|

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

# bonding 内核模块修改

## 上线
### 机器不重启方式
```bash
2. 编译生成修复版本的 bonding.ko
3. chmod +x bonding.ko
4. 将bonding.ko 移入到 /lib/modules/`uname -r`/updates 目录下；
5. depmod -a 
6. check: 
modinfo bonding ;此时看到 bonding模块就会显示/lib/modules/`uname -r`/updates目录下的bonding模块；
之前的老模块依然在/lib/modules/`uname -r`/kernel/drivers/net/bonding/下；
7. 执行卸载，加载的脚本；
# cat replace_mod_bonding.sh
rmmod bonding
modprobe bonding
systemctl restart network.service

执行：nohup sh replace_mod_bonding.sh &
上诉命令需要一起执行，如果分开执行，rmmod bonding可能导致bond口消失，
服务器失联；

8. 潜在影响：
    a. 服务器断网几秒
    b. 其他依赖bond的模块的影响；
```

### 机器重启方式
```bash
1>同上
将编译生成的新修复的bonding.ko 放入到 /lib/modules/`uname -r`/updates 目录下，然后 depmod -a；检查无误后，服务器重启。
需要基于各个内核版本，编译对应的修复版本。

2>发布新版本内核，修复该问题，上新内核版本；
```

# 参考
```bash

```