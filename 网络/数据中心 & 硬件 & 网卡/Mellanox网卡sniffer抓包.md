# 使用方法
```c
# 查看是否支持，一般是Mellanox网卡才支持
# ethtool --show-priv-flags eth03
Private flags for eth03:
rx_cqe_moder       : on
tx_cqe_moder       : off
rx_cqe_compress    : off
tx_cqe_compress    : off
rx_striding_rq     : off
rx_no_csum_complete: off
xdp_tx_mpwqe       : off
sniffer            : off
dropless_rq        : off
per_channel_stats  : on
tx_xdp_hw_checksum : off

```

## 开启&关闭
```
# 开启
ethtool --set-priv-flags eth03 sniffer on; tcpdump -nni eth03 tcp and host x.x.x.x

# 关闭
ethtool --set-priv-flags eth03 sniffer off
```

# 支持的设备
目前看 就是Mellanox网卡天然支持流量分叉；其他系列的网卡，比如Intel，博通的网卡不是天然的流量分叉，需要配置SO-IOV或者 FDIR 规则来实现流量分叉。

# 注意
正常情况下，通过Mellanox 的 sniffer功能进行tcpdump抓包，可以抓取从该物理网卡发出，以及该物理网卡收到的数据包（并且不影响原始流量到达DPDK应用程序）。

但是如果该Mellanox物理网卡开启了 txq_inline 以及  txq_mpw_en 功能，则很可能是抓取不到发出的数据包的。
```c
Mellanox Cx4网卡的 txq_inline 参数的原理详解：

Mellanox Cx4网卡的txq_inline参数用于启用内联数据（Inline Data）功能。
当启用内联数据功能时，网卡会将一部分数据包的有效负载直接插入到TLP（Transaction Layer Packet）中，而无需额外的DMA操作。

具体来说，当网卡收到一个需要发送的数据包时，如果数据包的有效负载小于等于txq_inline参数设置的值，
网卡会将有效负载直接插入到TLP的Payload字段中，并通过PCIe总线发送给主机。
这样可以减少DMA操作和PCIe总线的负载，提高数据传输的效率。

在Mellanox Cx4网卡中，txq_inline参数与其他参数（如txq_mpw_en）结合使用，
可以进一步提高网络传输效率。例如，当txq_inline设置为64字节时，如果一个数据包的有效负载小于等于64字节，
那么网卡可以直接将有效负载插入到TLP的Payload字段中，并使用txq_mpw_en参数启用MPW功能将多个数据包合并成一个大的数据包。
这样可以减少DMA操作和PCIe总线的负载，并提高数据传输的效率。
```

```c
【我的理解】： txq_mpw_en 参数 和  txq_inline 参数 是配合使用的，txq_inline 参数  指什么多大的包是小包，而 txq_mpw_en 参数 则是使能了多个小包作为TLP的payload。
即单个PCIe事务中包含了多个包，这样相比每个包的发送都是一个TLP，可以减少TLP的数量。
因为每个TLP都有各种overhead。减少了TLP的数量，进而提升了PCIe的效率。
```
![](attachments/image%20(9).png)
参考: [dpdk 20.11 mlx5 doc](https://doc.dpdk.org/guides-20.11/nics/mlx5.html)
```c
如下所示：
dpvs的启动参数：
dpvs -- -l 1,2,3,4,5,6,7,8,9,10,22,23,24,25,26,27,28,29,30 -w 0000:3b:00.1,txq_inline=200,txq_mpw_en=1 -w 0000:3b:00.0,txq_inline=200,txq_mpw_en=1 --syslog local5


抓包：eth04 为 0000:3b:00.1 对应的物理口。
ethtool --set-priv-flags eth04 sniffer on; tcpdump -nni eth04  host 192.20.1.11

结果：
只能抓取到从eth04发出去的包，抓取不到其收到的包。
```

DPDK 官方 的 Mellanox Cx4 25G网卡的性能调优：
![](attachments/image%20(12).png)


> 注：**实际在测试过程中，发现使能了 txq_inline=200,txq_mpw_en=1，即使在client发送超过了800B的数据包，在DPVS上通过sniffer抓包，也是没抓取到物理网卡发出去的包的**。

# 影响
开启sniffer，对于网卡的收包性能会存在影响。经过测试DPVS使用sniffer网卡，大概存在40-50%的影响。比如 25G Mellanox网卡，未开启的情况下，256B的数据包的转发能力达到了 23Gbps。 开启了Sniffer之后，可能就只有10Gbps。
![](attachments/Pasted%20image%2020230911201214.png)

# 补充说明
## flow  bifurcation 流分叉介绍
![](attachments/Pasted%20image%2020230912101922.png)
![](attachments/Pasted%20image%2020230912102315.png)
![](attachments/Pasted%20image%2020230912102658.png)

## 总结
（1）流量分叉的优点：
![](attachments/Pasted%20image%2020230912102540.png)
- 更好的性能  
    流量分叉是硬件特性，不需要CPU的参与。可以提供更好的性能。
- 和 kni 对比  
    kni 的话，需要在DPDK中实现具体的代码来进行流量从DPDK应用到内核协议栈。流量分叉只需要通过软件给硬件配置对应的规则即可。


（2）实现流量分叉的方式
> - SR-IOV
> - Flow filter(rte_flow/ fdir)



# 参考
```c
https://docs.nvidia.com/networking/display/MLNXOFEDv451010/Offloaded+Traffic+Sniffer

https://blog.csdn.net/legend050709/article/details/124872468
```