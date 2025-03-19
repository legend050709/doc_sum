```table-of-contents
```
# RoCE的背景
## 为什么需要RoCE
## 为什么需要Soft-RoCE

# RoCEv2 协议 
## 简介
RoCEv2 协议是存在于 UDP/IPv4 或 UDP/IPv6 之上的 RDMA 传输协议。InfiniBand (IB) 基本传输标头 (BTH) 封装在 UDP 数据包中。


# Soft-RoCE原理
Soft-RoCE就是把本来应该卸载到硬件的封包和解析工作，又拿到软件来做。其本身是基于Linux内核的TCP/IP协议栈实现的，网卡本身并不感知收发的数据包是RoCE报文，其驱动程序按照IB规范中的报文格式将用户数据封装成IB传输层报文，然后把报文整体当做数据填入Socket Buffer当中，由网卡进行下一步收发包处理。


# 虚拟机里跑Soft-RoCE
有了Soft-RoCE，我们基于Verbs API编写的程序，就可以不依赖于硬件进行执行，也可以很方便的跑在虚拟机里，便于开发和测试RDMA程序。

测试的网络拓扑很简单，一台PC，以及其上运行的两台CentOS虚拟机都连接到一个虚拟子网上，两台虚拟机上将运行Soft-RoCE，然后我们在宿主机上通过Wireshark抓取数据包。

![](attachments/Pasted%20image%2020250315100657.png)

# rxe
## 架构
![](attachments/Pasted%20image%2020250316204555.png)

```bash
Architecture:

     +-----------------------------------------------------------+
     |                          Application                      |
     +-----------------------------------------------------------+
                            +-----------------------------------+
                            |             libibverbs            |
User                        +-----------------------------------+
                            +----------------+ +----------------+
                            | librxe         | | HW RoCE lib    |
                            +----------------+ +----------------+
+---------------------------------------------------------------+
     +--------------+                           +------------+
     | Sockets      |                           | RDMA ULP   |
     +--------------+                           +------------+
     +--------------+                  +---------------------+
     | TCP/IP       |                  | ib_core             |
     +--------------+                  +---------------------+
                             +------------+ +----------------+
Kernel                       | ib_rxe     | | HW RoCE driver |
                             +------------+ +----------------+
     +------------------------------------+
     | NIC driver                         |
     +------------------------------------+
```


## rdma_rxe
rdma_rxe 内核模块提供 RoCEv2 协议的软件实现。



# 使用Soft-RoCE

## 安装步骤
(1) 首先需要加载 Soft-RoCE 内核驱动模块（rdma_rxe.ko）
```bash
modprobe rdma_rxe
```

(2) 基于以太网接口创建 RoCE
```bash
[root@host2 ~]# rdma link add r1 type rxe netdev ens160 
[root@host2 ~]# rdma link
link r1/1 state ACTIVE physical_state LINK_UP netdev ens160
```
其中r1代表创建的RoCE设备名称，可以自定义；基于以太网接口ens160.

(3) 查看生成的RoCE设备信息
![](attachments/Pasted%20image%2020250315095345.png)

(4) 删除RoCE设备
```bash
[root@host2 ~]# rdma link del r1
[root@host2 ~]# ibstat
```

## 测试
### RDMA write 延时测试
#### server端
```bash
ib_write_lat -a
```
#### client端
```bash
ib_write_lat -a host1
```




# 参考
```bash
```