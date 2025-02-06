```table-of-contents
```


# udp程序(named)收发包出现丢包的性能优化


## 收包丢包
### 统计查看
#### snmp中udp的RcvbufErrors 统计
### 设置
```bash
sysctl -w net.core.rmem_max=8000000
```
### 注意
`rmem_max` 设置之前，程序已经启动，那么其实是不生效的。
设置 `rmem_max` 之后， 需要重启 named 程序 才可以生效。

### 丢包范例的分析流程

## 发包丢包
### 统计查看
#### snmp中udp的SndbufErrors统计
### 设置
```bash
ip link set ethx txqueuelen 4000
```
### 注意
### 丢包范例的分析流程


# 参考
```bash
# udp缓冲区
https://blog.csdn.net/legend050709/article/details/128437143

# centos优化udp
https://www.volcengine.com/theme/2450474-C-7-1

# UDP SndbufErrors & ENOBUFS
https://zhensheng.im/2021/03/12/udp-sndbuferrors-enobufs.meow

# 问题记录】进程调度导致 UDP 丢包问题分析
https://blog.csdn.net/qq_45527937/article/details/142526697

# Linux内核UDP收包为什么效率低？性能怎么优化(超详细讲解)
https://zhuanlan.zhihu.com/p/457481940

# [关于UDP-读这篇就够了（疑难杂症和使用）](https://www.cnblogs.com/yajunLi/p/6605110.html)

```