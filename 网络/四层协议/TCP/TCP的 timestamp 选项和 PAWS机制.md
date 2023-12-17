```table-of-contents
```
# TCP timestamp选项
## 介绍
Timestamp 是 TCP 协议包首部的可选项，包含 2 个值，分别是报文发送的时间 (TSval)，以及收到的对端发送来的 TSval 的回放（原样返回） (TSecr)。
```c
+-------+-------+---------------------+---------------------+
|Kind=8 |  10   |   TS Value (TSval)  |TS Echo Reply (TSecr)|
+-------+-------+---------------------+---------------------+
    1       1              4                     4
```


## 作用
照 RFC-7323 描述，TCP Timestamp 由两个重要的理由，然而这两个理由都没有说服我们：

1. PAWS，防止陈旧的报文干扰正常的连接。
> 在超高速网络中，假设 TCP接收窗口有 4G 那么大，因为序列号是 4 字节的，有一个序列号是 x的包丢失重传的时候，这个包可能被当作 x+4G 位置的包。现在是 2022 年，我不相信这样的事情真会发生。
2. 更精确地估算报文往返时间(round-trip-time, RTT)。

还有一个作用：TCP Cookie 可以利用 TSval 保存 cookie，减少哈希冲突。
## 虚拟时钟

## 设置
```c
sysctl -w net.ipv4.tcp_timestamps=0
```

## 影响
打开 TCP Timestamp，可能存在一些问题：
1. TSval 的值是递增的，这就有人靠这个推算服务器的启动时间。 [这个问题在 2016 年修复了](https://github.com/torvalds/linux/commit/95a22caee396cef0bb2ca8fafdd82966a49367bb)。
2. 打开了 Timestamp 之后每个包多了 12 字节（timestamp option占用10B，但是考虑到Tcp option必须是4的倍数，因此会有2B的padding ）。在 Linux 源码 `include/net/tcp.h` 可以看到 `TCPOLEN_TSTAMP_ALIGNED` 。 [有人实验过关掉 Timestamp 之后带宽增加了约 1%](http://highscalability.com/blog/2015/10/14/save-some-bandwidth-by-turning-off-tcp-timestamps.html)
3. [S3 遇到过关于 Timestamp 的诡异问题](https://www.snellman.net/blog/archive/2017-07-20-s3-mystery/)。
4. 开启tcp_timestamps和tcp_tw_recycle造成NAT转发连接不上问题。

# PAWS
## 介绍

## Per-host 的 PAWS机制
我们知道开启 `tcp_timestamp` 就会启动 Paws 机制。
同时开启 `tcp_timestamp` 和 `tcp_tw_recycle` 会启用TCP/IP协议栈的 per-host 的 PAWS 机制。
这种机制要求所有来个同一个  SIP 的TCP数据包的  timestamp 值是递增的。当收到一个 timestamp 值，小于服务端记录的对应值后，则会认为这是一个过期的数据包，然后会将其丢弃。

### 问题
`tcp_tw_recycle`机制是用于内核快速回收**TIME_WAIT**状态的套接字。但是当网络中存在NAT设备时，该机制反而可能会导致NAT设备背后的客户端难以连接上服务器。

比如：服务器收到的**SYN报文中携带的时间戳早于之前已经收到的FIN报文的时间戳**，于是服务器认为该SYN报文是由于网络阻塞迟到的旧连接的SYN报文的重传，于是拒绝回复SYN-ACK。
> 出现这种情况的原因是传输链路上存在NAT设备。而缓存FIN报文时间戳的`TCP Metrics`是`Per-host`的，在有NAT的环境中，服务器看到的`host`是NAT设备，它看不到NAT设备背后还有多大的内部网络，内部网路的每台主机上无法保证SYN报文的时间戳递增。


### 原因
经过同一个 SNAT 转换的数据包在 Server 端看来是和同一个 Client 打交道，虽然经过 `SNAT` 转换，但是不同 Client 所携带的 `timestamp` 是不一样的，无法保证整过 NAT 转化后的数据包携带的 `timestamp` 值严格递增，所以会出现丢包。

### 解决
在使用 SNAT 做转发的时候，关闭后端服务器的 `tcp_tw_recycle` 功能，保留 `tcp_timestamps`。

### 4系内核移除了`tcp_tw_recycle` 
参考：[# TCP Metrics--remove per-destination timestamp cache](https://switch-router.gitee.io/blog/tcp-metrics-remove-timestamp/)

4系内核移除了`tcp_tw_recycle`机制，也就是移除了 Per-host 的 PAWS机制。

## per-Connection 的  PAWS机制



# 参考
```c
https://ylgrgyq.github.io/2017/06/30/tcp-time-wait/

tcp_tw_recycle 和 tcp_tw_reuse 内核参数
http://just4coding.com/2017/11/09/timewait/


# TCP timestamp 选项那点事
https://switch-router.gitee.io/blog/tcp-timestamp/

# tcp_tw_recycle 机制 移除
https://switch-router.gitee.io/blog/tcp-metrics-remove-timestamp/

# TCP timestamp
https://perthcharles.github.io/2015/08/27/timestamp-intro/

# 我们为什么要关闭 TCP timestamp
https://quant67.com/post/linux/net-ts-disable/why.html

# PAWS 的替代方案
https://datatracker.ietf.org/doc/html/draft-nishida-tcpm-disabling-paws-00
```