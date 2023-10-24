```table-of-contents
```

# 通用使用
Ethtool is a standard Linux utility for controlling network drivers and hardware, particularly for wired Ethernet devices. It can be used to:
- Get identification and diagnostic information
- Get extended device statistics
- Control speed, duplex, auto-negotiation and flow control for Ethernet devices
- Control checksum offload and other hardware offload features
- Control DMA ring sizes and interrupt moderation
- Flash device firmware using a .mfa2 image

## ethtool -k/-K 
见下
## ethtool -x/-X
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


## ethtool -a/-A
查看流控统计：
```c
ethtool -S eth1 | grep control
```
![](attachments/Pasted%20image%2020231023141453.png)
rx_flow_control_xon 是在网卡的 RX Buffer 满或其他网卡内部的资源受限时，给交换机端口发送的开启流控的pause帧计数。
对应的，tx_flow_control_xoff 是在资源可用之后发送的关闭流控的pause帧计数。

查看网络流控配置：`ethtool -a eth1`
![](attachments/Pasted%20image%2020231023141349.png)

关闭网卡流控：
```c
ethtool -A ethx autoneg off  # 自协商关闭
ethtool -A ethx tx off  # 发送模块关闭
ethtool -A ethx rx off # 接收模块关闭
```


## ethtool -g/-G
见下
## ethtool -l/-L
见下
## ethtool -c/-C

## ethtool -i 
排查一下网卡phy芯片firmware是不是有bug，安装的版本是不是符合预期，查看 ethtool -i eth1:
![](attachments/Pasted%20image%2020231023141552.png)

比如：集群中存在多台设备时，个别设备存在问题时，而机器的其他的配置，内核版本，内核参数都一样时，这个时候可能是硬件的固件版本不一致导致的。
## 其他操作
- Mellanox网卡设置抓包
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
3. 使用的 ethtool 版本足够新，支持 `-x` 和 `-X` 参数来设置 indirection table

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