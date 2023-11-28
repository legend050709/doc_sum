```table-of-contents
```
# 概述
TCP是一个复杂的协议，这种复杂来源于对报文传输的可靠性承诺。**对每条TCP连接来说**，除了有独立的状态机、定时器之外，还有拥塞控制相关的一些运行变量，比如**RTT**、**CWND**、**SSTHRESH**等。这些运行参数同样也是每连接(`Per-Connection`)的。

`Per-Connection`意味着每条连接的这些参数互不影响，这是理所应当的！但是，想想这个情景：A与B之间已经建立了一条稳定的TCP连接，此时若新建一条新的连接，它的参数该如何设置呢？显然，和原连接保持一致是个快速达到稳定的办法。这就好比一个人要去一个陌生的地方，却不知道该选择哪种交通工具，也不知道该预估多少时间，对他来说，汲取去过的人的经验总是一条捷径。

这就是Linux内核中`TCP Metrics`框架的作用，**它可以为后续的连接提供指导**。当主机之间需要频繁**建立**和**拆除**TCP连接时，它带来的好处更加明显。

`TCP Metrics`显然不能是`Per-Connection`的，而应该是`Per-Host`的。也就是说，`TCP Metrics`表项应该是基于`<源IP,目的IP>`二元组的。从一台主机的角度，到达另一个特定地址主机的网络链路状况应该是被两台主机之间的所有连接所**共享**的。



# TCP metrics
## destination metrics
rtt、ssthresh等变量，这些变量一般在TCP连接建立的时候有个初始值，然后随着TCP的数据交互逐渐调整到适应对应的网络状态的值。但是如果每次TCP建立连接都依靠默认初始值逐渐调整，那么可能需要一段时间才能调整到合适值，这显然会降低TCP性能，对于这种场景一种优化方案就是destination metrics。

RFC2140中描述，如果新建立的连接从已经关闭的连接缓存的状态信息中获取初始化信息，称呼为temporal sharing，如果新建立的连接从其他已建立的TCP连接获取初始化信息称为ensemble sharing。linux中实现的是temporal sharing。


## TCP metrics
destination metrics是指TCP根据用户预设的一些值或者之前TCP连接缓存的一些值来初始化相关的状态变量。也就是说destination metric其实可以分为两部分，一部分是用户预设的值，另外一部分则是之前TCP连接缓存缓存的值，后面这一部分也称为TCP metrics。

## TCP metrics 的 per-host特性
显然一个TCP连接的网络状态(如RTT时延、拥塞窗口cwnd)只与目的IP地址强相关，而与传输层的端口并无太大关系。TCP metric就是以IP地址来缓存的，每个IP地址对应一个缓存条目。

## destination metrics和 tcp metric包含的信息
destination metrics包含以下与TCP关系比较大的状态信息：
```c
mtu、 window、 rtt、 rttvar、 rto_min、 ssthresh、 cwnd、 initcwnd、 initrwnd、 quickack、 reordering、congctl、 advmss。
```

TCP metrics 包含的信息：
```c
 rtt、 rttvar、ssthresh、 cwnd、reordering
```
这些参数的详细解释请查阅 `man ip-route` 和 `man ip-tcp_metrics`。
需要注意的cwnd这个metric，这个值表示TCP连接拥塞窗口cwnd的上限，而不是拥塞窗口的初始值，metric中的cwnd改名为cwnd_clamp，显然更合适一些

- 设置destination metrics时候，加不加lock的关系
> 其中rtt、rttvar、ssthresh、cwnd、reordering这5个TCP metrics可以在设置的时候添加lock关键字，TCP连接在初始建立时候如果没有对应目标IP地址的TCP metric，则会根据设置值来初始化对应这条IP 地址的TCP metric，如果添加了lock关键字，那么随后这个TCP连接关闭的时候就不会更新对应的metric，如果没有添加lock字段，并且tcp_no_metrics_save参数为0，那么就会根据当前状态来更新TCP metric缓存。
```c
# ip route add local 127.0.0.2 dev lo congctl reno initcwnd 5  ssthresh lock 4
```

## TCP metrics的作用
一般来说，当TCP连接建立的时候，如果要初始化一个对应的状态变量。
首先会查询TCP metrics缓存中是否存在目标地址的metric，如果存在则根据metric信息来初始化连接的参数；如果不存在则会在TCP metrics缓存中创建对应这个IP地址的TCP metric。
创建的时候还会根据destination metrics的设置来初始化tcp metrics。TCP连接在关闭的时候也会尝试把最新的连接状态信息写入到TCP metrics缓存中。

- 使用ip route设置的destination metrics并不是立即生效的.
> 上面我们说了TCP连接建立的时候会先从TCP metrics缓存中初始化连接相关的状态信息，如果没有TCP metric缓存才会从ip route设置中读取参数配置来建立TCP metrics。也就是说destination metrics为TCP metrics提供了初始值，一旦缓存中的TCP metric有效，就不会ip route设置的destination metric来初始化TCP连接了。那么TCP metric什么时候会失效呢？如果读取TCP metric缓存的时候发现距离上次更新这条IP地址的metric时间超过一小时，那么这个TCP metric就无效了，就需要从destination metric来重新初始化TCP metric。

## 内核中的TCP Metrics框架
内核使用`tcp_metrics_block`表示一条`Metrics`表项，这些表项根据`<源IP,目的IP>`组织在`tcp_metrics_hash`冲突链表表中，记录的值保存在内部`tcpm_vals`数组
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

### tcp_metric结构的初始化
```c
tcp_rcv_synsent_state_process --> tcp_finish_connect --> tcp_init_metrics --> tcp_get_metrics --> tcpm_new(optional)

```
tcpm_new函数仅在第一次链接建立的时候调用，后续新建的tcp链接都可以使用曾经创建的tcp_metric，即说tcp_metric没有删除的机制（当然可以通过命令删除某个tcp_metric）。

### tcp_metric结构更新
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
其实想想也知道，tcp_metric主要关注的是拥塞控制相关的内容。更新的操作发生在链接关闭的时候，保存信息，便于下一次的参考使用。

切记，`TCP Metrics`表项是`Per-Host`的，因此，多条TCP连接的套接字可能会更新同一条表项。


# 其他
## ip tcp_metrics 命令
![](attachments/Pasted%20image%2020231128120252.png)
![](attachments/Pasted%20image%2020231128120501.png)
![](attachments/Pasted%20image%2020231128121840.png)


# 参考
```c
# 内核TCP Metrics框架
https://switch-router.gitee.io/blog/tcp-metrics/

# TCP系列53—拥塞控制—16、Destination Metrics和Congestion Manager
https://www.cnblogs.com/lshs/p/6038823.html

# ip-tcp_metrics(8) — Linux manual page
https://man7.org/linux/man-pages/man8/ip-tcp_metrics.8.html
```