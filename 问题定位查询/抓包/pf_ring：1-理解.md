```table-of-contents
```
# 背景
## Tcpdump的原理

# pf_ring 介绍
![](attachments/Pasted%20image%2020260512142644.png)
![](attachments/Pasted%20image%2020260512142612.png)


# pf_ring和libpcap对比
## 一般的数据包捕获（libpcap）
![](attachments/Pasted%20image%2020260512144225.png)

## 非零拷贝的pf_ring（pf_ring noZC）
![](attachments/Pasted%20image%2020260512144240.png)

## 零拷贝的pf_ring（pf_ring ZC）
![](attachments/Pasted%20image%2020260512144256.png)


# pf_ring组件
## 内核模块： pf_ring.ko
```bash
# insmod pf_ring.ko 
# 查看模块，当PF_RING激活时，会创建/proc/net/pf_ring目录，使用cat命令查看pf_ring的属性信息，如果使用rmmod停止该模块，/proc/net/pf_ring目录会自动删除

```

# PF_RING有三种工作模式
## Transparent_mode=0
```bash
insmod pf_ring.ko transparent_mode=0
```
这是**默认值**，这意味着**数据包通过标准内核机制发送到PF_RING**。_**即不需要安装任何驱动，利用Linux接口接受报文，任何驱动都能使用该模式**_

在这种设置中，数据包既被发送到PF_RING，也被发送到所有其他内核组件。所有网卡驱动均支持该模式。

## Transparent_mode=1
```bash
insmod pf_ring.ko transparent_mode=1
```
在这种模式下，**数据包直接由网卡驱动发送到PF_RING，数据包仍然传播到其他内核组件**。_**即需要安装驱动**_

在这种模式下，数据包捕获加速，因为数据包被NIC驱动程序复制而不经过通常的内核路径。

- 为了启用这种模式，必须使用支持PF_RING的网卡驱动程序。可用的启用PF_RING的驱动程序可以在PF_RING的`drivers/`目录中找到。

## Transparent_mode=2
```bash
insmod pf_ring.ko transparent_mode=2
```

# 参考
```bash
# pf ring 系列
https://github.com/moooofly/MarkSomethingDown/blob/master/Linux/PF_RING/PF_RING%20%E5%AE%B6%E6%97%8F%E5%8F%B2.md


```