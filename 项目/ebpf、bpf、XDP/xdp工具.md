```table-of-contents
```

# 抓包工具xdp-dump

## 问题

因为tcpdump是基于PACKET套接字的，而PACKET套接字是运行在Linux内核协议栈的，XDP在内核协议栈之前，所以tcpdump够不到它，也就无法抓取它的数据包。因此，需要一个xdpdump。

## 分析

之所以会这样，在于XDP还没有实现一个统一类似处理PACKET套接字的 **ptype_all框架**（至少在目前没有这种机制）。
这很容易理解，因为XDP本身就是网卡强相关的，不适合做generic操作。

## 现有的xdp-dump

## 自己实现xdp-dump

实现xdpdump之前，必须要解决的一个问题就是：

如何让两个或多个独立的eBPF程序在XDP实现串联？
这是必须的，因为抓包只是一个旁路功能，它不能影响到既有的XDP上eBPF程序的运行，如果当前某网卡的XDP运行着一个eBPF程序，我们希望的是xdpdump和它一起工作，而不是替换它。


# 参考
```bash
# xdp-tools 多种实现
https://github.com/xdp-project/xdp-tools

# dog250: # eBPF程序之间的协作-简单实现一个xdpdump
https://blog.csdn.net/dog250/article/details/103686162
```