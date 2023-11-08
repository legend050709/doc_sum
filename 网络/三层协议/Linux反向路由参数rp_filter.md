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

# 参考
```c
https://linuxgeeks.github.io/2017/03/20/212135-Linux%E5%86%85%E6%A0%B8%E5%8F%82%E6%95%B0rp_filter/
```