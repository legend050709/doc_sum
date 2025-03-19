```table-of-contents
```

# 背景
我在`PREROUTING`上做了一个`REDIRECT`端口改写，相应的服务处理请求后会返回应答。

按照我以往的认识，认为回包的流量应该先后经过`OUTPUT`和`POSTROUTING`，所以我利用`iptables -t nat -nvL`去查看`NAT`表在`OUTPUT`链和`POSTROUTING`链上的`packge`计数器，结果发现没有上涨，这让我陷入了沉思。


# 分析
经过谷歌后找到了完美的解释：[linux-netfilter-how-does-connection-tracking-track-connections-changed-by-nat](https://superuser.com/questions/1269859/linux-netfilter-how-does-connection-tracking-track-connections-changed-by-nat)。

实际上我是知道`OUTPUT`链会先过`conntrack`表恢复原始IP关系的，但是超出我理解的是`NAT`表压根就不会再执行。

上述URL中给出了解释：==`NAT`表只在连接状态是`NEW`的时候（也就是TCP的第一个握手包）才会执行计算，一旦改写关系存入了`conntrack`，那么这条连接后续的通讯就不会再过`POSTROUTING`和`OUTPUT`上面的`NAT`表了，而是直接换成了匹配`conntrack`来复原连接之前的改写状态==。

因此，如果我们想看到回包的`package`计数器增长，就应该==去看`OUTPUT`或者`POSTROUTING`上面的`filter`表计数==，一定会看到上涨。


# 整体流程

如果我们是服务端，那么SYN包到达的时候，在`PREROUTING`链的`NAT`表执行过之后（可能做`DNAT`或者`REDIRECT`），路由表将决定是`FORWARD`还是`INPUT`：

(1) 如果`INPUT`，那么`conntrack`记录就此生成（原理如下），当回包的时候会首先根据`conntrack`作地址复原，并且是不会经过`OUTPUT/POSTROUTING`链`NAT`表（但是会经过`filter`表）的。

(2) 如果`FORWARD`，那么`conntrack`记录不会立即生成，需要经过`POSTROUTING`之后才知道是否做了`SNAT/MASQUERADE`，此时才会生成`conntrack`记录（原理如下）。当收到上游回包的时候，不会过`PREROUTING`的`NAT`表，而是直接根据`conntrack`复原为原始IP地址，然后直接`FORWARD->POSTROUTING`（不会过`NAT`表）送回原始客户端。


# 原理
 
![](attachments/Pasted%20image%2020250211123309.png)

如上图所示，Netfilter 在四个 Hook 点对包进行跟踪：

(1)`PRE_ROUTING` 和 `LOCAL_OUT`：
调用 `nf_conntrack_in()` 开始连接跟踪，正常情况 下会创建一条新连接记录，然后将 `conntrack entry` 放到 `unconfirmed list`。 

```text
Q: 为什么是这两个 hook 点呢？

A: 因为它们都是新连接的第一个包最先达到的地方：
	`PRE_ROUTING` 是外部主动和本机建连时包最先到达的地方
	`LOCAL_OUT` 是本机主动和外部建连时包最先到达的地方
```

(2) `POST_ROUTING` 和 `LOCAL_IN`：
调用 `nf_conntrack_confirm()` 将 `nf_conntrack_in()` 创建的连接移到 `confirmed list`。 
```text
Q: 同样要问，为什么在这两个 hook 点呢？

A: 因为如果新连接的第一个包没有被丢弃，那这 是它们离开 netfilter 之前的最后 hook 点.
1> 外部主动和本机建连的包，如果在中间处理中没有被丢弃，`LOCAL_IN` 是其被送到应用（例如 nginx 服务）之前的最后 hook 点

2> 本机主动和外部建连的包，如果在中间处理中没有被丢弃，`POST_ROUTING` 是其离开主机时的最后 hook 点。
```

## nf_conntrack_in 和 nf_conntrack_confirm

`nf_conntrack_in()` 创建的新 `conntrack entry` 会插入到一个 未确认连接（ unconfirmed connection）列表。如果这个包之后没有被丢弃，那它在经过 `POST_ROUTING` 时会被 `nf_conntrack_confirm()` 方法处理。`nf_conntrack_confirm()` 完成之后，状态就变为了 `IPS_CONFIRMED`，并且连接记录从 未确认列表移到正常的列表。

之所以要将创建一个合法的新 entry 的过程分为创建（new）和确认（confirm）两个阶段 ，是因为包在经过 nf_conntrack_in() 之后，到达 nf_conntrack_confirm() 之前 ，可能会被内核丢弃。这样会导致系统残留大量的半连接状态记录，在性能和安全性上都 是很大问题。分为两步之后，可以加快半连接状态 conntrack entry 的 GC。



# 参考
```bash
# iptables nat和conntrack的特殊之处
https://luckymrwang.github.io/2022/04/24/iptables-nat-conntrack%E7%9A%84%E7%89%B9%E6%AE%8A%E4%B9%8B%E5%A4%84/

# 云计算底层技术-netfilter框架研究
https://opengers.github.io/openstack/openstack-base-netfilter-framework-overview/

# 连接跟踪（conntrack）：原理、应用及 Linux 内核实现
https://arthurchiao.art/blog/conntrack-design-and-implementation-zh/#2-netfilter-hook-%E6%9C%BA%E5%88%B6%E5%AE%9E%E7%8E%B0

```