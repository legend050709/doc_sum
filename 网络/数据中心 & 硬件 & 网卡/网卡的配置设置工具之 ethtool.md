```table-of-contents
```
# 介绍 

ehttool 只是一个 Linux 中管理 NIC的用户态的管理工具，其代码并没有集成到Linux内核中。类似于 iptables/netfilter 中的的 iptables。内核中集成了各个网卡驱动的 ethtool的操作API 接口。

# 通用使用
Ethtool is a standard Linux utility for controlling network drivers and hardware, particularly for wired Ethernet devices. It can be used to:
- Get identification and diagnostic information
- Get extended device statistics
- Control speed, duplex, auto-negotiation and flow control for Ethernet devices
- Control checksum offload and other hardware offload features
- Control DMA ring sizes and interrupt moderation
- Flash device firmware using a .mfa2 image


==`ethtool` 参数 有个惯例，小写一般都是查询某个配置，对应的大写表示修改这个配置==。

参见：[# Linux 网络栈接收数据（RX）：配置调优（2022）](https://arthurchiao.art/blog/linux-net-stack-tuning-rx-zh/)


参见：[不同版本的 ethtool 镜像包 ](https://mirrors.edge.kernel.org/pub/software/network/ethtool/)

## ethtool -k/-K 打开关闭N元组过滤

使用 如下所示：

![](attachments/Pasted%20image%2020240625103908.png)

范例如下：

![](attachments/Pasted%20image%2020240620181951.png)

## ethtool -n/-N 设置 RSS hash

RSS (Receive Side Scaling)

```bash
# ethtool -N <ethX> rx-flow-hash <type> <option>

Where <type> is:
  tcp4    signifying TCP over IPv4
  udp4    signifying UDP over IPv4
  gtpc4   signifying GTP-C over IPv4
  gtpc4t  signifying GTP-C (include TEID) over IPv4
  gtpu4   signifying GTP-U over IPV4
  gtpu4e  signifying GTP-U and Extension Header over IPV4
  gtpu4u  signifying GTP-U PSC Uplink over IPV4
  gtpu4d  signifying GTP-U PSC Downlink over IPV4
  tcp6    signifying TCP over IPv6
  udp6    signifying UDP over IPv6
  gtpc6   signifying GTP-C over IPv6
  gtpc6t  signifying GTP-C (include TEID) over IPv6
  gtpu6   signifying GTP-U over IPV6
  gtpu6e  signifying GTP-U and Extension Header over IPV6
  gtpu6u  signifying GTP-U PSC Uplink over IPV6
  gtpu6d  signifying GTP-U PSC Downlink over IPV6
And <option> is one or more of:
  s     Hash on the IP source address of the Rx packet.
  d     Hash on the IP destination address of the Rx packet.
  f     Hash on bytes 0 and 1 of the Layer 4 header of the Rx packet.
  n     Hash on bytes 2 and 3 of the Layer 4 header of the Rx packet.
  e     Hash on GTP Packet on TEID (4bytes) of the Rx packet.
```



查看：

![](attachments/Pasted%20image%2020240625104013.png)

设置：

![](attachments/Pasted%20image%2020240625104106.png)

action 如下：

![](attachments/Pasted%20image%2020240625104611.png)


范例如下：
```bash
(1) 查看tcp和udp
# ethtool -n eth02 rx-flow-hash udp4
UDP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
L4 bytes 0 & 1 [TCP/UDP src port]
L4 bytes 2 & 3 [TCP/UDP dst port]

# ethtool -n eth02 rx-flow-hash tcp4
TCP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA
L4 bytes 0 & 1 [TCP/UDP src port]
L4 bytes 2 & 3 [TCP/UDP dst port]

（2）设置 tcp 和 udp

ethtool -N eth02 rx-flow-hash udp4 sdfn

```
![](attachments/Pasted%20image%2020240620181732.png)

![](attachments/Pasted%20image%2020240725174150.png)


### intel 网卡的 flow-type 的限制

（1）flow-type 相同的多个过滤规则，需要有相同类型的匹配条件。

![](attachments/Pasted%20image%2020240624112406.png)


### user-def 字段

参考：[#  Intel(R) Ethernet Controller 700 Series](https://www.kernel.org/doc/html/v4.20/networking/i40e.html)

参考：[intel E800 系列（ice驱动）网卡](https://docs.kernel.org/networking/device_drivers/ethernet/intel/ice.html)




![](attachments/Pasted%20image%2020240624110300.png)

#### flow-type l3/l4 设置 user-def 

![](attachments/Pasted%20image%2020240625164807.png)

```bash
# ethtool -U <ethX> flow-type tcp4 src-ip 192.168.10.1 dst-ip \
192.168.10.2 user-def 0x4FFFF action 2 [loc 1]

where the value of the user-def field contains the offset (4 bytes) and
the pattern (0xffff).
```

![](attachments/Pasted%20image%2020240625165209.png)
即：payload 偏移指定字节数后，然后看2个字节的数据是否是设置的数据（flexible data）？？


注：用户定义（user-defined）的灵活偏移也被视为输入集（input set）的一部分，不能为同一类型(flow-type )的多个滤波器单独编程。然而，灵活数据不是输入集的一部分，并且多个过滤器可以使用相同的偏移量但匹配不同的数据。


#### i40e 通过 user-def 设置 VF

![](attachments/Pasted%20image%2020240624120745.png)

![](attachments/Pasted%20image%2020240624120917.png)

```
如上所示：
(1) user-def 一共是 64bit；
如果高32bit是 0xffffffff，那么过滤器规则就是当做 L3 VEB filter, 即 非隧道包。
如果高32bit不是 0xffffffff，那么 过滤器规则就是当做 Cloud Filter，高32bit 携带 vni；

（2） user-def 的 低32位：
user-def 的 低32位 指定的是 VF，如果 低32位的值 >= max_vfs, 那么 代表的是PF;
action 字段指定具体的  queue。


(3) Cloud Filter：
dst 表示的是 外层；
src 表示的 内层；

```




 **范例**

![](attachments/Pasted%20image%2020240624121926.png)

![](attachments/Pasted%20image%2020240624121951.png)






### ethtool -x/-X 查看设置RSS表和key



设置：
![](attachments/Pasted%20image%2020240625115707.png)

```bash
(1) 查看 与设置

ethtool -x|--show-rxfh-indir|--show-rxfh devname

ethtool -X|--set-rxfh-indir|--rxfh devname [hkey xx:yy:zz:aa:bb:cc:...]  [ equal N | weight W0 W1 ... | default ] [hfunc FUNC] [context CTX | new] [delete]

```


![](attachments/Pasted%20image%2020240625115617.png)



## ethtool -u/-U 设置与查看N元祖过滤

```bash
# ethtool -U <ethX> flow-type <type> src-ip <ip> [m <ip_mask>] dst-ip <ip>
[m <ip_mask>] src-port <port> [m <port_mask>] dst-port <port> [m <port_mask>]
action <queue>

Where:
  <ethX> - the Ethernet device to program
  <type> - can be ip4, tcp4, udp4, sctp4, ip6, tcp6, udp6, sctp6
  <ip> - the IP address to match on
  <ip_mask> - the IPv4 address to mask on
            NOTE: These filters use inverted masks.
  <port> - the port number to match on
  <port_mask> - the 16-bit integer for masking
            NOTE: These filters use inverted masks.
  <queue> - the queue to direct traffic toward (-1 discards the
            matched traffic)
```


```bash
ethtool -U eth0 flow-type tcp4 src-ip 192.168.1.100 dst-port 80 action 0
```

### 删除N元祖过滤条件

![](attachments/Pasted%20image%2020240624111513.png)

```bash
基于 ID 进行删除；在 设置 n-tuple filter的时候，也可以通过 loc 来设置 规则的 id。

如下所示：

```

![](attachments/Pasted%20image%2020240624111923.png)

```bash
ethtool -u eth04  # 查看规则的id
ethtool -U eth04 delete 7679  #基于规则id进行删除
```


### 修改 GRO 配置

![](attachments/Pasted%20image%2020240620182244.png)


## ethtool -x/-X 调整队列权重

![](attachments/Pasted%20image%2020240620181518.png)

```c
       -x --show-rxfh-indir --show-rxfh
              Retrieves the receive flow hash indirection table and/or RSS hash key.

       -X --set-rxfh-indir --rxfh
              Configures the receive flow hash indirection table and/or RSS hash key.

           hkey   Sets  RSS  hash key of the specified network device. RSS hash key should be of device supported length.  Hash key format must be in xx:yy:zz:aa:bb:cc format meaning both the nibbles of a byte should be
                  mentioned even if a nibble is zero.

           hfunc  Sets RSS hash function of the specified network device.  List of RSS hash functions which kernel supports is shown as a part of the --show-rxfh command output.

           equal N
                  Sets the receive flow hash indirection table to spread flows evenly between the first N receive queues.

           weight W0 W1 ...
                  Sets the receive flow hash indirection table to spread flows between receive queues according to the given weights.  The sum of the weights must be non-zero and must not exceed the size of the indirec‐
                  tion table.

           default
                  Sets the receive flow hash indirection table to its default value.

           context CTX | new
                  Specifies an RSS context to act on; either new to allocate a new RSS context, or CTX, a value returned by a previous ... context new.

           delete Delete the specified RSS context.  May only be used in conjunction with context and a non-zero CTX value.
```
调整网卡RSS队列配置：
查看：`ethtool -x ethx`；  
调整：`ethtool -X ethx xxxx`；
``` 
ethtool -X ens4f0 start 8 equal 8  //均衡的分发到8-15号队列；
```

### context 设置：将指定规则的流量分发到多个队列

**背景**
当前的网卡可能支持创建多个RSS规则并存，实际使用时基于匹配的流量选择对应的规则。
当应用程序想要限制接收流量的队列集时，这非常有用，例如特定的目标端口或 IP 地址。下面的示例显示如何将 TCP 端口 22 的所有流量定向到队列 0 和 1。

![](attachments/Pasted%20image%2020240711110449.png)

```bash
(1) 创建
ethtool -X eth0 start 2 equeal 8 hfunc toeplitz context new

创建一个新的RSS context：起始的队列是2， 一共8个队列。
比如，返回的 context 为 1

(2) 查看 
ethtool -x eth0 context 1


（3）设置FDIR 匹配规则：
ethtool -N eth0 flow-type tcp6 dst-port 22 context 1

（4）删除 fdir规则和 RSS context 规则：
# ethtool -N eth0 delete 1023
# ethtool -X eth0 context 1 delete
```

#### ethtool 工具支持 RSS context

 ethtool 从 4.8.9 版本开始支持，参见：[CentOS 7 - Updates for x86_64: applications/system: ethtool](https://linuxsoft.cern.ch/cern/centos/7/updates/x86_64/repoview/ethtool.html); 但是很多网卡可能自身不支持 设置 rss context。

![](attachments/Pasted%20image%2020240711112253.png)


#### 网卡驱动支持 FDIR 导流到queue group

目前看 intel  e810系列的网卡，通过 ethtool 好像无法设置 RSS context，可能是 ice 驱动版本的限制 或者 是ice 驱动的限制 ??？ 

参考：[#  Intel® Ethernet 700 Series Configure RSS Queue Regions Tests](https://doc.dpdk.org/dts/test_plans/queue_region_test_plan.html)
![](attachments/Pasted%20image%2020240711121032.png)

如下所示，查看了最新版本的内核，Intel E810系列网卡的 ICE驱动程序是不支持 RSS context的。博通网卡的bnxt驱动也是不支持RSS context的，目前看内核也就是Marvel的mvpp2驱动和  Solarflare（2019年被赛灵思Xilinx 收购，Xilinx 又被AMD收购） 的 sfc 驱动支持 RSS context设置。

![](attachments/image%20(10).png)

![](attachments/Pasted%20image%2020240712121901.png)

#### DPDK的PMD中支持 FDIR 导流到queue group
但是 看 `Intel® Ethernet Controller E810 DDP—Comms DDP Package Utilization` 中介绍，通过 DPDK 的 rte_flow 是可以设置  queue的 group的「DPDK不在使用 Ice的驱动，而是使用 用户态的 PMD 驱动」。
如下所示：DPDK中 将具有特定DSCP值的IPv4包 重定向到多个 queue中，多个queue之间通过RSS负载均衡。

![](attachments/Pasted%20image%2020240711115200.png)

![](attachments/Pasted%20image%2020240711114914.png)

![](attachments/Pasted%20image%2020240711115413.png)


testpmd 中 FDIR 规则使用 queue group 如下：

![](attachments/Pasted%20image%2020240711120400.png)


## ethtool -a/-A 流控的设置与查看 
查看流控统计：
```c
ethtool -S eth1 | grep control
```
![](attachments/Pasted%20image%2020231023141453.png)
rx_flow_control_xon 是在网卡的 RX Buffer 满或其他网卡内部的资源受限时，给交换机端口发送的开启流控的pause帧计数。
对应的，tx_flow_control_xoff 是在资源可用之后发送的关闭流控的pause帧计数。

![](attachments/Pasted%20image%2020240625103643.png)



查看网络流控配置：`ethtool -a eth1`
![](attachments/Pasted%20image%2020231023141349.png)

关闭网卡流控：
```c
ethtool -A ethx autoneg off  # 自协商关闭
ethtool -A ethx tx off  # 发送模块关闭
ethtool -A ethx rx off # 接收模块关闭
```


## ethtool -g/-G 调整队列大小

![](attachments/Pasted%20image%2020240625103610.png)

![](attachments/Pasted%20image%2020240620181339.png)

## ethtool -l/-L 调整队列个数
使用：

![](attachments/Pasted%20image%2020240625104157.png)

范例：
![](attachments/Pasted%20image%2020240620181212.png)

`combined` 表示 接受队列和发送队列个数绑定在一起，所以两者的数量都是一样的。调整的时候，调整一个，另外一个也会调整，且数值一样。

如果你的设备和驱动支持分别设置 TX queue 和 RX queue 的数量，那你可以分别设置。
```bash
ethtool -L eth0 tx 8
```


**注意**
对于大部分驱动，调整以上设置会导致网卡先 down 再 up，经过这个网卡的连接会断掉  
。

## ethtool -c/-C 进行中断合并

![](attachments/Pasted%20image%2020240620182144.png)


**intel i40e驱动的中断速率限制**

![](attachments/Pasted%20image%2020240624115431.png)

## ethtool priv-flags的设置与查看

![](attachments/Pasted%20image%2020240625104228.png)

```bash
   ethtool --show-priv-flags devname
   ethtool --set-priv-flags devname flag on|off ...
```

## ethtool 的  FEC设置与查看
FEC(Forward Error Correction): 前向纠错。

使用：
![](attachments/Pasted%20image%2020240625104335.png)


范例：
```bash
   ethtool --show-fec devname
   ethtool --set-fec devname encoding auto|off|rs|baser|llrs [...]
```

## ethtool -i 
排查一下网卡phy芯片firmware是不是有bug，安装的版本是不是符合预期，查看 ethtool -i eth1:
![](attachments/Pasted%20image%2020231023141552.png)

比如：集群中存在多台设备时，个别设备存在问题时，而机器的其他的配置，内核版本，内核参数都一样时，这个时候可能是硬件的固件版本不一致导致的。
## 其他操作
### Mellanox网卡设置抓包
```c
ethtool --set-priv-flags enp130s0f0 sniffer on
```
 在要监听的以太网接口上设置`sniffer` 标志后，运行tcpdump捕获该接口上的bypass kernel 流量。
注意：使能Offloaded Traffic Sniffer会降低bypass kernel数据流的速度。

## 范例
### 查看 RX 队列数量
```c
# ethtool -l eth02
Channel parameters for eth02:
Pre-set maximums:
RX:		0
TX:		0
Other:		0
Combined:	63
Current hardware settings:
RX:		0
TX:		0
Other:		0
Combined:	63
```
注意：不是所有网卡驱动都支持这个操作。如果你的网卡不支持，会看到如下类似的错误：
```c
$ sudo ethtool -l eth0
Channel parameters for eth0:
Cannot get device channel parameters
: Operation not supported
```
这意味着驱动没有实现 ethtool 的 `get_channels` 方法。可能的原因包括：该网卡不支持调整 RX queue 数量，不支持 RSS/multiqueue，或者驱动没有更新来支持此功能。

### 调整队列个数
`ethtool -L` 可以修改 RX queue 数量。

注意：一些网卡和驱动只支持 combined queue，这种模式下，RX queue 和 TX queue 是一对一绑定的，上面的例子我们看到的就是这种。

设置 combined 类型网卡的收发队列为 8 个：
```c
sudo ethtool -L eth0 combined 8
```
如果你的网卡支持独立的 RX 和 TX 队列数量，那你可以只修改 RX queue 数量：
```c
sudo ethtool -L eth0 rx 8
```
注意：对于大部分驱动，修改以上配置会使网卡先 down 再 up，因此会造成丢包。请酌情使用。

### 调整Rx queue的大小
增加 RX queue 的大小可以在流量很大的时候缓解丢包问题，但是，只调整这个还不够，软件层面仍然可能会丢包，因此还需要其他的一些调优才能彻底的缓解或解决丢包问题。

`ethtool -g` 可以查看 queue 的大小。
```c
$ sudo ethtool -g eth0
Ring parameters for eth0:
Pre-set maximums:
RX:   4096
RX Mini:  0
RX Jumbo: 0
TX:   4096
Current hardware settings:
RX:   512
RX Mini:  0
RX Jumbo: 0
TX:   512
```

以上显式网卡支持最多 4096 个接收和发送 descriptor（描述符，简单理解，存放的是指向包的指针），但是现在只用到了 512 个。

用 `ethtool -G` 修改 queue 大小：
```c
sudo ethtool -G eth0 rx 4096
```
注意：对于大部分驱动，修改以上配置会使网卡先 down 再 up，因此会造成丢包。请酌情使用。

### 调整 RX queue 的权重（weight）
一些网卡支持给不同的 queue 设置不同的权重，以此分发不同数量的网卡包到不同的队列。
如果你的网卡支持以下功能，那你可以使用：

1. 网卡支持 flow indirection（flow 重定向）
2. 网卡驱动实现了 `get_rxfh_indir_size` 和 `get_rxfh_indir` 方法
3. 使用的 ethtool 版本足够新，支持 `-x` 和 `-X` 参数来设置 indirection table(间接表)

`ethtool -x` 检查 flow indirection 设置：
```c
# ethtool -x eth02
RX flow hash indirection table for eth02 with 63 RX ring(s):
    0:      0     1     2     3     4     5     6     7
    8:      8     9    10    11    12    13    14    15
   16:     16    17    18    19    20    21    22    23
   24:     24    25    26    27    28    29    30    31
   32:     32    33    34    35    36    37    38    39
   40:     40    41    42    43    44    45    46    47
   48:     48    49    50    51    52    53    54    55
   56:     56    57    58    59    60    61    62     0
   64:      1     2     3     4     5     6     7     8
   72:      9    10    11    12    13    14    15    16
   80:     17    18    19    20    21    22    23    24
   88:     25    26    27    28    29    30    31    32
   96:     33    34    35    36    37    38    39    40
  104:     41    42    43    44    45    46    47    48
  112:     49    50    51    52    53    54    55    56
  120:     57    58    59    60    61    62     0     1
RSS hash key:
c7:08:92:5d:e9:f4:00:64:67:69:05:49:df:52:83:39:74:74:b6:da:e8:cc:35:9f:d8:99:fa:09:98:69:0a:22:24:27:c9:1e:41:3e:fb:e6
RSS hash function:
    toeplitz: on
    xor: off
    crc32: off
```
第一列是哈希值索引，冒号后面的是每个哈希值对于的 queue，
例如，第一行分别是哈希值 0，1，2，3，4，5，6，7，对应的 packet 应该分别被放到 RX queue 0，1，2，3，4，5，6，7。


用 `ethtool -X` 设置自定义权重：
```c
sudo ethtool -X eth0 weight 6 2
```
以上命令分别给 rx queue 0 和 rx queue 1 不同的权重：6 和 2，因此 queue 0 接收到的数量更多。注意 queue 一般是和 CPU 绑定的，因此这也意味着相应的 CPU 也会花更多的时间片在收包上。

### 调整 RX 哈希字段 for network flows
可以用 ethtool 调整 RSS 计算哈希时所使用的字段。
例子：查看 UDP RX flow 哈希所使用的字段：
```c
$ sudo ethtool -n eth0 rx-flow-hash udp4
UDP over IPV4 flows use these fields for computing Hash flow key:
IP SA
IP DA

可以看到只用到了源 IP（SA：Source Address）和目的 IP。
```

加入源端口和目的端口：
```c
sudo ethtool -N eth0 rx-flow-hash udp4 sdfn
```
`sdfn` 的具体含义解释起来有点麻烦，请查看 ethtool 的帮助（man page）。


### n元组过滤ntuple filter/fdir

一些网卡支持 “ntuple filtering” 特性。该特性允许用户（通过 ethtool ）指定一些参数来在硬件上过滤收到的包，然后将其直接放到特定的 RX queue。例如，用户可以指定到特定目端口的 TCP 包放到 RX queue 1。

Intel 的网卡上这个特性叫 Intel Ethernet Flow Director，其他厂商可能也有他们的名字，这些都是出于市场宣传原因，底层原理是类似的。

这个特性适用的场景：最大化数据本地性（data locality），以增加 CPU 处理网络数据时的缓存命中率。例如，考虑运行在 80 口的 web 服务器：

1. webserver 进程运行在 80 口，并绑定到 CPU 2
2. 和某个 RX queue 关联的硬中断绑定到 CPU 2
3. 目的端口是 80 的 TCP 流量通过 ntuple filtering 绑定到 CPU 2
4. 接下来所有到 80 口的流量，从数据包进来到数据到达用户程序的整个过程，都由 CPU 2 处理
5. 仔细监控系统的缓存命中率、网络栈的延迟等信息，以验证以上配置是否生效

检查 ntuple filtering 特性是否打开：
```c
$ sudo ethtool -k eth0
Offload parameters for eth0:
...
ntuple-filters: off
receive-hashing: on

可以看到，上面的 ntuple 是关闭的。
```


打开 ntuple-filter:
```c
sudo ethtool -K eth0 ntuple on
```
打开 ntuple filtering 功能，并确认打开之后，可以用 `ethtool -u` 查看当前的 ntuple rules.
```c
$ sudo ethtool -u eth0
40 RX rings available
Total 0 rules

可以看到当前没有 rules。
```

我们来加一条：目的端口是 80 的放到 RX queue 2：
```c
sudo ethtool -U eth0 flow-type tcp4 dst-port 80 action 2
```

你也可以用 ntuple filtering 在硬件层面直接 drop 某些 flow 的包。当特定 IP 过来的流量太大时，这种功能可能会派上用场。更多关于 ntuple 的信息，参考 ethtool man page。
> 注：`ethtool -S <DEVICE>` 的输出统计里，Intel 的网卡有 `fdir_match` 和 `fdir_miss` 两项，是和 ntuple filtering 相关的。关于具体的、详细的统计计数，需要查看相应网卡的设备驱动和 data sheet。

### GRO 开启关闭
`-k` 查看 GRO 配置：
```c
$ ethtool -k eth0 | grep generic-receive-offload
generic-receive-offload: on
```

`-K` 修改 GRO 配置：
```c
sudo ethtool -K eth0 gro on
```
注意：对于大部分驱动，修改 GRO 配置会涉及先 down 再 up 这个网卡，因此这个网卡上的连接  都会中断。

# Mellanox网卡配置
## 背景
目前主流的网卡驱动都是以太网驱动，例如最常见的 Intel 系列：
- igb：老网卡，其中的 `i` 是 `intel`，`gb` 表示（每秒 1）`Gb`
- ixgbe：`x` 是罗马数字 10，所以 `xgb` 表示 `10Gb`，`e` 表示以太网
- i40e：`intel` 40Gbps 以太网
- ice: intel E810系列网卡

`mlx5_core` 这个驱动有点特殊，它支持以太网驱动，但由于历史原因，它的实现与普通以太网驱动有很大不同： Mellanox 是做高性能传输起家的（2019 年被 NVIDIA 收购），早期产品是 InfiniBand， 这是一个**==平行于以太网==**的二层传输和互联方案：
![](attachments/Pasted%20image%2020230810104045.png)
Infiniband 在高性能计算、RDMA 网络中应用广泛，但毕竟市场还是太小了，所以 后来 Mellanox 又对它的网卡添加了以太网支持。表现在驱动代码上，就是会看到它有一些 特定的术语、变量和函数命名、模块组织等等，读起来比 ixgbe 这样原生的以太网驱动要累一些。 这里列一些，方便后面看代码：

- WR：work request, work items that HW should perform
- WC: work completion, information about a completed WR
- WQ: work queue contains WRs, scheduled by HW, aka **==ring buffer==**
- SQ: sending queue
- SR: sending request
- RQ: receive queue
- RR: receive request
- QP: queue pair
- EQ: event queue, e.g. **==HW events==**

## Mellanox网卡的ethernet设置
参考：[MLNX_OFED Documentation v5.6-1.0.3.3](https://docs.nvidia.com/networking/spaces/viewspace.action?key=MLNXOFEDv561033)
![](attachments/Pasted%20image%2020230815104315.png)

![](attachments/Pasted%20image%2020230815122611.png)
![](attachments/Pasted%20image%2020230815122758.png)

### 抓包
```c
ethtool --show-priv-flags devname
ethtool --set-priv-flags devname flag on|off ...

--show-priv-flags
	Queries the specified network device for its private flags.  The names and meanings of private flags (if any) are defined by each network device driver.

--set-priv-flags
	Sets the device's private flags as specified.
	   flag on|off Sets the state of the named private flag.


范例：
# ethtool --set-priv-flags eth03 sniffer on
# ethtool --show-priv-flags eth03
Private flags for eth03:
rx_cqe_moder       : on
tx_cqe_moder       : off
rx_cqe_compress    : off
tx_cqe_compress    : off
rx_striding_rq     : off
rx_no_csum_complete: off
xdp_tx_mpwqe       : off
sniffer            : on
dropless_rq        : off
per_channel_stats  : on
tx_xdp_hw_checksum : off
```
![](attachments/Pasted%20image%2020230815105703.png)

### Dropless Receive Queue
```c
ethtool --set-priv-flags eth03 dropless_rq on
ethtool --show-priv-flags DEVNAME
```
![](attachments/Pasted%20image%2020230815142922.png)

## mlxconfig
```c
# which mlxconfig
/bin/mlxconfig

# rpm -qf /bin/mlxconfig
mft-4.14.0-105.x86_64

# rpm -ql mft-4.14.0-105 |grep -e '/usr/bin'
/usr/bin/dimax_init
/usr/bin/flint
/usr/bin/flint_ext
/usr/bin/fwtrace
/usr/bin/i2c
/usr/bin/itrace
/usr/bin/mcra
/usr/bin/mdevices_info
/usr/bin/mft_uninstall.sh
/usr/bin/mget_temp
/usr/bin/mget_temp_ext
/usr/bin/minit
/usr/bin/mlx_fpga
/usr/bin/mlx_fpga_ext
/usr/bin/mlxburn
/usr/bin/mlxcables
/usr/bin/mlxcables_ext
/usr/bin/mlxconfig
/usr/bin/mlxdump
/usr/bin/mlxdump_ext
/usr/bin/mlxfwmanager
/usr/bin/mlxfwreset
/usr/bin/mlxgearbox
/usr/bin/mlxi2c
/usr/bin/mlxlink
/usr/bin/mlxlink_ext
/usr/bin/mlxmcg
/usr/bin/mlxmdio
/usr/bin/mlxpci
/usr/bin/mlxphyburn
/usr/bin/mlxprivhost
/usr/bin/mlxreg
/usr/bin/mlxreg_ext
/usr/bin/mlxtrace
/usr/bin/mlxtrace_ext
/usr/bin/mlxuptime
/usr/bin/mlxvpd
/usr/bin/mremote
/usr/bin/mst
/usr/bin/mst_cable
/usr/bin/mst_ib_add
/usr/bin/mstdump
/usr/bin/mstop
/usr/bin/mtserver
/usr/bin/pckt_drop
/usr/bin/resourcedump
/usr/bin/wqdump
/usr/bin/wqdump_ext
```

### 配置 cqe compression
```c
# 配置
mlxconfig -y -d 0000:3b:00.1 set CQE_COMPRESSION=1

# 查看
mlxconfig -d 0000:3b:00.1 i
mlxconfig -d 0000:3b:00.1 q
```
## mlnx_tune工具

```c
mlnx_tune -d 3b:00.1 -q

```

# intel网卡配置
# 参考