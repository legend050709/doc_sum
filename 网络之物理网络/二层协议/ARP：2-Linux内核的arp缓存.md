```table-of-contents
```
# 问题
使用 `bind` 作为 `dns`服务器，出现了`dns` 查询的超时问题。
通过在 问题的`dns`服务器上指定指定进行查询。如下所示：

```bash
# for i in `seq 0 100`;do echo $(date +%T.%N) && dig xxx-test.internal @127.0.0.1 +short;done
```

![](attachments/Pasted%20image%2020250322121032.png)

## 复现方法
```bash
Steps to reproduce:  
1) sudo apt-get install dnsmasq  
2) sudo sysctl -w net.ipv4.neigh.default.gc_thresh1=1  
3) sudo sysctl -w net.ipv4.neigh.default.gc_thresh2=1  
4) sudo sysctl -w net.ipv4.neigh.default.gc_thresh3=1  
5) dig @127.0.0.1 google.com

Result:  
~$ dig @127.0.0.1 google.com  
../../../../lib/isc/unix/socket.c:2104: internal_send: 127.0.0.1#53: Invalid argument  
../../../../lib/isc/unix/socket.c:2104: internal_send: 127.0.0.1#53: Invalid argument  
../../../../lib/isc/unix/socket.c:2104: internal_send: 127.0.0.1#53: Invalid argument

; <<>> DiG 9.10.3-P4-Ubuntu <<>> @127.0.0.1 google.com  
; (1 server found)  
;; global options: +cmd  
;; connection timed out; no servers could be reached
```

# 分析
顺着这个错误提示进行谷歌，然后就找到了这篇文章:
[https://bugs.launchpad.net/ubuntu/+source/dnsmasq/+bug/1702726](https://bugs.launchpad.net/ubuntu/+source/dnsmasq/+bug/1702726) ，虽然是说出现在 dnsmasq 上的文章，但是跟我们的错误原因是相同的，其中里面一个网友的下面描述解开了我们多年的疑惑：
```bash
Testing this, the results are not quite as clear-cut as the example. I  
don't always see the same errors.

Also, I don't understand why the send() calls in dig, which are sending  
UDP packets over the loopback interface, should return the invalid  
argument. ARP is not needed over loopback, surely?

Looking at an strace of dnsmasq, what I see is that either the query  
never arrives at dnsmasq, or it gets answered correctly but the answers  
never makes it back to dig: the UDP packets are being dropped in the  
kernel. (In the later case, the send() of the reply gets the same  
invalid argument error that dig is seeing)

The lesson here is that if the arp-cache overflows, UDP, (even over lo)  
drops packets. There's really not much dnsmasq can do about that. I  
guess the only answer is "don't let your arp-cache overflow".

(or possibly, work on getting the kernel to behave better under these  
circumstances)

TL;DR not a dnsmasq bug.
```

总结一下就是如果操作系统的 arp 缓冲区不够，是没法发送 UDP 数据包的，而 DNS 请求就是基于 UDP 协议的。

# 查看
 arp 缓冲区不够的查看
 
## dmesg 查看
![](attachments/Pasted%20image%2020250320174757.png)

## `/proc/net/stat/arp_cache`文件查看
查看arp状态：cat /proc/net/stat/arp_cache ，table_fulls统计：

![](attachments/Pasted%20image%2020250322121633.png)


# 原因
两个方面的原因：
1》linux内核配置的arp表项缓存个数太小。
2》设备接口上配置的`ip`在一个大的网段内。比如: `/16`或者`/20`的网段。

接口 IP在一个大的网段内，导致同网段的 ARP 表项个数，超过了配置的缓存 ARP 表项的个数。

# 解决
解决方案就是增加其缓冲区大小，但是这个需要改宿主机的配置，最终把如下命令发给运维来执行：
```bash
sysctl -w net.ipv4.neigh.default.gc_thresh1=1048576
sysctl -w net.ipv4.neigh.default.gc_thresh2=4194304
sysctl -w net.ipv4.neigh.default.gc_thresh3=4194304
```

```text
sysctl -a |grep net.ipv4.neigh.default.gc_thresh
gc_thresh1：存在于ARP高速缓存中的最少层数，如果少于这个数，垃圾收集器将不会运行。缺省值是128。

gc_thresh2 ：保存在 ARP 高速缓存中的最多的记录软限制。垃圾收集器在开始收集前，允许记录数超过这个数字 5 秒。缺省值是 512。

gc_thresh3 ：保存在 ARP 高速缓存中的最多记录的硬限制，一旦高速缓存中的数目高于此，垃圾收集器将马上运行。缺省值是1024。
```

# 其他

# 参考
```bash
# dnsmasq fails when the ARP cache is full
https://bugs.launchpad.net/ubuntu/+source/dnsmasq/+bug/1702726

# DNS 解析失败问题追踪
https://blog.whyun.com/posts/dns-lookup-failed-due-to-udp-cache/

# 解密网络丢包排查方法：从抓包到故障修复的实用指南 (++++++)
https://zhuanlan.zhihu.com/p/692288382
```