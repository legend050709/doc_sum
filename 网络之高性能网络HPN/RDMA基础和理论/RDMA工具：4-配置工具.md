```table-of-contents
```
# RoCE中有损和无损网络
在RoCE网络中分为有损和无损网络，如果用最朴实的语言描述：

**无损**：交换机和网卡上开启PFC和ECN，但很多新的交换机不支持PFC，并且PFC会影响性能。
**有损**：就是不开PFC。如果不开PFC，那就需要用其他的技术手段将可能的丢包减少到最低、将丢包的影响减低到最低。
因此引入了有损网络几大参数。

```bash
# mlxreg -d 5e:00.0 --reg_name ROCE_ACCL --get

Field Name                             | Data
====================================================
roce_adp_retrans_field_select          | 0x00000001
roce_tx_window_field_select            | 0x00000001
roce_slow_restart_field_select         | 0x00000001
roce_slow_restart_idle_field_select    | 0x00000001
roce_adp_retrans_en                    | 0x00000001
roce_tx_window_en                      | 0x00000000
roce_slow_restart_en                   | 0x00000001
roce_slow_restart_idle_en              | 0x00000000
====================================================
```

![](attachments/Pasted%20image%2020260304201249.png)

整体上看，对于ROCE lossy网络，transition windows 和adaptive timer效果比较明显。

# ACK和重传的知识点
## 在有损网络下，当接收端有一个包没收到的时候，会向接收端发什么

接收端向发送端发送两个包：oos_nack、cnp（ECN标记产生的，让发送端降速）。

## 接收端什么时候向发送端发ack
- 报文传完
- OOS（产生的原因是：收到两个psn，号不对）

## 关于重传机制
- ConnectX-4：从一个IB重传协议的丢失段重复传输，go-back-N。这可能会造成大量包重传
- ConnectX-5及以上。通过使用硬件重传来改善对丢包的反应
- ConnectX-6 Dx。使用一个专有的选择性重复协议

## 在有损网络中，什么时候发送端重传？
- 当发送端收到out-of-syquence（乱序）nack,重传。
-  最后一个报文丢包
-  Ack自己丢了
-  OOS nack丢包
    

后三种情况，需要依赖timer过期来重传。如果等timer过期，需要等很久，会增加延迟。adaptive timer原理是不用静态的timer过期，硬件自己来管。自己猜测，timer timeout自适应调小，来适应过期的时间。
 对于静态timer下，简单判断是：如果包重传的有点多，那就是timer out太小。如果丢包，那就是timer超时设置的太大。


# 参考
```bash

```