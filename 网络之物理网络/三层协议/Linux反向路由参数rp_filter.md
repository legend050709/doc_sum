```table-of-contents
```
# 介绍
内核的 rp_filter 参数用于控制系统是否开启对数据包源地址的校验。
# 概念
所谓反向路由校验，就是在一个网卡收到数据包后，把源地址和目标地址对调后查找路由出口，从而得到反身后路由出口。然后根据反向路由出口进行过滤。

当rp_filter的值为1时，要求反向路由的出口必须与数据包的入口网卡是同一块，否则就会丢弃数据包。  
当rp_filter的值为2时，要求反向路由必须是可达的，如果反路由不可达，则会丢弃数据包。

# 说明
Linux [内核文档](https://www.baidu.com/link?url=2x1snmPs3FowXF4IgsntmrWh6TbFiojKKvF3lTRYe8K5oci44n83yIJvXRGJz-0V0rXxUhmqkaeWUBT7C_7AE7u-Vq2YlwBb-PkUhodVa5C&wd=&eqid=a1e7ed7e0000f7730000000359a423c3) 中的描述：
![](attachments/Pasted%20image%2020231106170937.png)
rp_filter 参数有三个值：

- 0：不开启源地址校验
- 1：开启严格的反向路径校验。对每个进来的数据包，校验其反向路径是否是最佳路径（即反向出口和入向口是否是同一个口）。如果反向路径不是最佳路径，则直接丢弃该数据包
- 2：开启松散的反向路径校验。对每个进来的数据包，校验其源地址是否可达，即反向路径是否能通（通过任意网口），如果反向路径不通，则直接丢弃该数据包

`/etc/sysctl.conf` 中包含 all 和 eth/lo（具体网卡）的 rp_filter 参数，取其中较大的值生效。

# 作用
- 减少DDoS攻击
启用了 rp_filter 之后可以减少 DDoS 攻击：校验数据包的反向路径，如果反向路径不合适，则直接丢弃数据包，避免过多的无效连接消耗系统资源。

- 防止IP Spoofing
还可以防止 IP Spoofing：校验数据包的反向路径，如果客户端伪造的源 IP 地址对应的反向路径不在路由表中，或者反向路径不是最佳路径，则直接丢弃数据包，不会向伪造 IP 的客户端回复响应。

# 配置
## 临时生效的配置方式
### 使用 sysctl 指令配置
sysctl 命令的 -w 参数可以实时修改Linux的内核参数，并生效。所以使用如下命令可以修改Linux内核参数中的rp_filter 。
```cpp
sysctl -w net.ipv4.conf.default.rp_filter =1
sysctl -w net.ipv4.conf.all.rp_filter =1
sysctl -w net.ipv4.conf.lo.rp_filter =1
sysctl -w net.ipv4.conf.eth0.rp_filter =1
sysctl -w net.ipv4.conf.eth1.rp_filter =1
……
```
### 修改内核参数的映射文件
在Linux文件系统映射出的内核参数配置文件中记录了Linux系统中网络接口 反向路由校验 配置参数 rp_filter 的值。可使用vi编辑器修改文件的内容，也可以使用如下指令修改文件内容：
```ruby
echo 1 > /proc/sys/net/ipv4/conf/default/rp_filter 
echo 1 > /proc/sys/net/ipv4/conf/all/rp_filter 
echo 1 > /proc/sys/net/ipv4/conf/lo/rp_filter 
echo 1 > /proc/sys/net/ipv4/conf/eth0/rp_filter 
echo 1 > /proc/sys/net/ipv4/conf/eth1/rp_filter 
……
```

## 永久生效的配置方式
永久生效的配置方式，在系统重启、或对系统的网络服务进行重启后还会一直保持生效状态。这种方式可用于生产环境的部署搭建。

修改/etc/sysctl.conf 配置文件可以达到永久生效的目的。

在sysctl.conf配置文件中有一项名为可以添加如下面代码段中的配置项，用于配置Linux内核中的各网络接口的 rp_filter 参数。
```cpp
net.ipv4.conf.default.rp_filter =1
net.ipv4.conf.all.rp_filter =1
net.ipv4.conf.lo.rp_filter =1
net.ipv4.conf.eth0.rp_filter =1
net.ipv4.conf.eth1.rp_filter =1
……
```
需要注意的是，修改sysctl.conf文件后需要执行指令sysctl -p 后新的配置才会生效。

# 区分
## ip_nonlocal_bind
ip_nonlocal_bind： 是否运行服务绑定一个本机不存在的IP地址。
### 配置说明

![](attachments/Pasted%20image%2020240425111206.png)

0：默认值，表示不允许服务绑定一个本机不存在的地址。
1：表示运行服务绑定一个本机不存在的地址。


### 使用场景
有些服务需要依赖一个VIP才可以启动，但是可能正常情况下，此VIP并不在本机上，当VIP漂移到本机上时才存在；但是服务又需要提前启动。例如，haproxy、nginx 等代理需要绑定VIP时。

#### arp欺骗

比如，arping 使用一个本机不存在的ip 进行arp欺骗，将二层流量引流到本机。
```bash
echo 1 > /proc/sys/net/ipv4/ip_nonlocal_bind

arping -s 192.21.8.133 192.21.8.133 -I eth03 -A

如上所示，使用本机不存在的IP发送免费ARP。

-s: 指定sip
-I： 设置发送接口
-A: 发送的是arp响应包

```

## ip_forward 和 forwarding
linux服务器经常被用来提供防火墙、路由器、NAT、负载均衡等功能。
在这些场景下，linux内核需要将网卡上收到的报文转发给其他网络设备。linux内核提供了ip_forward参数用于开关内核的报文转发功能，只有这个开关被打开时，内核才会执行报文的转发。


### 配置
ip_forward功能的配置开关有三个位置：
```bash
/proc/sys/net/ipv4/ip_forward
/proc/sys/net/ipv4/conf/{all/default/devname}/forwarding
/proc/sys/net/ipv6/conf/{all/default/devname}/forwarding
```

范例如下所示：
```bash
net.ipv4.ip_forward = 1


net.ipv4.conf.all.forwarding = 1
net.ipv4.conf.eth01.forwarding = 1
```

#### ip_forward

![](attachments/Pasted%20image%2020240425112546.png)

`ip_forward` 是许多文章中提到的ip报文转发开关，这个配置开关是linux早期版本中定义的，它**只能控制IPv4报文的转发功能，其功能和取值都等价于`/proc/sys/net/ipv4/conf/all/forwarding`**。

此外，实际上真正决定报文能否被转发的，是`conf/devname/forwarding`，这个配置在每个网卡设备的粒度控制这个网卡上收到的**ipv4/ipv6**报文能否被转发。

即：**ip_forward等价于ipv4/conf/all/forwarding(即设置 ip_forward 等效于设置 ipv4/conf/all/forwarding），而真正有效的是conf/devname/forwarding**。

设置了 `net.ipv4.ip_forward = 1`, 那么 就会存在 
```bash
net.ipv4.conf.all.forwarding =1 
各个接口的forwarding = 1, 即 net.ipv4.conf.ethX.forwarding = 1
```

设置了 `net.ipv4.ip_forward = 0`, 那么 就会存在
```bash
net.ipv4.conf.all.forwarding = 0 
各个接口的forwarding = 0, 即 net.ipv4.conf.ethX.forwarding = 0
```

可以通过设置  `net.ipv4.ip_forward = 0`，然后在某个具体的接口设置 forwarding =1;
```
即：
net.ipv4.ip_forward = 0
net.ipv4.conf.eth01.forwarding = 1
```

注：设置 ip_forward 等效于设置 ipv4/conf/all/forwarding，设置 `net.ipv4.ip_forward = 1`，那么 `net.ipv4.conf.all.forwarding = 1` ;  
同理，设置  `net.ipv4.conf.all.forwarding = 1`，那么  `net.ipv4.ip_forward = 1`；

#### forwarding
**/proc/sys/net/ipv4/conf/all/forwarding的说明**
以及
**/proc/sys/net/ipv4/conf/ethx/forwarding的说明**

![](attachments/Pasted%20image%2020240425112852.png)

**/proc/sys/net/ipv6/conf/all/forwarding的说明**

![](attachments/Pasted%20image%2020240428103751.png)

**ipv6/conf/ethx/forwarding的说明**

![](attachments/Pasted%20image%2020240428103256.png)

**注意**
forwarding 的值 会影响 其他参数（如 accept_ra、accept_redirects ）的行为。


**conf/default/forwarding又是用来干嘛的**？
首先，`conf/default/forwarding` 用于控制新建设备的配置，如果在配置这个参数后创建一个新的网络设备（例如`veth pair`），那么新创建设备的`conf/devname/forwarding`值就会等于`default`配置。

**conf/all/forwarding作用**？
`conf/all/forwarding`则可以配置当前所有设备的`forwarding`参数，例如将all参数配置从0修改为1，则包括default在内的所有forwarding配置都将被改成1。


### forwarding 原理

接口的 forwarding参数是配置使能IP层的转发，那应该在Linux内核的转发部分对该参数进行了判断，该参数的判断实际上是在查找路由时进行判断的，下面这张图显示了其中的调用关系；

![](attachments/Pasted%20image%2020240428104658.png)

在函数`ip_route_input_slow`中；
```c
if (!IN_DEV_FORWARD(in_dev))
        goto e_hostunreach;

```

看一下该宏是如何进行定义的：
```c
#define IN_DEV_FORWARD(in_dev)                 IN_DEV_CONF_GET((in_dev), FORWARDING)

#define IN_DEV_CONF_GET(in_dev, attr) \ ipv4_devconf_get((in_dev), NET_IPV4_CONF_ ## attr)//这里的##表示连接两个字符串。
```



# 其他
## 不存在 ipv6 的 rp_filter 系统配置

![](attachments/Pasted%20image%2020240429143003.png)


## 接收的TCP报文的DIP不等于接收口的IP如何处理
### 问题

Linux 设备上，收到的TCP报文的目的ip不是本机接收端口的ip，而是本机某个虚拟口的IP，是否丢弃该报文？还是再转发给正确的接口进行处理？

### 理解

在Linux设备上，**如果收到的TCP报文的目的IP地址不是接收端口的IP，而是本机某个虚拟接口的IP，那么入接口会变为虚接口，然后内核会根据路由表来决定如何处理**。

```bash
一般虚拟接口上的IP，其对应的主机路由为：
	ip route show table local
```

比如：IPIP隧道报文，收到该报文之后，基于DIP、SIP、Proto可以查找到 IPIP 隧道规则，匹配到该规则的报文为隧道报文，需要进行解隧道，然后**入接口就变成了虚拟的隧道口（这也是为什么不论隧道口是否配置VIP都可以在隧道口抓到入向的流量包）**，然后基于DIP（比如：LB的 IPIP tun模式下解隧道之后DIP为VIP），查找路由，如果隧道口上配置了该VIP，那么就继续走Local_In 上送给本机。
如果隧道口没有配置VIP，隧道口上开启了 forwarding 功能，那么就基于路由查询，转发解封装之后的流量出去。


```bash
# 查看当前设置
sysctl net.ipv4.conf.all.forwarding
sysctl net.ipv4.conf.<interface>.forwarding

# 修改设置（临时）
sysctl -w net.ipv4.conf.all.forwarding=1
sysctl -w net.ipv4.conf.<interface>.forwarding=1

# 修改设置（永久）
echo "net.ipv4.conf.all.forwarding = 1" >> /etc/sysctl.conf
echo "net.ipv4.conf.<interface>.forwarding = 1" >> /etc/sysctl.conf
sysctl -p
```

### 小结
其实对于三层而言，如果 rp_filter 设置为 2 或 0(即不关注反向路由查询) .

那么收到包之后，其实不关心DIP 是否是入接口的IP，也有可能DIP是其他的接口的IP。
比如：
```
ipip隧道模式，在RS上配置了多个隧道。
隧道1 以及 隧道口1 + VIP1
隧道2 以及 隧道口2 + VIP2

有可能，LB发送的 IPIP封装的报文，外层的 DIP 为 隧道1的 local-ip, 内层的 DIP为 
VIP2，而不是VIP1.

这样也是可以处理的，因为：
	隧道1对应的物理口收到包之后，外层DIP为其接口的IP，发现是隧道报文，查找隧道，进行解封装
```

# 参考
```c
# Linux内核参数rp_filter
https://linuxgeeks.github.io/2017/03/20/212135-Linux%E5%86%85%E6%A0%B8%E5%8F%82%E6%95%B0rp_filter/
```