```table-of-contents
```


# 介绍
 ip monitor 是 Linux iproute2 包中的一个命令，它用于实时监控网络接口的状态变化。这个命令可以用来监视路由表的变化、地址的增减、网络接口的状态变化等。


# 作用
`ip monitor` 命令可用于监视路由表（网络接口上的网络寻址）的更改或本地主机上 ARP 表的更改。此命令在调试与容器和网络相关的网络问题时特别有用，如当两个虚拟机应该能彼此通信，但实际不能。

在使用 `all` 时，`ip monitor` 会报告所有的更改，前缀以 `[LINK]`（网络接口更改）、`[ROUTE]`（更改路由表）、`[ADDR]`（IP 地址更改）或 `[NEIGH]`（邻居的 ARP 地址相关的变化）。

# 原理

![](attachments/Pasted%20image%2020240725112506.png)
`ip monitor all` 监控，如果没有事件发生，则会一直阻塞（阻塞在 recvmsg），事件发生了之后，就打印一条日志，接着阻塞等待下一个事件的发生。

![](attachments/Pasted%20image%2020240725112726.png)

# 使用
## 语法
```bash
ip monitor [options] [OBJECT]
```
（1）OPTIONS：用于指定监视的详细级别和过滤条件。
（2）OBJECTS：指定要监视的网络对象，如链路（link）、地址（address）、路由（route）等

### 对象类型
```bash
- all：监视所有对象的变化。

- route：监视路由表的变化。

- link：监视网络接口（如 eth0, wlan0 等）状态的变化。

- address：监视网络接口地址的变化。

- label：监视标签对象的变化。

- rule：监视路由规则的变化。

- netconf：监视网络配置的变化。

- mroute：监视多播路由表的变化。

- neigh：监视邻居表（ARP 表）的变化。
```


### 常用选项
```bash
-r, --raw：输出原始格式的数据。

-t, --timestamp：在每行输出前加上时间戳。

-h, --human-readable：以人类可读的方式显示输出。

-f, --file FILE：将输出重定向到文件 FILE 而不是标准输出。

-s, --stats：显示统计信息。

-d, --daemon：以后台进程的形式运行。

-q, --quiet：减少输出量，通常用于脚本中。
```

![](attachments/Pasted%20image%2020241227150637.png)

### 帮助
```bash
ip monitor help
```

## 范例
### 监视链路层变化
```bash
ip monitor link
```
这个命令会实时显示网络接口（如 eth0、wlan0 等）的状态变化，如接口启用、禁用、速度变化等。
实际操作如下：
![](attachments/Pasted%20image%2020241211152318.png)

### 监视特定网络接口的状态变化
```bash
  ip -ts monitor link dev eth0
```

### 监视路由表的变化
```bash
ip -ts monitor route
```
### 监视网络接口地址的变化
```bash
 ip -ts monitor address
```

### 监视所有的网络变化
```bash
 ip -ts monitor all
```

实际操作如下：

![](attachments/Pasted%20image%2020241211152558.png)

继续监视，出下如下图所示：

![](attachments/Pasted%20image%2020241211152634.png)


# 应用
在维护Linux服务器时，ip monitor命令非常有用，尤其是在网络配置发生变化时，可以实时监控并快速诊断问题。

例如，当网络接口因为物理原因down掉，或者有新的路由信息加入到路由表时，ip monitor能够立即显示这些变化。

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
# Linux网络命令：它用于实时监控网络接口的状态变化的命令 ip monitor详解
https://blog.csdn.net/weixin_70208651/article/details/143529984
```