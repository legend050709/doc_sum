```table-of-contents
```
# 概述
TCP是一个复杂的协议，这种复杂来源于对报文传输的可靠性承诺。**对每条TCP连接来说**，除了有独立的状态机、定时器之外，还有拥塞控制相关的一些运行变量，比如**RTT**、**CWND(拥塞窗口)**、**SSTHRESH（慢启动阈值）**等。这些运行参数同样也是每连接(`Per-Connection`)的。

**`Per-Connection`意味着每条连接的这些参数互不影响，这是理所应当的**！
但是，想想这个情景：A与B之间已经建立了一条稳定的TCP连接，此时若新建一条新的连接，它的参数该如何设置呢？
显然，和原连接保持一致是个快速达到稳定的办法。这就好比一个人要去一个陌生的地方，却不知道该选择哪种交通工具，也不知道该预估多少时间，对他来说，汲取去过的人的经验总是一条捷径。

这就是Linux内核中`TCP Metrics`框架的作用，**它可以为后续的连接提供指导。
当主机之间需要频繁建立和拆除TCP连接时，它带来的好处更加明显**。

## 小结
因此，tcp_metric主要关注的是拥塞控制相关的内容。**==`TCP Metrics`显然不能是`Per-Connection`的，而应该是`Per-Host`的==**。

也就是说，`TCP Metrics`表项应该是**==基于`<源IP,目的IP>`二元组==**的。从一台主机的角度，到达另一个特定地址主机的网络链路状况应该是被两台主机之间的所有连接所**共享**的。


# TCP metrics
##  dest metrics
### 背景
rtt、ssthresh等变量，这些变量一般在TCP连接建立的时候有个初始值，然后随着TCP的数据交互逐渐调整到适应对应的网络状态的值。
但是**如果每次TCP建立连接都依靠默认初始值逐渐调整，那么可能需要一段时间才能调整到合适值，这显然会降低TCP性能，对于这种场景一种优化方案就是 dest metrics**。

RFC2140中描述，如果**新建立的连接从已经关闭的连接缓存的状态信息中获取初始化信息**，称呼为`temporal sharing`。
如果新建立的连接从其他已建立的TCP连接获取初始化信息称为`ensemble sharing`。

linux中实现的是temporal sharing。

## TCP metrics
`destination metrics`是指TCP根据用户预设的一些值或者之前TCP连接缓存的一些值来初始化相关的状态变量。

也就是说`destination metric`其实可以分为两部分：
一部分是用户预设的值；
另外一部分则是之前TCP连接缓存缓存的值。
后面这一部分也称为`TCP metrics`。

## TCP metrics 的 per-host特性

显然一个TCP连接的网络状态(如RTT时延、拥塞窗口cwnd)只与目的IP地址强相关，而与传输层的端口并无太大关系。TCP metric就是以IP地址来缓存的，每个IP地址对应一个缓存条目。

## dest metrics和 tcp metric包含的信息

### dest metrics包含以下与TCP关系比较大的 metric
dest metrics包含以下与TCP关系比较大的 metric：
```c
mtu、 window、 rtt、 rttvar、 rto_min、 ssthresh、 cwnd、 initcwnd、 initrwnd、 quickack、 reordering、congctl、 advmss。
```

### TCP metrics 包含的 metric

```c
 rtt、 rttvar、ssthresh(慢启动阈值)、 cwnd（拥塞窗口）、reordering
```

这些参数的详细解释请查阅 **`man ip-route` 和 `man ip-tcp_metrics`**。
需要注意的cwnd这个metric，这个值表示TCP连接拥塞窗口cwnd的上限，而不是拥塞窗口的初始值，metric中的cwnd改名为cwnd_clamp，显然更合适一些，另外就是这个metric的设置需要加lock关键字才生效。

### 设置destination metrics时候，加不加 lock 的关系

其中rtt、rttvar、ssthresh、cwnd、reordering这5个TCP metrics可以在设置的时候添加lock关键字；
TCP连接在初始建立时候如果没有对应目标IP地址的TCP metric，则会根据设置值来初始化对应这条IP 地址的TCP metric；
如果添加了lock关键字，那么随后这个TCP连接关闭的时候就不会更新对应的metric；
如果没有添加lock字段，并且tcp_no_metrics_save参数为0，那么就会根据当前状态来更新TCP metric缓存。
```c
# ip route add local 127.0.0.2 dev lo congctl reno initcwnd 5  ssthresh lock 4
```

## 清理tcp metric
```bash
# 清除 tcp_metric
sudo ip tcp_metrics flush all

# 关闭 tcp_metrics 功能
net.ipv4.tcp_no_metrics_save = 1
 
sudo ip tcp_metrics delete 100.118.58.7
```
## 查看 tcp metric
```bash
# ip tcp_metrics help
Usage: ip tcp_metrics/tcpmetrics { COMMAND | help }
       ip tcp_metrics { show | flush } SELECTOR
       ip tcp_metrics delete [ address ] ADDRESS
SELECTOR := [ [ address ] PREFIX ]
```

![](attachments/Pasted%20image%2020240423151423.png)

```bash
man ip-tcp_metrics
```

![](attachments/Pasted%20image%2020240510195005.png)


范例：
```bash
ip tcp_metrics show address 192.168.0.0/24
   Shows the entries for destinations from subnet

ip tcp_metrics show 192.168.0.0/24
   The same but address keyword is optional

ip tcp_metrics
   Show all is the default action

ip tcp_metrics delete 192.168.0.1
   Removes the entry for 192.168.0.1 from cache.

ip tcp_metrics flush 192.168.0.0/24
   Removes entries for destinations from subnet

ip tcp_metrics flush all
   Removes all entries from cache

ip -6 tcp_metrics flush all
   Removes all IPv6 entries from cache keeping the IPv4 entries.
```

## TCP metrics的作用
tcp_metrics会记录下之前已关闭TCP连接的状态，包括发送端CWND和ssthresh。
一般来说，当TCP连接建立的时候，如果要初始化一个对应的状态变量。
首先会查询TCP metrics缓存中是否存在目标地址的metric，如果存在则根据metric信息来初始化连接的参数；如果不存在则会在TCP metrics缓存中创建对应这个IP地址的TCP metric。
创建的时候还会根据destination metrics的设置来初始化tcp metrics。TCP连接在关闭的时候也会尝试把最新的连接状态信息写入到TCP metrics缓存中。


# 内核中的TCP Metrics框架
内核使用`tcp_metrics_block`表示一条`Metrics`表项，这些表项根据`<源IP,目的IP>`组织在`tcp_metrics_hash`冲突链表表中，记录的值保存在内部`tcpm_vals`数组。

```
struct tcp_metrics_block {
	struct tcp_metrics_block __rcu	*tcpm_next;
	struct inetpeer_addr		tcpm_saddr;
	struct inetpeer_addr		tcpm_daddr;
	......
	u32				tcpm_vals[TCP_METRIC_MAX_KERNEL + 1];
    ......
};
```

当新建TCP连接时，内核使用下面的接口来为TCP套接字设置`TCP Metrics`指导下的参数
```c
void tcp_init_metrics(struct sock *sk)
```

当某条TCP连接的运行参数发生变化时，比如重新计算RTT了，内核会使用下面的接口来更新它对应的`TCP Metrics`表项。
```c
void tcp_update_metrics(struct sock *sk)
```

切记，`TCP Metrics`表项是`Per-Host`的，因此，多条TCP连接的套接字可能会更新同一条表项。



## tcp_metric结构的初始化

```c
tcp_rcv_synsent_state_process --> tcp_finish_connect --> tcp_init_metrics --> tcp_get_metrics --> tcpm_new(optional)

```
tcpm_new函数仅在第一次链接建立的时候调用，后续新建的tcp链接都可以使用曾经创建的tcp_metric，即说tcp_metric没有删除的机制（当然可以通过命令删除某个tcp_metric）。

## tcp_metric结构更新

当某条TCP连接收的运行参数发生变化时，比如重新计算RTT了，内核会使用下面的接口来更新它对应的`TCP Metrics`表项。

```
void tcp_update_metrics(struct sock *sk)
```

在内核注释里头，tcp_update_metrics有以下这段话:
```text
/* Save metrics learned by this TCP session.  This function is called
 * only, when TCP finishes successfully i.e. when it enters TIME-WAIT
 * or goes from LAST-ACK to CLOSE.
 */

```

其实想想也知道：**tcp_metric主要关注的是拥塞控制相关的内容。更新的操作发生在链接关闭的时候，保存信息，便于下一次新建链接的参考使用**。

切记，**`TCP Metrics`表项是`Per-Host`的，因此，多条TCP连接的套接字可能会更新同一条表项**。

# tcp metric的问题

从系统cache中查看 tcp_metrics item

```bash
$sudo ip tcp_metrics show | grep  100.118.58.7
100.118.58.7 age 1457674.290sec tw_ts 3195267888/5752641sec ago rtt 1000us rttvar 1000us ssthresh 361 cwnd 40 metric_5 8710 metric_6 4258
```

**如果因为之前的网络状况等其它原因导致tcp_metrics缓存了一个非常小的ssthresh（这个值默应该非常大），ssthresh太小的话tcp的CWND指数增长阶段很快就结束，然后进入CWND+1的慢增加阶段导致整个速度感觉很慢**。

![](attachments/Pasted%20image%2020240511110916.png)

```bash
清除 tcp_metrics 
sudo ip tcp_metrics flush all 
 
关闭 tcp_metrics 功能
net.ipv4.tcp_no_metrics_save = 1
sudo ip tcp_metrics delete 100.118.58.7
```

tcp_metrics会记录下之前已关闭TCP连接的状态，包括发送端CWND和ssthresh，如果之前网络有一段时间比较差或者丢包比较严重，就会导致TCP的ssthresh降低到一个很低的值，这个值在连接结束后会被tcp_metrics cache 住，在新连接建立时，即使网络状况已经恢复，依然会继承 tcp_metrics 中cache 的一个很低的ssthresh 值，对于rt很高的网络环境，新连接经历短暂的“慢启动”后(ssthresh太小)，随即进入缓慢的拥塞控制阶段（rt太高，CWND增长太慢），导致连接速度很难在短时间内上去。而后面的连接，需要很特殊的场景之下(比如，传输一个很大的文件)才能将ssthresh 再次推到一个比较高的值更新掉之前的缓存值，因此很有很能在接下来的很长一段时间，连接的速度都会处于一个很低的水平。


## ssthresh 是如何降低的

在网络情况较差，并且出现连续dup ack情况下，ssthresh 会设置为 cwnd/2， cwnd 设置为当前值的一半，
如果网络持续比较差那么ssthresh 会持续降低到一个比较低的水平，并在此连接结束后被tcp_metrics 缓存下来。下次新建连接后会使用这些值，即使当前网络状况已经恢复，但是ssthresh 依然继承一个比较低的值。

## ssthresh 降低后为何长时间不恢复正常

ssthresh 降低之后需要在检测到有丢包的之后才会变动，因此就需要机缘巧合才会增长到一个比较大的值。
此时需要有一个持续时间比较长的请求，在长时间进行拥塞避免之后在cwnd 加到一个比较大的值，而到一个比较
大的值之后需要有因dup ack 检测出来的丢包行为将 ssthresh 设置为 cwnd/2, 当这个连接结束后，一个
较大的ssthresh 值会被缓存下来，供下次新建连接使用。

也就是如果ssthresh 降低之后，需要传一个非常大的文件，并且网络状况超级好一直不丢包，这样能让CWND一直慢慢稳定增长，一直到CWND达到带宽的限制后出现丢包，这个时候CWND和ssthresh降到CWND的一半那么新的比较大的ssthresh值就能被缓存下来了。


# 其他

##  使用ip route设置的destination metrics并不是立即生效的.
上面我们说了，TCP连接建立的时候会先从TCP metrics缓存中初始化连接相关的状态信息；

如果没有TCP metric缓存才会从ip route设置中读取参数配置来建立TCP metrics。

也就是说destination metrics为TCP metrics提供了初始值，一旦缓存中的TCP metric有效，就不会从ip route设置的destination metric来初始化TCP连接了。

## 那么TCP metric什么时候会失效呢？
如果读取TCP metric缓存的时候发现距离上次更新这条IP地址的metric时间超过一小时，那么这个TCP metric就无效了，就需要从destination metric来重新初始化TCP metric。


## 开启tcp_tw_recycle内核参数在NAT环境会丢包问题
### 现象
`tcp_tw_recycle`机制是用于内核快速回收**TIME_WAIT**状态的套接字。但是当网络中存在NAT设备时，该机制反而可能会导致NAT设备背后的客户端难以连接上服务器。

### 原因

导致这些问题的原因是服务器收到的**SYN报文中携带的时间戳早于之前已经收到的FIN报文的时间戳**，于是服务器认为该SYN报文是由于网络阻塞迟到的旧连接的SYN报文的重传，于是拒绝恢复SYN-ACK；
传输链路上存在NAT设备。而服务器上缓存FIN报文时间戳的`TCP Metrics`是 `Per-Destination`，  `Per-Host` 的，在有NAT的环境中，服务器看到的`Destination`是NAT设备，它看不到NAT设备背后还有多大的内部网络，内部网路的每台主机上无法保证SYN报文的时间戳递增。

### 解决
关闭 `tcp_tw_recycle`；
后面新版本内核 把此功能去掉了linux 4.12 `tcp: remove tcp_tw_recycle`。

# 参考
```c
# 内核TCP Metrics框架
https://switch-router.gitee.io/blog/tcp-metrics/

# TCP系列53—拥塞控制—16、Destination Metrics和Congestion Manager
https://www.cnblogs.com/lshs/p/6038823.html

# ip-tcp_metrics(8) — Linux manual page
https://man7.org/linux/man-pages/man8/ip-tcp_metrics.8.html
```