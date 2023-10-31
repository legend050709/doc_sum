```table-of-contents
```
# 介绍
# 操作
如下所示：/proc/sys/net/ipv4/neigh/default 目录下的各个配置。
![](attachments/Pasted%20image%2020231027174848.png)

## arp_ignore
## arp_annoce

## arp_filter
## 配置arp
# 查看
## ip neigh查看arp表项
## /proc/net/stat/arp_cache说明
ARP的缓存时间约10分钟。
APR缓存列表没有对方的MAC地址或缓存过期的时候，会发送ARP请求获取MAC地址，在没有获取到MAC地址之前，用户发送出去的UDP数据包会被内核缓存到arp_queue这个队列中，默认最多缓存3个包，多余的UDP包会被丢弃。
被丢弃的UDP包可以从/proc/net/stat/arp_cache的最后一列的unresolved_discards看到。
当然我们可以通过echo 30 > /proc/sys/net/ipv4/neigh/eth1/unres_qlen来增大可以缓存的UDP包。
# 参考
```c
# [Linux内核参数之arp_ignore和arp_announce](https://www.cnblogs.com/lipengxiang2009/p/7451050.html)

https://syxdevcode.github.io/2021/03/01/Linux%E4%B8%8B%E7%BD%91%E7%BB%9C%E4%B8%A2%E5%8C%85%E6%95%85%E9%9A%9C%E5%AE%9A%E4%BD%8D/
【查看arp相关部分】

# arp_ignore=1与arp_filter=1之争
https://zhuanlan.zhihu.com/p/661896100
```