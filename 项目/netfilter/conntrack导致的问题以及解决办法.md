```table-of-contents
```
# 问题介绍
今天将一个业务的流量切到新部署的一台机器上。前几天已经灰度了一个业务到这台机器上，一直很稳定，所以准备切更多的流量。  
11点左右开始把业务的流量切到这台机器上，没多久业务反馈服务不可访问，紧急把流量切回到原服务，留下新机器备查。
# 分析
由于启用了`nf_conntrack模块`，业务短链接请求访问量大，由于conntrack采用默认的配置参数，短时间内导致conntrack的连接追踪表达到`65536*4=262144`默认的最大限制，新的连接无法建立，导致大量的丢包，业务因此无法正常访问。

针对于各种协议的各种连接状态，连接追踪表中会保留对应的记录一段时间。
![](attachments/Pasted%20image%2020240405174726.png)
## 内核日志查看
`/var/log/message`中错误信息如下：
```bash
Jul 30 11:50:01 dbl14192 systemd: Starting Session 486429 of user root.  
Jul 30 11:50:02 dbl14192 kernel: nf_conntrack: table full, dropping packet  
Jul 30 11:50:02 dbl14192 kernel: nf_conntrack: table full, dropping packet  
Jul 30 11:50:07 dbl14192 kernel: net_ratelimit: 3626 callbacks suppressed
```
## 连接跟踪日志查看
那么既然我们确定是连接跟踪表满了，那么检查一下连接跟踪的日志信息：
```bash
[root@dbl14195 redisservice_product]# tail -f /proc/net/nf_conntrack

ipv4 2 tcp 6 28 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=63518 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=63518 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 64 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=60390 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=60390 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 86 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8788 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=8788 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 111 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=57070 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=57070 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 37 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=12400 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=12400 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 9 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=43830 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=43830 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 40 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=33886 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=33886 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 30 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=41580 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=41580 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 42 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=51388 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=51388 [ASSURED] mark=0 zone=0 use=2

ipv4 2 tcp 6 117 TIME_WAIT src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=63766 dport=8063 src=xxx.xxx.xxx.xxx dst=xxx.xxx.xxx.xxx sport=8063 dport=63766 [ASSURED] mark=0 zone=0 use=2
```
不光是UDP的，还包括TCP的连接。

**连接跟踪配置查看**
查看nf_conntrack的有些参数配置：
```bash
[root@localhost product]# cat /proc/sys/net/netfilter/nf_conntrack_count

4489

[root@localhost product]# cat /proc/sys/net/netfilter/nf_conntrack_max

65536
```
默认情况下`nf_conntrack_max`是65536，可能的原因是短时间内有大量的短连接到了这台机器，超过了这个最大值，导致连接大量丢包，服务不可能用。
# 解决
## 修改参数
```bash
vim /etc/sysctl.conf
#加大 ip_conntrack_max 值
net.ipv4.ip_conntrack_max = 393216
net.ipv4.netfilter.ip_conntrack_max = 393216


#降低 ip_conntrack timeout时间
net.ipv4.netfilter.ip_conntrack_tcp_timeout_established = 300
net.ipv4.netfilter.ip_conntrack_tcp_timeout_time_wait = 120
net.ipv4.netfilter.ip_conntrack_tcp_timeout_close_wait = 60
net.ipv4.netfilter.ip_conntrack_tcp_timeout_fin_wait = 120

```
加完以上内容后，通过sysctl -p 使配置生效 。
不过该方法有两个缺点：一是重启iptables后，ip_conntrack_max值又会变成65535默认值，需要重新sysctl -p ；
另一个是该法治标不治本，在高并发时，很快又会悲剧重演。

## 尝试移除 conntrack 模块
查看conntrack模块的情况：
```bash
[root@localhost product]# lsmod|grep -i -e Module -e conntrack
Module                  Size  Used by
xt_conntrack           12760  1
nf_conntrack_ipv4      14862  2
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
nf_conntrack          105702  6 xt_CT,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_ipv4

```

注：lsmod最右边的输出为 对应的模块被引用的次数。一般只有值为0，模块才可以被卸载。
如上所示：
### 引用模块无法卸载问题
尝试移除这些模块:
```bash
[root@localhost product]# modprobe -r  nf_conntrack_netbios_ns nf_conntrack_ipv4 xt_conntrack
modprobe: FATAL: Module nf_conntrack_ipv4 is in use.

[root@localhost product]# lsmod|grep nf_conntrack_ipv4
nf_conntrack_ipv4      14862  2
nf_defrag_ipv4         12729  1 nf_conntrack_ipv4
nf_conntrack          105702  6 xt_CT,nf_nat,nf_nat_ipv4,xt_conntrack,nf_nat_masquerade_ipv4,nf_conntrack_ipv4

[root@localhost product]# rmmod nf_conntrack
rmmod: ERROR: Module nf_conntrack is in use by: xt_CT nf_nat nf_nat_ipv4 xt_conntrack nf_nat_masquerade_ipv4 nf_conntrack_ipv4

```

模块被其它模块所使用，而且其它模块又被另外的模块所使用，不容易删除，而且删很多的模块还是一个计较危险的操作。

## 禁用连接跟踪
在Linux系统中，nf_conntrack作为一个内核模块用于跟踪网络连接状态。如果需要关闭nf_conntrack，可以使用以下方法：
1. 通过sysctl禁用nf_conntrack模块
```bash
sudo sysctl -w net.netfilter.nf_conntrack_max=0
```
3. 编辑系统启动文件`/etc/modprobe.d/blacklist.conf`，在其中添加`blacklist nf_conntrack`模块
4. 通过修改iptables防火墙规则，直接禁用某些需要nf_conntrack支持的特性。

### 方法一：禁用 nf_conntrack 模块
编辑系统模块的启动文件：/etc/modprobe.d/blacklist.conf,  添加
```bash
blacklist nf_conntrack
```
则会禁用 nf_conntrack 模块。但是不会影响iptables的功能。

### 方法二：使用raw表的不跟踪动作
**连接状态跟踪可以供iptables其他模块使用，最常见的两个使用场景是 iptables 的 nat 模块以及 state 模块**。

通过修改iptables防火墙规则，直接禁用某些需要nf_conntrack支持的特性。
即：使用裸表(raw)的不跟踪动作。

#### raw表介绍
**基础介绍**
iptables有5个链:PREROUTING，INPUT，FORWARD，OUTPUT，POSTROUTING，4个表:filter，nat，mangle，raw 。
4个表的优先级由高到低的顺序为:raw–>mangle–>nat–>filter
举例来说:如果PRROUTING链上，即有mangle表，也有nat表，那么先由mangle处理，然后由nat表处理 。

**raw表介绍**
RAW表只使用在PREROUTING链和OUTPUT链上，如下所示：

![](attachments/Pasted%20image%2020240405163240.png)

因为RAW表优先级最高，从而可以对收到的数据包在连接跟踪前进行处理。一但用户使用了RAW表，在某个链上，RAW表处理完后，将跳过NAT表和 ip_conntrack处理，即不再做地址转换和数据包的链接跟踪处理了。


RAW表可以应用在那些不需要做nat的情况下，以提高性能。如大量访问的web服务器，可以让80端口不再让iptables做数据包的链接跟踪处理，以提高用户的访问速度 。


#### 禁用操作
执行下面的命令可以对所有的连接停止跟踪：
```bash
iptables -t raw -A PREROUTING -j NOTRACK

iptables -t raw -A OUTPUT -j NOTRACK
```

如果只想禁掉对特定端口的跟踪，可以使用下面的命令：
```bash
iptables -t raw -A PREROUTING -p tcp -m multiport --dports 80,3128 -j NOTRACK
iptables -t raw -A PREROUTING -p tcp -m multiport --sports 80,3128 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp -m multiport --dports 80,3128 -j NOTRACK
iptables -t raw -A OUTPUT -p tcp -m multiport --sports 80,3128 -j NOTRACK
```


# 参考
```c
# 连接跟踪模块导致的网络不可用
https://colobu.com/2019/07/30/network-issue-because-of-nf-conntrack/
```