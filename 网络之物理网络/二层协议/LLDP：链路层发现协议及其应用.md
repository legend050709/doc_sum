```table-of-contents
```
# 介绍
lldp(Link Layer Discovery Protocol)  链路层发现协议, 工作在 L2（数据链路层）。
用于：告诉网络邻居："我是谁，我连接到了哪里。"
它属于网络设备之间的"自我介绍"协议。


# lldp命令
实现交换机端口的抓取，目前我知道的有两种工具，一种是 lldpad，另一种是 lldpd。

## 介绍
lldpad 工具 通过 lldpad –d 启动守护进程，然后通过 lldptool 来进行查看。
lldpd 工具通过 lldpd 来启动守护进程，然后通过 lldpcli 来进行查看。
## 安装 & 启动
### 安装启动 lldpad
```c
yum install lldpad -y

启动守护进程
lldpad –d
```
![](attachments/Pasted%20image%2020231103115215.png)

### 安装启动 lldpd

```c
yum install lldpd -y

启动守护进程
lldpd
```
![](attachments/Pasted%20image%2020231103115147.png)


## 应用
### 查看接口的对端信息
#### 使用 lldptool 查看 lldpad 进程的交互信息
![](attachments/Pasted%20image%2020231103113840.png)
查看网卡的邻居信息 `lldptool -t -n -i eth01`
```c
# lldptool -t -n -i eth01
Chassis ID TLV
	MAC: 18:cf:24:3c:72:c1
Port ID TLV
	Ifname: 10GE1/0/15
Time to Live TLV
	120
System Name TLV
	HB.LF-RZ.M1.102-HW-CE6855-U1-A2-In-254.79-DPVS-Test
System Description TLV
	Huawei Versatile Routing Platform Software
VRP (R) software, Version 8.180 (CE6855HI V200R005C10SPC800)
Copyright (C) 2012-2018 Huawei Technologies Co., Ltd.
HUAWEI CE6855-48S6Q-HI

System Capabilities TLV
	System capabilities:  Bridge, Router
	Enabled capabilities: Bridge, Router
Management Address TLV
	IPv4: 10.44.2.79
	Ifindex: 110
	OID: 0.6.15.43.6.1.4.1.-113.91.5.25.41.1.2.1.1.1
Port VLAN ID TLV
	PVID: 100
Port and Protocol VLAN ID TLV
	PVID: 0, not supported, not enabled
VLAN Name TLV
	VID 100: Name VLAN100
MAC/PHY Configuration Status TLV
	Auto-negotiation not supported and not enabled
	PMD auto-negotiation capabilities: 0x0000
	MAU type: 10G BaseX
Link Aggregation TLV
	Aggregation capable
	Currently aggregated
	Aggregated Port ID: 15
Maximum Frame Size TLV
	9216
End of LLDPDU TLV
```

#### 使用lldpcli 查看lldpd的交互信息

![](attachments/Pasted%20image%2020231103115354.png)

`lldpcli show neighbors ports eth01： （可以简写为： lldpcli s n）` 查看接口的对端信息，如下所示：
![](attachments/Pasted%20image%2020231103115433.png)

如上所示：
SysName 表示的交换机的名称；
PortID：表示的是交换机的某个接口。
> 注：**可以通过上诉命令，查看2个物理机是否连接在同一个TOR交换机下**。

### 查看接口的本地信息
查看所有端口的本地信息。(即使未接线，也会显示)
`lldpcli show interfaces ports eth0 summary` 以及 `lldpcli show interfaces`
分别是查看本地的单个接口以及所有接口的信息。
![](attachments/Pasted%20image%2020231103115814.png)

### 查看本机信息
`lldpcli show chassis` 用于查看本机信息。
![](attachments/Pasted%20image%2020231103115910.png)
# 参考
```c
https://blog.csdn.net/legend050709/article/details/128097276

lldp工具的使用：
https://www.cnblogs.com/zhangxinglong/p/14163351.html


# LLDP链路层发现协议介绍
https://blog.csdn.net/legend050709/article/details/128097276
```