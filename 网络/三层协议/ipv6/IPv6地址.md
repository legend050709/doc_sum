```table-of-contents
```
# 地址格式
# 地址类型
## 单播(unicast)
unicast address: 单播地址对应唯一的一个网口，发送给单播地址的数据包都只会发送到对应的网口上。
### 分类
根据不同的功能需要，单播地址又可以分为如下几个：

|单播地址类型|二进制前缀|IPV6缩写|
|---|---|---|
|link-local unicast addresss|1111111010|FE80::/10|
|unique local unicast address|1111110|FC00::/7|
|loopback address|00…1(128bits)|::1/128|
|unspecified address|00..0(128bits)|::/128|
|Global unicast adress|其他||

## 任播(anycast)
anycast address: 用于标识一组网口（通常属于不同的网络节点），发送给组播地址的数据会发送到该组网络节点中的一个。

Pv6 anycast地址的特点：

1. 可用于多个节点 - IPv6 anycast地址可以被多个节点同时使用，这些节点通常位于不同的位置或子网中。这使得IPv6 anycast地址非常适合用于实现分布式系统或负载均衡。
> 例如DNS服务器的负载均衡、路由器的故障切换、网络服务的高可用性等。

2. 最近路由选择 - 当有多个节点使用同一IPv6 anycast地址时，数据包将被路由到距离源节点最近的那个节点。这种路由选择方式可以实现更快的响应时间和更好的网络资源利用。

3. 不同于广播地址 - IPv6 anycast地址与IPv6广播地址不同。广播地址会将数据包发送到所有连接到该广播网络的设备，而IPv6 anycast地址仅将数据包发送到最近的节点。

## 多播(multicast)
> 注：广播地址(broadcast)的功能则由多播地址替代。


# 参考
```c
https://sniffer.site/2020/10/14/ipv6%E5%9C%B0%E5%9D%80%E7%9A%84%E9%82%A3%E4%BA%9B%E4%BA%8B%E5%84%BF/#/%E4%BB%80%E4%B9%88%E6%98%AFlink-local-%E5%9C%B0%E5%9D%80
```