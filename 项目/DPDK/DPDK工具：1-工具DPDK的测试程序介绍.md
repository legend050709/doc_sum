```table-of-contents
```
# 介绍

# dpdk-testpmd
## 介绍
[# Testpmd Runtime Functions](https://doc.dpdk.org/guides/testpmd_app_ug/testpmd_funcs.html)

[# Testpmd Application User Guide](https://doc.dpdk.org/guides-22.11/testpmd_app_ug/index.html)

## 参数
### 控制函数

![](attachments/Pasted%20image%2020240729125733.png)

### fwd函数

![](attachments/Pasted%20image%2020240729121532.png)

常用的参数：io、rx-only、macswap、5tswap



## 使用
### testpmd的参数和rte_eal的参数


### testpmd接管网卡如何打流到testpmd
**方法一: 测试设备和待测设备直连**

**方法二: 使用Mellanox网卡**
Mellanox网卡被DPDK接管，其在Linux中依然可见，可在上面配置ip，然后通过工具（比如：arping 或者 scapy ）发送 arp报文引流过来。

**方法三: testpmd中创建KNI 或 Vf ??**
不知道能否在testpmd中创建KNI，如果可以创建KNI，就可以通过arping 或者 scapy向外发送arp报文，引流过来。
如果可以创建Vf这样的虚拟口，然后通过vf口，进行arping 或者 scapy向外发送arp报文，引流过来？
``` 
比如：创建 VF；
然后DPDK接管的是VF，那么物理口依然在Linux内核中可见；
然后在物理口上进行arpping/scapy 向外发送arp报文 进行arp 欺骗。
```


**其他**
上面只能借助arping 或者 scapy在二层引流过来，如果本机的ip在三层中不可路由，还要借助bgp（比如bird）引流三层流量。


## 范例
### testpmd使用af-xdp
如下所示，af-xdp应用程序和xdp驱动程序同core的设置。
```bash
ethtool -L eth02 rx 14 tx 14
/usr/sbin/set_irq_affinity.sh eth02
for a in `seq 1227 1240`; do cat /proc/irq/${a}/smp_affinity_list; done
bb=1; for a in `seq 1227 1240`; do echo $bb > /proc/irq/${a}/smp_affinity_list; bb=$((bb+1)) ;done


此时相对于只使用了14core
echo 2 | sudo tee /sys/class/net/eth02/napi_defer_hard_irqs
echo 200000 | sudo tee /sys/class/net/eth02/gro_flush_timeout

// busy-budget 不设置，默认为64；force-copy=0；
// 14个队列，1-14号core，作为收包和af-xdp进行处理的core。
./dpdk-testpmd -l 1-15  --no-pci --main-lcore=15 --vdev=net_af_xdp,iface=eth02,start_queue=0,queue_count=14,force_copy=0,xdp_prog=/home/relay/liuchuanqi/xdp-prog/bpf-kernel.o -- -i --a --nb-cores=14 --rxq=14 --txq=14 --forward-mode=macswap

```


### testpmd接管网卡测试


# dpdk-procinfo
## 介绍

`dpdk-procinfo`是`DPDK`开发套件里面的一个工具，运行方式是DPDK的`secondary` 进程方式运行，它能够取回`port`的统计信息，重置`port`的统计信息，并且打印DPDK的内存信息等等。
它是原来`dump_cfg`功能的一个扩展

参考：[# dpdk-proc-info Application](https://doc.dpdk.org/guides-22.11/tools/proc_info.html)

![](attachments/Pasted%20image%2020250102102802.png)



## 使用限制

![](attachments/Pasted%20image%2020250102102925.png)

- dpdk-procinfo 应该与具有相同 DPDK 版本的主进程一起运行。
- 当以共享库模式运行 dpdk-procinfo 时，需要传递与主应用程序相同的 NIC PMD 库。 PMD 库参数中的任何不匹配都可能导致未定义的行为和结果也影响主要应用程序。
- PCAP 和 TAP 等虚拟设备不支持使用 dpdk-procinfo 检索统计信息。
- 由于 dpdk-procinfo 的默认 DPDK EAL 参数是 `-c1、-n4 和 --proc-type=secondary`，因此不希望用户传递任何 EAL 参数。

## 详细实现
### 接口信息打印

![](attachments/Pasted%20image%2020250102102234.png)

### 内存信息打印
`DPDK-22.11`中的信息打印。

![](attachments/Pasted%20image%2020250102101329.png)

![](attachments/Pasted%20image%2020250102101718.png)

### PMD相关查看

![](attachments/Pasted%20image%2020250102102325.png)


## 使用
```bash
./<build_dir>/app/dpdk-proc-info -- -m | [-p PORTMASK] [--stats | --xstats |
--stats-reset | --xstats-reset] [ --show-port | --show-tm | --show-crypto |
--show-ring[=name] | --show-mempool[=name] | --iter-mempool=name |
--show-port-private | --version | --firmware-version | --show-rss-reta |
--show-module-eeprom | --show-rx-descriptor queue_id:offset:num |
--show-tx-descriptor queue_id:offset:num ]
```


# dpdk-dumpcap  
参考专门的文章 `pdump` 和 `dumpcap` 工具。

# dpdk-pmdinfo


# 参考
```bash
# DPDK官方信息查看
https://blog.csdn.net/legend050709/article/details/121277438
```