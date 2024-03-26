```table-of-contents
```
# 简介
一直以来，我都知道top、vmstat、mpstat中有一个叫wa(%iowait)的cpu指标，但对它表示的具体含义又不是很清楚，故专门去网上学习了一下。

# man中含义
man文档是学习命令的第一手资料，先来看看man文档中的介绍，如下：
```bash
$ man top
wa, IO-wait : time waiting for I/O completion.

$ man vmstat
wa: Time spent waiting for IO.  Prior to Linux 2.5.41, included in idle.

$ man mpstat
%iowait
    Show the percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request.

$ man iostat
%iowait
    Show the percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request.

$ man sar
 %iowait
    Percentage of time that the CPU or CPUs were idle during which the system had an outstanding disk I/O request.

```

翻译过来就是`CPU空闲且有正在进行的磁盘IO的情况下所占的时间比例`。
# iowait真相
在man中查不到，那就只能去网上找找了，看有没有大佬在网上分享过这方面知识，在经过好一段时间的baidu、google，终于发现了如下两篇文章：
![](attachments/Pasted%20image%2020231030175717.png)

- [The precise meaning of I/O wait time in Linux](https://link.juejin.cn?target=http%3A%2F%2Fveithen.io%2F2013%2F11%2F18%2Fiowait-linux.html "http://veithen.io/2013/11/18/iowait-linux.html") ，英文的讲解，并含有一些命令实践。

- [Linux Kernel iowait 时间的代码原理](https://link.juejin.cn?target=https%3A%2F%2Fmp.weixin.qq.com%2Fs%2FLTIJliTZPB-fJXHMki48Iw "https://mp.weixin.qq.com/s/LTIJliTZPB-fJXHMki48Iw")，从内核代码层面讲解。


## 概括
可以说这两篇文章介绍得非常清楚了，我用白话概括一下内核处理过程，如下：

1. 我们知道操作系统给每个CPU分配了一个运行队列rq，用于存放可运行的线程task_stuct，调度器每次都是从此队列中取线程执行，若队列为空则CPU执行内核空闲线程（idle）的代码。
2. 当线程执行io操作时，需要将当前任务切换出去，会先将当前线程`task_stuct.in_iowait`设置为1，线程状态设置为`TASK_UNINTERRUPTIBLE`(D状态)，并将运行队列上`rq.nr_iowait`加1，而io完成后`task_stuct.in_iowait`还原为0，`rq.nr_iowait`减1，具体见内核`io_schedule_timeout`函数，可见`rq.nr_iowait`代表当前CPU上等待io操作的线程数量。
3. 当CPU执行内核空闲代码时，会判断`rq.nr_iowait`，若大于0则将空闲时间计算在iowait上，否则计算在idle上，具体见内核`account_idle_time`函数。

总结一下idle与iowait区别，如下：
- iowait时间实际上就是CPU空闲时间
>Linux上空闲时间有两类，一类是普通的idle，另一类是iowait。

- 普通idle与iowait区别
iowait是CPU空闲时，有任务正在做磁盘io操作，而idle则没有。
> 注：idle 和 iowait 并不是包含关系。如下所示，idle 几乎为0， io-wait 达到了88.6%。
> ![](attachments/Pasted%20image%2020231030191324.png)


# iowait高代表io压力大？
**iowait指标是从CPU角度看io，但毕竟不是从io层面看的。**
所以iowait高也不一定代表有问题

1. 如果程序迁移到性能更好的CPU上，由于CPU运行代码变快，会导致空闲时间变多，而iowait时间实际上就是空闲时间，所以有时会发现，性能更好的机器上iowait反而更高了。
2. 比如有这样两个程序，程序A在10s内每1s都做1次io操作，假设io操作需要1s，那么10s内的iowait是100%，而程序的IOPS是1。另一程序B在10s的前1s内并发执行了10次io操作，那么iowait是10%，而程序的峰值IOPS是10。虽然例子比较极端，但这里很明显程序B的IOPS峰值更高，但它的iowait却更低。

**虽说CPU的 iowait并不能准确反映io情况，但它也不是完全没用的，在大多数情况下，iowait高确实代表了io压力大，并且它至少提示了你应该进一步检查一下io情况，这方面iostat、iotop、blktrace等可以做到。**

>另外，从上面的iowait原理中你会发现，处在iowait的线程/task，其必定是D状态的(TASK_UNINTERRUPTIBLE)，这在之前也提到过，D状态是会影响系统负载的。

# 测试实验
`stress -d`命令可以模拟做大量的磁盘操作，并使用taskset将stress绑定到3号核上运行，如下：
```c
$ taskset --cpu-list 3 stress -d 1 --hdd-bytes 100G
stress: info: [9160] dispatching hogs: 0 cpu, 0 io, 0 vm, 1 hdd
```

然后使用`mpstat -P ALL 1`观察iowait情况，发现3号核iowait高达88%，如下：
![](attachments/Pasted%20image%2020231030180452.png)

然后新开一个shell窗口，使用`stress -c`命令模拟大量的CPU运算，也使用taskset绑定到3号核上运行，如下：
```c
$ taskset --cpu-list 3 stress -c 1
stress: info: [9178] dispatching hogs: 1 cpu, 0 io, 0 vm, 0 hdd
```
再次使用`mpstat -P ALL 1`观察iowait情况，发现3号核iowait变成了0%，而%usr几乎满载，如下：
![](attachments/Pasted%20image%2020231030180721.png)

如果你理解了上面的内容，就能理解这个示例了。  
- 第一次因为只有`stress -d`在做io操作，所以CPU基本都是空闲的，而空闲期间基本都是有在做io的任务存在的，所以iowait高。  
- 第二次因为加入`stress -c`使得CPU不再空闲，而iowait实际上就是空闲时间，所以iowait自然就变成0了。

当然，如果你有兴趣，还可以试下这两个命令，它们可以将`%sys`、`%nice`变成100%，如下：
```c
# 使得%sys满载
$ taskset --cpu-list 3 dd if=/dev/zero of=/dev/null bs=1M count=1000000

# 使得%nice满载
$ taskset --cpu-list 3 nice -n 19 stress -c 1

```

# 小结
io-wati 代表的是 CPU空闲 且 有任务正在进行磁盘IO。
io-wait 是从CPU的角度来看IO的：
它的值低，不代表IO压力小。
它的值高，也不代表IO压力大，但是可以通过io相关工具（iostat、iotop）继续查看IO的情况。
> 注：对于CPU独占的进程/线程，如果CPU的io-wait高，就代表着IO压力大。

# 参考
```c
https://juejin.cn/post/7063709470173429768
https://mp.weixin.qq.com/s?__biz=Mzg2OTc0ODAzMw==&mid=2247501965&idx=1&sn=f4bad528c2b7728479fce2d653b1969f&source=41#wechat_redirect
https://veithen.io/2013/11/18/iowait-linux.html
```