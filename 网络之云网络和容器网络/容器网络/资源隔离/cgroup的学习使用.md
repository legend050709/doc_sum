```table-of-contents
```
# 基本概念
## 概述
`Cgroup` 是 Linux kernel 的一项功能：用来限制、控制与分离一个进程组的资源（如CPU、内存、磁盘输入输出等）。通过使用 cgroup，硬件资源可以在应用程序和用户间智能分配，可以进行精细化控制，从而增加整体效率。

**版本**

cgroup 分 v1 和 v2 两个版本，v1 实现较早，功能比较多，但是由于它里面的功能都是零零散散的实现的，所以规划的不是很好，导致了一些使用和维护上的不便，v2 的出现就是为了解决 v1 中这方面的问题，在最新的 4.5 内核中，cgroup v2 声称已经可以用于生产环境了，但它所支持的功能还很有限，随着 v2 一起引入内核的还有 cgroup namespace。
v1 和 v2 可以混合使用，但是这样会更复杂，所以一般没人会这样用。


## 为什么需要 cgroup


在 Linux 里，一直以来就有对进程进行分组的概念和需求，比如 session group， progress group 等，后来随着人们对这方面的需求越来越多，比如需要追踪一组进程的内存和 IO 使用情况等，于是出现了 cgroup，用来统一将进程进行分组，并在分组的基础上对进程进行监控和资源控制管理等。



## 什么是 cgroup

cgroups： 就是control groups 的缩写

### cgroup限制的资源

cgroup 主要限制的资源是：
- CPU
- 内存
- 网络
- 磁盘 I/O


当我们将可用系统资源按特定百分比分配给 cgroup 时，剩余的资源可供系统上的其他 cgroup 或其他进程使用。

![](attachments/Pasted%20image%2020240605152536.png)



## 其他

### cgroup 和 `namespace`
cgroup 和 `namespace` 类似，也是将进程进行分组，但它的目的和 `namespace` 不一样，`namespace` 是为了隔离进程组之间的资源，而 cgroup 是为了对一组进程进行统一的资源监控和限制。

- cgroup 的主要作用：管理资源的分配、限制；
- namespace 的主要作用：封装抽象，限制，隔离，使命名空间内的进程看起来拥有他们自己的全局资源；



# CPU设置

# 内存设置
# 问题
## cgroup引起的应用性能抖动
系统通过`cgroup`(控制群组：`control group`)可以对系统内的资源进行分配、管理、监控等操作。不合理的`cgroup`层级或数量可能引起系统中应用性能的不稳定。
### 问题现象
在容器相关的业务场景下，系统中的应用偶然会出现请求延时增大，并且容器所属宿主机的CPU使用率中，sys指标（内核空间占用CPU的百分比）达到`30%`及以上。例如，通过`top`命令查看Linux系统性能数据时，CPU的`sy`指标达到`30%`.
![](attachments/Pasted%20image%2020240314174148.png)

### 可能原因
例如，运行`cat /proc/cgroups` 查看当前所有控制群组的状态，发现`memory`对应的`cgroup`数目高达`2040`。
![](attachments/Pasted%20image%2020240314174300.png)

### 详细分析
可以通过perf工具动态分析，定位出现问题的原因。
（1）对系统的进程进行采样、分析
```shell
perf record -a -g sleep 10
```

（2）查看分析结果
```shell
perf report
```
![](attachments/Pasted%20image%2020240314174404.png)

从分析结果中，可得出Linux内核运行时间大多集中在`memcg_stat_show`函数，这是由于memory对应的cgroup数量过多，系统在遍历这些cgroup所导致的内核长时间运行。

除了`memory`之外，`cpuacct`、`cpu`对应的`cgroup`过多还可能使`CFS`、`load-balance`的性能受影响。

### 解决方案
对Linux实例运维时，可参考以下两条建议，避免因cgroup引起的应用性能抖动。
- cgroup的层级建议不超过`10`层。
- cgroup的数量上限建议不超过`1000`，且应当尽可能地减少`cgroup`的数量。

# 参考
```bash
# 深入理解 Linux Cgroup 系列（一）：基本概念
https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484140&idx=1&sn=c18a86d6a2d426f4d627dafd85f5ae3a&chksm=fbee4221cc99cb370fd50af1c21d504b547b8f1e052fb16dc1755df5695b4acc34c48abd825e&scene=21#wechat_redirect


# 深入理解 Linux Cgroup 系列（二）：玩转 CPU
https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484160&idx=1&sn=d593f4693a07a2a5f958fe2e0f489edd&chksm=fbee43cdcc99cadbd2df86c57d051c9742b2f4864071f6269156ec405662037c1bb9eea4c89d&scene=21#wechat_redirect


# 深入理解 Linux Cgroup 系列（三）：内存
https://mp.weixin.qq.com/s?__biz=MzU1MzY4NzQ1OA==&mid=2247484312&idx=1&sn=6f83f08d190974baf539dc36a7b4477b&chksm=fbee4355cc99ca43ce1c7abdb5e68bd0a793023bdaf2de00d0a90fc2d89d038f038c6c6a8e62&cur_album_id=1339585084919332864&scene=189#wechat_redirect


# 一篇搞懂容器技术的基石： cgroup
https://moelove.info/2021/11/17/%E4%B8%80%E7%AF%87%E6%90%9E%E6%87%82%E5%AE%B9%E5%99%A8%E6%8A%80%E6%9C%AF%E7%9A%84%E5%9F%BA%E7%9F%B3-cgroup/#%E4%BB%80%E4%B9%88%E6%98%AF-cgroup

# CPUSETS
https://www.cnblogs.com/pengdonglin137/p/17891570.html

```
