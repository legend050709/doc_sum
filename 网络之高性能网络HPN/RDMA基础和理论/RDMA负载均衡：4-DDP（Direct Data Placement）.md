```table-of-contents
```


# 概述

DDP（Direct Data Placement）既是一种技术，也可以说是一种协议，是iWARP协议栈的核心成员。
所谓DDP技术就是将报文直接放置在host内存中，那要实现这一点我们就需要知道这个报文要放置的内存地址，所以就需要协议上的支持。
在iwarp协议中其中用于实现DDP功能的协议名称就叫DDP，其具体的位置如下图，关于协议的格式我们就不详细展开了，但是从功能上我们肯定知道其中包含了报文要放置的地址（在iwarp中称作Steering Tag，简称Stag）。

![](attachments/Pasted%20image%2020260324160607.png)

![](attachments/Pasted%20image%2020260324233410.png)

在Infiniband或者RoCE中，实现DDP的功能的协议为`RDMA Extended Transport Header (RETH)`，具体协议如下图所示：

![](attachments/Pasted%20image%2020260324160657.png)

# 参考
```bash
# 16. RDMA之DDP(Direct Data Placement)
https://zhuanlan.zhihu.com/p/408817872

# RDMA中的load balance技术
http://blog.chinaunix.net/uid-28541347-id-5883126.html
```