# ethtool的统计
## 统计范例
```c
intel 82599 10G网卡的统计
# ethtool -S eth03 | grep -i -e miss -e out_of -e disca -e fifo -e buff
     rx_no_buffer_count: 0
     fdir_miss: 0
     rx_fifo_errors: 0
     rx_missed_errors: 0
     tx_fifo_errors: 0
     alloc_rx_buff_failed: 0

Mellanox Cx4-Lx 25G 网卡的统计
# ethtool -S eth03 | grep -i -e disc -e fifo -e out_of -e miss -e buffer
     rx_out_of_buffer: 0
     rx_steer_missed_packets: 24627
     rx_out_of_range_len_phy: 0
     rx_discards_phy: 0
     tx_discards_phy: 0
     rx_buffer_passed_thres_phy: 0
     rx_prio0_discards: 0
     rx_prio1_discards: 0
     rx_prio2_discards: 0
     rx_prio3_discards: 0
     rx_prio4_discards: 0
     rx_prio5_discards: 0
     rx_prio6_discards: 0
     rx_prio7_discards: 0

主要是：
   rx_out_of_buffer：
   rx_discards_phy：

Mellanox Cx4-Lx 25G 网卡的统计
# ethtool -S eth03 | grep -i -e pci -e pause -e control
     tx_mac_control_phy: 1266552
     rx_mac_control_phy: 0
     rx_pause_ctrl_phy: 0
     tx_pause_ctrl_phy: 1266552
     rx_pci_signal_integrity: 0
     tx_pci_signal_integrity: 3
     outbound_pci_stalled_rd: 0
     outbound_pci_stalled_wr: 0
     outbound_pci_stalled_rd_events: 0
     outbound_pci_stalled_wr_events: 0
     rx_global_pause: 0
     rx_global_pause_duration: 0
     tx_global_pause: 1266552
     tx_global_pause_duration: 555665182
     rx_global_pause_transition: 0
     tx_pause_storm_warning_events: 0
     tx_pause_storm_error_events: 0

主要是：
 tx_pause_ctrl_phy：
 outbound_pci_stalled_wr：
```

注：ethtool -S 的输出中，不同网卡，不同固件版本，驱动版本，甚至操作系统版本，其输出可能都不一样。有rx_dropped，rx_fifo_error, rx_missed_errors, rx_discards_phy 等等。

## 重要的统计说明
### 收包丢包统计
#### Mellanox 网卡的 rx_discards_phy
![](attachments/Pasted%20image%2020230809143104.png)

按照上面的理解：  
rte_eth_stats_get 指的是软件的统计；  
rte_eth_xstats_get 展示了物理硬件的一些统计计数（驱动通过寄存器读取），rx_discards_phy 的统计应该是网卡的硬件缓存满或者PCIe 总线阻塞导致无法把包发送给内核的内存中。



#### Mellanox 网卡的 

### pci相关
### pause frame相关
### 其他
## 统计分类
![](attachments/Pasted%20image%2020230809142948.png)

# dpdk程序中的统计
在开发DPDK应用的时候，我们可以通过rte_eth_stats_get函数获取网卡统计信息中的imissed计数来判断网卡是否出现丢包。
![](attachments/Pasted%20image%2020230809142615.png)
![](attachments/Pasted%20image%2020230809142619.png)
注：对于ixgbe 驱动，则DPDK中 rte_eth_stats_get 调用的是 ixgbe_dev_stats_get；

## imiss
### 定义
![](attachments/Pasted%20image%2020230809142649.png)
Q: imiss 是 网卡的接收队列满吗？还是 ring_buffer 满？  
A: 根据上面的定义，是被硬件丢弃，应该是网卡的接收队列满。不应该是 ring_buffer 满，ring_buffer 是内核使用的，应该是一个软件的概念。

![](attachments/Pasted%20image%2020230809142728.png)
SW： software 

Q: imiss 统计是否可以基于队列？  
A: imiss统计，是全局的。并且是从寄存器中获取到的，因此是硬件统计。

ixgbe_dev_stats_get 中的实现如下。
![](attachments/Pasted%20image%2020230809142835.png)


# ifconfig的统计

# 参考
```c
https://www.houzhibo.com/archives/1373
https://docs.kernel.org/networking/device_drivers/ethernet/mellanox/mlx5/counters.html
```