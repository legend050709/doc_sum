```table-of-contents
```

# 为什么进行IB数据包的分析
![](attachments/Pasted%20image%2020250721110214.png)


# RoCEv2的传输和抓包原理

## RoCEv2的传输

当RDMA数据包通过以太网到达并被RDMA网卡接收时，理论上，与RDMA传输相关的数据包处理（包括IP和UDP头部的处理）是直接在RDMA网卡（硬件）层面上完成的，而不需要操作系统内核的介入。

在RoCE使用场景中，RDMA网卡负责执行与RDMA传输相关的大部分工作，包括解析数据包、执行DMA操作将数据直接传输到目标内存等，绕过传统的CPU处理和操作系统内核，由RDMA网卡的硬件完成的。

RoCEv2是基于以太网的RDMA协议，它利用以太网作为传输介质。因此，数据包的基本格式是以太网帧。RoCEv2数据包在以太网帧内部使用UDP（用户数据报协议）作为传输层协议。UDP为数据包提供了端口号，使得源和目标主机能够正确识别和处理数据包。RoCEv2的UDP使用4791作为默认端口。
在RoCEv2数据包中数据载荷就是RDMA请求或响应:RDMA相关的头部信息，如操作类型、目标队列对（QP）号、序列号等。

![](attachments/Pasted%20image%2020250925112132.png)

![](attachments/Pasted%20image%2020250925112158.png)


### QA
#### RDMA网卡数据包处理（包括IP和UDP头部的处理）是直接在RDMA网卡（硬件）层面上完成的，当解析到载荷数据不是RoCEv2数据时,是不是就把数据包传递给内核的IP/TCP协议栈处理?

是的。

**RoCEv2数据传输**：
当网卡检测到传入的数据包是RDMA协议的一部分（例如，使用特定的UDP端口和协议标识符），它会按照RDMA的方式处理这些数据，利用硬件加速路径直接在网卡和应用程序之间传输数据，绕过传统的内核网络栈。这样可以提供极低的延迟和高带宽，适用于高性能计算和存储网络。

**非RoCEv2数据传输**：
当网卡检测到传入的数据包不属于RDMA操作（即，数据载荷不是RDMA数据时），如常规的IP数据包，这些包会被传递给操作系统的网络协议栈进行处理。


#### RoCE数据包处理（包括IP和UDP头部的处理）是直接在RDMA网卡（硬件）层面上完成的，而不需要操作系统内核的介入。是不是意味着,RDMA网卡的固件也需要实现IP/TCP协议栈？

确实，对于`RDMA over Converged Ethernet（RoCEv2）`这类情况，RDMA网卡的固件需要能够处理一定层面上的IP和UDP协议功能，尽管这不是一个完整的传统意义上的TCP/IP协议栈。

总的来说，RDMA网卡并不需要具备一个完整的TCP/IP协议栈，但它确实需要实现那些直接相关于其数据包协议（在RoCEv2的情况下是IP和UDP）的基础处理能力。

#### RoCE数据包处理（包括IP和UDP头部的处理）是直接在RDMA网卡（硬件）层面上完成的，而不需要操作系统内核的介入,那运行在linux上的tcpdump是如何抓取到RoCEv2数据包的?

说白了,就是tcpdump 直接读取网卡的缓存堆栈等抓取数据.

tcpdump是一个强大的网络数据包分析工具，它基于网卡抓取流动在网卡上的数据包。在Linux系统中，tcpdump通过直接访问网络接口（如RDMA网卡）来捕获数据包。这意味着，尽管RDMA网卡内部进行了复杂的数据处理，但数据包在通过网络接口发送或接收时，tcpdump仍然能够捕获到它们。

其次，当RDMA网卡通过DMA（直接内存访问）技术将数据直接传输到应用程序的内存时，这些数据包在传输过程中仍然会经过网络接口。这意味着，尽管DMA操作绕过了CPU和操作系统内核，但数据包本身仍然会在网络接口层面可见。

**在网络接口层级上监听数据流来捕获数据包。 怎么理解这句话?**

这句话指的是在计算机网络通信中，有一层专门负责处理数据包的传输和接收，即网络接口层。在这个层级上，网络接口会接收来自网络的数据流，并将其传递给操作系统的网络协议栈进行处理。同时，它也会将操作系统要发送的数据包传输到网络中。

当我们说在网络接口层级上监听数据流时，意味着某个软件或工具（比如 tcpdump）会在这个网络接口层级上监视网络通信的数据流。它可以实时地获取从网络中接收到的数据包，并且也能够捕获操作系统要发送的数据包。



# Mellanox 网卡抓包

有三种方法。(抓包：sniffer packet、Packet capture）

## 方法一：ibdump
### 背景
嗅探RDMA流量（抓包RDMA）非常棘手，因为一旦两端完成了初始握手，数据便会不经过内核协议栈，而是通过网卡（HCA）直接到达内存。
除了**在网络的中间节点上**放置专用硬件嗅探器来抓包，剩下的唯一方法就是**在网卡内放置有网卡商的`hook`接口，然后网卡商提供使用这些接口的 软件工具**。

例如：`Mellanox HCA`（网卡）的`ibdump`，`This tool is also a part of Mellanox OFED package`.




### ibdump包
```bash
# which ibdump
/bin/ibdump

# rpm -qf /bin/ibdump
ibdump-6.0.0-1.54310.x86_64

# rpm -ql ibdump-6.0.0-1.54310.x86_64
/usr/bin/ibdump
/usr/bin/vpi_tcpdump
```

![](attachments/Pasted%20image%2020250623152059.png)


### 特性

![](attachments/Pasted%20image%2020250721110055.png)

![](attachments/Pasted%20image%2020250721110115.png)

### 使用限制

![](attachments/Pasted%20image%2020250721110524.png)


### 高版本的 ibdump 

```bash
（1）下载ibdump源码：
https://github.com/Mellanox/ibdump#


（2）编译：
cd ibdump/
make clean; make -j30
如果编译没有问题，可以看到生成了 ibdump 二进制文件。
```

#### 下载高版本的 mft
`ibdump`的编译，可能依赖`mft`。

```bash
（1）下载编译依赖mft:
https://network.nvidia.com/products/adapter-software/firmware-tools/

比如：wget https://www.mellanox.com/downloads/MFT/mft-4.33.0-169-x86_64-rpm.tgz
tar xzf mft-4.33.0-169-x86_64-rpm.tgz
cd mft-4.33.0-169-x86_64-rpm
./install.sh

然后就可以看到：
$ ll /usr/include/mft/
total 36
drwxr-xr-x 2 root root 4096 Sep 24 16:52 cmdif
drwxr-xr-x 2 root root 4096 Sep 24 16:52 common
drwxr-xr-x 2 root root 4096 Sep 24 16:52 memaccess
drwxr-xr-x 2 root root 4096 Sep 24 16:52 mtcr
-rw-r--r-- 1 root root  800 Jul 25 00:52 mtcr_com_defs.h
-rw-r--r-- 1 root root  764 Jul 25 00:52 mtcr.h
-rw-r--r-- 1 root root  776 Jul 25 00:52 mtcr_mf.h
drwxr-xr-x 4 root root 4096 Sep 24 16:52 sdk
drwxr-xr-x 2 root root 4096 Sep 24 16:52 tools_layouts
```

##### mft的 release note

![](attachments/Pasted%20image%2020250924170557.png)




### 使用范例
```bash

```




### 问题
#### ibdump版本过低的问题
```bash
# ibdump -d mlx5_0 --mem-mode 100000000
Initiating resources ...
searching for IB devices in host
-E- Unsupported HW device id (212)
-E- failed to create resources

# ibdump 版本
# ibdump -v
ibdump, 3.0.0-7, built on Nov 15 2021, 11:06:35. GIT Version: NA

# 设备型号
# lspci -nn | grep Mellanox
b1:00.0 Ethernet controller [0200]: Mellanox Technologies MT28841 [15b3:101d]
b1:00.1 Ethernet controller [0200]: Mellanox Technologies MT28841 [15b3:101d]

```

#### ibdump无法使用的问题

![](attachments/Pasted%20image%2020250925172057.png)



## tcpdump方法
### 抓rdma包

对于低于5.1的`MLNX_OFED`版本，请运行：
```bash
＃tcpdump -i ens785f0 -s 65535 -w rdma_traffic.pcap
```

对于`MLNX_OFED v5.1`及更高版本，运行：
```bash
＃tcpdump -i mlx5_1 -s 65535 -w rdma_traffic.pcap

注：如果您使用的是MLNX_OFED v5.1或更高版本，请确保在您的设置中安装了libpcap库v1.9或更高版本。


查看tcpdump和libpcap版本
[root@rdma63 dscp]# tcpdump --help  
tcpdump version 4.9.2  
libpcap version 1.5.3
```

Ofed 版本查看：
```bash
# ofed_info -s
MLNX_OFED_LINUX-5.4-3.1.0.0:

# ofed_info -n
5.4-3.1.0.0
```

### 方法二：tcpdump-rdma（docker，Linux内核从4.9以上）

最新消息：Linux内核从4.9版开始就支持抓包RDMA(RoCE)流量。tcpdump发展到使用RDMA verbs接口直接 捕获流量。请确保使用最新的Linux内核service：[https://hub.docker.com/r/mellanox/tcpdump-rdma](https://hub.docker.com/r/mellanox/tcpdump-rdma "https://hub.docker.com/r/mellanox/tcpdump-rdma")


但是，在某些系统上很难升级`tcpdump`应用程序和关联的库以利用最新功能（特别是RDMA嗅探器）。
使用该`docker`容器是用户能使用`tcpdump`捕获和分析`RDMA`数据包的简单，优雅且最快的方式。

![](attachments/Pasted%20image%2020250924174524.png)




#### 使用方法
```bash
docker pull mellanox/tcpdump-rdma  
docker run -it -v /dev/infiniband:/dev/infiniband -v /tmp/traces:/tmp/traces --net=host --privileged mellanox/tcpdump-rdma
docker exec -it bold_payne  bash 
tcpdump -i mlx5_cx6_0 -s 0 -w /tmp/traces/capture1.pcap
```

### 方法三： tcpdump  （tcpdump，ConnectX®-4以上的版本，libpcap库v1.9或更高版本）
原文：[https://docs.mellanox.com/display/MLNXOFEDv451010/Offloaded+Traffic+Sniffer](https://docs.mellanox.com/display/MLNXOFEDv451010/Offloaded+Traffic+Sniffer "https://docs.mellanox.com/display/MLNXOFEDv451010/Offloaded+Traffic+Sniffer")

**在ConnectX®-4和更高版本的网卡中受支持**。

Offloaded 流量嗅探器 使得bypass kernel的数据传输方式 (如 RoCE, VMA, and DPDK)的流量可以被tcpdump等现有的抓包分析工具捕获。

使能Offloaded  Traffic Sniffer:
```bash
$ ethtool --set-priv-flags enp130s0f0 sniffer on

注意：如果您使用的是`MLNX_OFED v5.1`或更高版本，则此步骤无关紧要。
```

在要监听的以太网接口上设置`sniffer` 标志后，运行`tcpdump`捕获该接口上的`bypass kernel` 流量。

# 其他

## tcpdump抓包RDMA数据包

![](attachments/Pasted%20image%2020250925171445.png)

### 当前版本查看
```bash
查看 libpcap的版本：

# tcpdump --help
tcpdump version 4.99.5
libpcap version 1.10.4 (with TPACKET_V3)

```


```bash
查看tcpdump 包：
# which tcpdump
/sbin/tcpdump

# rpm -qf /sbin/tcpdump
tcpdump-4.9.0-5.el7.x86_64

# rpm -ql tcpdump
/usr/sbin/tcpdump
/usr/sbin/tcpslice
/usr/share/doc/tcpdump-4.9.0
/usr/share/doc/tcpdump-4.9.0/CHANGES
/usr/share/doc/tcpdump-4.9.0/CREDITS
/usr/share/doc/tcpdump-4.9.0/LICENSE
/usr/share/doc/tcpdump-4.9.0/README.md
/usr/share/man/man8/tcpdump.8.gz
/usr/share/man/man8/tcpslice.8.gz
```

### 源码编译安装高版本 libpcap
```bash
# 卸载tcpdump
yum remove -y tcpdump

# 升级 libpcap
wget https://www.tcpdump.org/release/libpcap-1.10.4.tar.gz
tar xzf libpcap-1.10.4.tar.gz
cd libpcap-1.10.4
./configure
make -j10
sudo make install
```

![](attachments/Pasted%20image%2020251107143515.png)

### 源码编译安装高版本 tcpdump:

```bash
# 升级后再编译 tcpdump
wget https://www.tcpdump.org/release/tcpdump-4.99.5.tar.gz
tar xzf tcpdump-4.99.5.tar.gz
cd tcpdump-4.99.5
./configure
make -j10
sudo make install
```

![](attachments/Pasted%20image%2020251107145321.png)


### 抓取RDMA流量
```bash
# ibdev2netdev
mlx5_bond_0 port 1 ==> bond0 (Up)

# ./tcpdump -nni mlx5_bond_0 -w 111.pcap
```
![](attachments/Pasted%20image%2020251107145855.png)

![](attachments/Pasted%20image%2020251107150453.png)


# 参考
```bash
# Analyzing InfiniBand Packets
https://openfabrics.org/images/eventpresos/workshops2015/UGWorkshop/Thursday/thursday_09.pdf

# 【网络】TCP抓包|RDMA抓包|ibdump、tcpdump用法说明
https://blog.csdn.net/bandaoyu/article/details/115791233

# RDMA测试杂谈
https://mp.weixin.qq.com/s/qDjOKj4ISMdZXlhN54vVkQ

# RoCEv2智能流量分析
https://support.huawei.com/enterprise/zh/doc/EDOC1100198802/5d4312fb


#【RDMA】LRH和GRH InfiniBand标头（LRH and GRH InfiniBand Headers）
https://blog.csdn.net/bandaoyu/article/details/117464053


# RDMA(4)协议栈：你追我赶，快快跑 【系列文章++++++】
https://mp.weixin.qq.com/s/0td4YuE8MBw5RtvOGOFylw  
```