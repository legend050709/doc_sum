```table-of-contents
```


# 介绍
`ip monitor` 命令可用于监视路由表（网络接口上的网络寻址）的更改或本地主机上 ARP 表的更改。此命令在调试与容器和网络相关的网络问题时特别有用，如当两个虚拟机应该能彼此通信，但实际不能。

在使用 `all` 时，`ip monitor` 会报告所有的更改，前缀以 `[LINK]`（网络接口更改）、`[ROUTE]`（更改路由表）、`[ADDR]`（IP 地址更改）或 `[NEIGH]`（邻居的 ARP 地址相关的变化）。

# 原理

![](attachments/Pasted%20image%2020240725112506.png)
`ip monitor all` 监控，如果没有事件发生，则会一直阻塞（阻塞在 recvmsg），事件发生了之后，就打印一条日志，接着阻塞等待下一个事件的发生。

![](attachments/Pasted%20image%2020240725112726.png)

# 使用


# 其他

## net-tools 包和 iproute2包的对比

![](attachments/Pasted%20image%2020240725105824.png)

## rtmon

## ip route monitor

## ip -s
另一个适用于许多命令的有用选项是 `ip -s`，它提供了一些统计信息。添加第二个 `-s` 选项可以添加更多统计信息。上面的 `ip -s link list wlp4s0` 会给出很多关于接收和发送的数据包的信息、丢弃的数据包数量、检测到的错误等等。

```bash
ip -s link list wlp4s0
```

# 参考
```bash

```