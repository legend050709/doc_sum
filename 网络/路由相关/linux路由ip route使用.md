```table-of-contents
```
# 配置ECMP
配置如下：
```
ip route add 100.0.0.0/16 nexthop via 1.1.1.1 dev vnet0 weight 1  nexthop via 2.2.2.1 dev vnet1 weight 1 
```

说明：
- `nexhop` 指定下一跳
- `dev` 指定接口
- `weight` 指定权重

查看配置：
```
$ ip route show 100.0.0.0/16
100.0.0.0/16 proto zebra  metric 20
  nexthop via 1.1.1.1 dev vnet0 weight 1 onlink
  nexthop via 2.2.2.1 dev vnet0 weight 1 onlink
```
# ip route get

## 背景
ECS云主机一般都会有多个网卡，网络的联通性是经常碰到的问题。比如在一个VM上有3个网卡，分别为ens160(和寄主机进行桥接的网卡10.0.0.128)、ens224（连接仅主机网络10.0.0.0/24的网卡10.0.0.128）和docker0(容器化平台的虚拟网卡)。当我想知道连接Internet网络的路由是经过那个网卡时，我们可以用ip route get ip地址来实现。

即：主机上存在多个网卡，多个路由，具体的到达某个目的ip的报文，匹配到哪个路由，可以通过 ip route get xxx 来查询。

## 介绍
```bash
ip route get ADDRESS [ from ADDRESS iif STRING  ] [ oif STRING ] [ tos TOS 
```

![](attachments/Pasted%20image%2020240425121733.png)

**ip route get 和 ip route show的区别**:
ip route show 是显示某个具体的路由的具体信息。
ip route get 是显示某个dip匹配了哪个路由，从哪个出口出。
```bash
注：ip route show 可以指定table，比如：ip route show table xxx.
但是  ip route get 不可以指定table，其会在结果中自动基于优先级匹配到某个table下的路由。
```


## 范例

![](attachments/Pasted%20image%2020240425122301.png)


使用方法：
```bash
(1) ip route get xxx
or 
ip route get to xxx

(2) ip route get to xxx from xxx iif xxx

```

## 注意

ip route  get xxxx 不可以指定在哪个具体的 table 中进行查询。其会自动在所有的table中，按照table的优先级进行查询。

![](attachments/Pasted%20image%2020240425122526.png)

```bash
# ip route get to 192.21.13.1 from 192.21.14.1 iif eth04
local 192.21.13.1 from 192.21.14.1 dev lo
    cache <local> iif eth04
```
# 参考
```c

```