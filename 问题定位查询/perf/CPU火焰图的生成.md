```table-of-contents
```
# 介绍
![](attachments/image%20(1).png)
火焰图是基于`stack`信息生成的`SVG` 图片, 用来展示 CPU 的调用栈。
说明：
y 轴表示调用栈，每一层都是一个函数。调用栈越深，火焰就越高，顶部就是正在执行的函数，下方都是它的父函数。
x 轴表示抽样数，如果一个函数在 x 轴占据的宽度越宽，就表示它被抽到的次数多，即执行的时间长。
火焰图就是看顶层的哪个函数占据的宽度最大。只要有"平顶"(plateaus)，就表示该函数可能存在性能问题。颜色没有特殊含义, 因为火焰图表示的是 CPU 的繁忙程度, 所以一般选择暖色调.

# 基础
## IPC
### 定义
IPC（Instructions Per Cycle）是指每个周期的指令执行数，用于衡量处理器的执行效率。IPC越高，表明处理器在相同频率下可以执行更多指令。

在现代高性能处理器中，每个时钟周期可以发射4~6条指令，因此在理想情况下，IPC可以达到4~6以上。然而在实际情况中难以达到，主要原因包括数据依赖、缓存未命中（Cache Miss）、分支预测错误等。

因此，较低的IPC意味着CPU执行效率低，CPU有大量时间处于停滞等待状态，无法有效利用硬件的峰值性能。


### 使用perf工具测量IPC
perf 可以监测多种性能事件，例如指令周期、缓存命中率、分支预测准确率、IPC（每周期指令数）等。`perf` 提供了详细的报告，帮助用户识别性能瓶颈，从而优化系统或应用。

perf工具测量IPC示例：
```bash
perf stat -C 32,4,36,8,40,12,44,16,48,20,52,24,56,28,60,2,34,6,38 sleep 10
```

该命令使用 `perf stat` 来测量指定CPU核心（-C 后的核心列表）上的性能指标，运行 `sleep 10` 来采集10秒的数据，包括指令数、周期数和IPC（每周期指令数）。

![](attachments/Pasted%20image%2020260301133632.png)

从这个结果可见，`perf stat`命令默认包含了测量指令数（instructions）和周期数（cycles），使用指令数除以周期数便得到了IPC。

### 是否有可能出现“IPC增加，但是IOPS性能数据下降”的情况呢？

例如，基准IPC为0.8，IOPS为200万，而异常情况下IPC为0.9，IOPS为180万。这表明CPU虽然在卖力地工作，但却做了许多无用功。
这种情况可以通过对比两种IPC情况下的`perf`火焰图，再生成差分火焰图 来进行分析。


# 火焰图类型

在性能分析中，我们常常会用到如下所示的火焰图：

![](attachments/Pasted%20image%2020260301134619.png)

一般来说，我们将这种火焰图称为`on-cpu`火焰图，可以用来记录CPU上运行的程序的占比情况。

根据创始人Gregg给出的类型，常见的火焰图类型有5种，on-cpu、Off-CPU、Memory、Hot/Cold、Differential差分火焰图。

|   |   |   |   |   |
|---|---|---|---|---|
|类型|横轴|纵轴|解决的问题|采样方式|
|CPU|CPU 占用时间|调用栈|找出CPU占用高的问题函数，分析代码热路径|固定频率采样CPU调用栈|
|Off-CPU|阻塞时间|调用栈|i/o、网络等阻塞场景导致的性能下降；锁竞争、死锁导致的性能下降问题|固定频率采样阻塞事件调用栈|
|Memory|内存申请/释放函数调用次数，或分配的总字节数|调用栈|内存泄漏问题、内存占用高的对象/申请内存多的函数、虚拟内存或物理内存泄漏问题|跟踪malloc/free、跟踪brk、跟踪mmap、跟踪页错误|
|Hot/Cold|CPU和Off-CPU结合|调用栈|需要结合CPU占用以及阻塞分析的场景、Off-CPU无法直观判断问题的场景|CPU和Off-CPU结合|
|Differential|前后火焰图之间的差异|调用栈|性能回归问题、调优效果分析|与前后火焰图一致|



## on-cpu火焰图和off-cpu火焰图区别

`on-CPU`/`off-cpu` 的区别就是一个是用于CPU是性能瓶颈，一个是IO是性能瓶颈.

on-CPU：线程在CPU上运行时花费时间。  
off-cpu：时间花在等待 I/O、锁、定时器（timer）、页面/交换（paging/swapping,）等阻塞状态上。

CPU火焰图展现的是在CPU上发生的事情，为下图中的红色部分。Off-CPU火焰图展现的是在CPU之外发生的事情，也就是在 I/O、锁、定时器、分页/交换等阻塞时等待的时间，在下图中用蓝色展示。

![](attachments/Pasted%20image%2020260301140830.png)

I/O期间有File I/O、Block Device I/O，通过采集进程让出CPU时调用栈，可以知道哪些函数正在频繁地等待其它事件，以至于需要让出CPU，通过采集进程被唤醒时的调用栈，可以知道哪些函数让进程等待的时间比较长。

![](attachments/Pasted%20image%2020260301141001.png)



## 适用场景
**如果无法确定当前的系统瓶颈, 可以通过压测工具来确认** 。
> 通过压测工具看看能否让CPU使用率趋于饱和, 如果能那么使用 `On-CPU` 火焰图； **如果不管怎么压, CPU 使用率始终上不来, 那么多半说明程序被 `IO` 或锁卡住了, 此时适合使用 `Off-CPU` 火焰图**. 你可以通过压测工具进行测试。



# on-cpu火焰图
Github上有`Brendan D. Gregg` 的 `Flame Graph` 工程实现了一套生成火焰图的脚本.我们可以直接克隆下来直接用。
```bash
git clone https://github.com/brendangregg/FlameGraph.git
```
## 流程
生成火焰图，我们一般都遵循以下流程
![](attachments/Pasted%20image%2020240130153434.png)

- `捕获堆栈`: 使用`perf`捕捉进程运行堆栈信息
- `折叠堆栈`: 对抓取的系统和程序运行每一时刻的堆栈信息进行分析组合, 将重复的堆栈累计在一起, 从而体现出负载和关键路径，通过`stackcollapse`脚本完成
- `生成火焰图`：分析 stackcollapse 输出的堆栈信息渲染成火焰图

### 获取堆栈
```bash
// 指定某个coreid
perf record -F 99 -C 1 --call-graph dwarf -- sleep 30

// 指定某个进程id
perf record -F 99 -p xxx --call-graph dwarf -- sleep 30

// 指定某个线程id
perf record -F 99 -t xxx --call-graph dwarf -- sleep 30

默认在当前路径下生成一个 perf.data 文件
```
### 折叠堆栈
`Flame Graph`中提供了抓取不同信息的脚本，可以按需使用。下面我们需要对捕获到的进程堆栈信息`perf.data`进行折叠，生成折叠的堆栈信息:
```bash
 perf script -i /root/perf.data &> /root/perf.unfold
```

用 `stackcollapse-perf.pl` 将 perf 解析出的内容 `perf.unfold` 中的符号进行折叠
```bash
 ./stackcollapse-perf.pl /root/perf.unfold &> /root/perf.folded
```
最后就是生成火焰图
```bash
./flamegraph.pl /root/perf.folded > /root/perf.svg
```

最后在谷歌浏览器上打开该火焰图文件（perf.svg）：
![](attachments/Pasted%20image%2020240130153922.png)


### 小结
```bash
(1) 先执行perf record, 默认生成了  ./perf.data
perf record -F 99 -C 1 --call-graph dwarf -- sleep 30


(2) 然后执行脚本生成svg
# cat get_perf_graph.sh

outfile=$1
perf script -i ./perf.data &> ./${outfile}.unfold
./FlameGraph/stackcollapse-perf.pl ./${outfile}.unfold &> ./${outfile}.folded
./FlameGraph/flamegraph.pl ./${outfile}.folded > ./${outfile}.svg


(3) perf report 直接读取 perf.data
perf report -i ./perf.data --sort comm,dso,symbol

perf report -i ./perf.data 
```

# 红蓝差分火焰图（Differential Flame Graph）

差分火焰图（Differential Flame Graph）通过对比两组火焰图的数据差异，将性能变化可视化。
通常，一组火焰图为基准状态，另一组为调整后的状态。对比结果会用不同颜色表示变化——如红色表示性能退化，绿色表示性能提升。差分火焰图在性能调优和资源利用的对比分析中很有帮助，尤其是测量两种IPC的差异对比。
参考：[Differential](http://www.brendangregg.com/blog/2014-11-09/differential-flame-graphs.html "Differential")  


下面我们将介绍差分火焰图。主要介绍以下的内容：
- 为什么要有差分火焰图
- 如何生成差分火焰图
- 差分火焰图的形成原理
- 开源项目`pyroscope`

## 为什么要有差分火焰图？

在性能分析和优化的过程中，我们经常使用使用火焰图；而当一轮优化完成过后，我们需要做回归验证来判断性能是否提升。除了尝试通过诸如延时、qps等指标来判断是否优化成功，我们也可以通过火焰图的热点变化来进行判断。

而当比较两个火焰图的时候，我猜你大概会这么做：

![](attachments/Pasted%20image%2020260301135252.png)

打开两个标签，横向的比对两个火焰图的区别，这样麻烦且不够精确。因此，我们尝试引入差分火焰图：

![](attachments/Pasted%20image%2020260301135303.png)

差分火焰图是两个火焰图A、B比较之后的结果，我们可以认为是B-A。往往采取红蓝配色，我们也可以称之为是红蓝对比火焰图，其中红色代表增长，蓝色代表减少。例如`deflate_slow`函数：

![](attachments/Pasted%20image%2020260301135336.png)

这个函数是红色的，说明A火焰图相对于火焰图B，函数`deflate_slow`的调用变多了18.16%；而蓝色的部分则是相反的，表达A火焰图相较于B火焰图，某函数的调用变少了。

想象一下现在有一个火焰图，我们优化了其中某个调用占比1%的函数，如果采取对比方法，恐怕很难发现，而采取差分火焰图，则能够很快的发现哪里变化了以及变化了多少。

总结：**差分火焰图可以帮助我们快速的进行回归验证，比较两个火焰图的变化。**

## 如何生成差分火焰图

步骤和说明：
- 1）采集堆栈1
- 2）再采集堆栈2
- 3）基于堆栈2生成一个火焰图，所有的函数帧都会**以堆栈2为准**
- 4）根据`堆栈2-堆栈1`的差值对上面的火焰图染色，如果2中出现的帧次数更多，则是**红色**，如果次数更少则是**蓝色**
红蓝颜色的**饱和度是表示差距大小**，颜色越深表示次数相差越大


我们可以用如下的方式生成差分火焰图：
```bash
# 第一次Profiling结果
perf record -ag -F 999 -- sleep 20
perf script > A.stacks

# 第二次Profiling结果
perf record -ag -F 999 -- sleep 20
perf script > B.stacks

# 下载FlameGraph仓库
git clone --depth 1 http://github.com/brendangregg/FlameGraph 

# 折叠A.stacks和B.stacks
cd FlameGraph 
./stackcollapse-perf.pl ../A.stacks > A.folded
./stackcollapse-perf.pl ../B.stacks > B.folded

# 基于折叠结果做差
./difffolded.pl -n A.folded B.folded > diff.folded

# 生成差分火焰图
./flamegraph.pl diff.folded > diff.svg
```

### 差分火焰图的形成原理

从实现角度而言，**火焰图是一种“栈-值”数据结构的图，只要满足该数据结构的数据，都可以转化为火焰图的展示方式**。

在生成差分火焰图的过程中，和生成一般的火焰图不同的一步是我们调用了`difffolded.pl`对两个折叠后的堆栈文件进行了比对，并生成了比对后的堆栈文件。
我们不妨尝试构造两个堆栈数据，来举例说明：
```text
# 堆栈数据1
[root@VM-16-2-centos ~]# cat Atest.folded 
YDService;foo 1000
YDService;other 1000
A;B;C 1000
A;B;D 400

# 堆栈数据2
[root@VM-16-2-centos ~]# cat Btest.folded 
YDService;foo 1000
A;B;C 400
A;B;D 1000
```

接着对这两个数据进行差分：
```bash
[root@VM-16-2-centos ~]# ./FlameGraph/difffolded.pl Atest.folded Btest.folded > diff.folded

[root@VM-16-2-centos ~]# cat diff.folded 
YDService;foo 1000 1000
A;B;D 400 1000
YDService;other 1000 0
A;B;C 1000 400
```

可以看到，数据格式变成了`调用栈 调用次数1 调用次数2`的形式。我们将数据生成火焰图看看：

![](attachments/Pasted%20image%2020260301135858.png)

我们不妨生成一个B数据的火焰图看看：
![](attachments/Pasted%20image%2020260301135917.png)

可以看到除了配色，这两个火焰图的结构是完全一致的。我们可以得出一个结论：**差分火焰图以采样数据B为基准**。既然是这样的话，当涉及到回归的时候，我们应当将初始数据作为采样数据B，这样我们的比较基准就是初始数据，这样能够更好的进行比较。

我们再来看配色：
![](attachments/Pasted%20image%2020260301135957.png)

可以看到这里`A->B->C`这个调用栈的调用次数是减少了25%，那么这个25%是相对什么来说的呢？
我们不妨看原始数据，`A->B->C`在数据A中出现了1000次，数据B中出现了400次，减少调用了600次。而整体的调用次数是2400，所以这里的减少百分比是相较于所有的采样数据而言的。

**注意**：
看到这里不知道聪明的读者有没有发现一个问题，`YDService->other`这个调用对比消失了。这是因为我们前文说了差分火焰图是以采样数据B为基准的，如果某个调用栈在B中完全没有出现的话，那我们就无法比对前后的变化。如果想要初步解决这个问题，可以反转顺序来进行比较，也即将采样数据A作为基准，这样就能看到A中有的部分了。
==因此，在进行回归验证的时候，我们可以考虑进行两次反转的差分，这样更能帮助我们发现变化的地方==。


## Continuous Profiling

Continuous Profiling是一种**持续性能分析**技术，它从任何环境（包括生产环境）连续收集代码行级别性能数据。之后提供数据的可视化，使开发人员可以分析、排除故障和优化他们的代码。

与传统的静态分析技术不同，Continuous Profiling可以在实际运行环境下获取性能数据，并且**不会对应用程序的性能产生显著的影响**。这使得它可以更加准确地分析应用程序的性能问题，并且可以在**实际部署环境中**进行性能优化和调试。

开发人员可以为生产环境实施持续集成和部署。然后，生产反馈到Continuous Profiler，这是一个反馈回路，为开发人员提供剖析数据的回馈。

### pyroscope

[pyroscope](https://link.zhihu.com/?target=https%3A//github.com/grafana/pyroscope)是一个开源项目，目前已被grafana收购。其主要做的是持续性的Profiling，能够让我们从时间的维度查看系统上的调用栈信息：

![](attachments/Pasted%20image%2020260301140407.png)

其提供了Diff的功能，可以帮助我们查看某两个时间段内的调用栈变化：
![](attachments/Pasted%20image%2020260301140433.png)



# off-cpu火焰图
## 场景
通过压测工具，如果不管怎么压, CPU 使用率始终上不来, 那么多半说明程序被 `IO` 或锁卡住了, 此时适合使用 `Off-CPU` 火焰图。

另外，就是通过压测工具，看压测的qps是否和 **context-switch** 成正比。
```bash
perf stat -p PID sleep 20 // 查看一段时间内的 context-switch个数；
```

Off-CPU 能够识别的类型包含：==阻塞在 I/O、锁、定时器、缺页、swap等 事件上的时间消耗==。

Off-CPU 分析关注的是：线程不在 CPU 上执行的时间花在哪了。
典型场景：
- 等锁（mutex / spinlock）
- 等 IO（磁盘 / 网络）
- 等内核事件（epoll_wait）
- 被调度器抢占

因此，**Off-CPU 本质上依赖：调度事件（context switch）**。每当一个线程从运行状态（On-CPU）切换到阻塞状态（Off-CPU），或者反过来，操作系统调度器都会介入。

## 分析要点
off-cpu火焰图分析有几个要点：
(1)二进制需要带 frame-point。
不巧的是，大部分二进制都不带，因此需要重新编译一个带 fp的版本；

(2)聚焦业务处理路径上的Off-cpu。
和On-cpu不一样，不在业务路径上的Off-cpu也会很突出；

(3)在问题显现时进行采集。
低压力下，线程本身利用率就比较低，到处都是 Off-cpu，会有很大的干扰。

## off-cpu的原理

要分析Off-cpu瓶颈，一个有力的工具就是Brendan Gregg提出的 [off-cpu火焰图](https://www.brendangregg.com/offcpuanalysis.html)，
==该工具基于 bcc-tools，利用内核BPF功能==。
统计**每个线程从其挂起到被重新唤醒的这部分时间，并记录线程挂起时的调用栈**。


## 为什么 Off-CPU 特别需要 eBPF？

**Off-CPU分析依赖于对调度事件的追踪**，而调度事件在高负载系统中发生得极其频繁（每秒百万级）。
传统工具试图将所有事件传回用户态处理，导致了无法接受的CPU和I/O开销。
eBPF通过**在内核中就地处理和聚合数据**，避免了昂贵的数据传输和上下文切换，从而使生产环境下的高频事件追踪成为可能。

这句话其实就是：==你想观察“线程为什么没在跑（Off-CPU）”，  但为了观察它，使用传统的工具（比如：`perf`, `strace`），你自己反而可能制造大量额外开销；而eBPF能有效解决此问题。==

### 调度的发生时机
Linux 调度器的核心函数：
```c
- `schedule()`
- `finish_task_switch()`
- `context_switch()`
```

每一次：
- 线程 sleep
- 线程 wakeup
- 锁竞争
- 阻塞 IO
- 时间片用完

都会发生一次调度。

### 传统工具（如perf）面临的性能挑战

当使用传统工具进行Off-CPU追踪时，频繁的调度事件会引发严重的性能雪崩，主要体现在以下两个方面：

**（1）用户态与内核态的频繁切换**：  
传统工具通常需要将内核产生的事件数据（Event Data）实时或定期转储（Dump）到用户态进行处理或写入磁盘。
- **数据传输开销**：每秒数百万个事件意味着巨大的数据吞吐量。将这些数据从内核空间拷贝到用户空间，再写入磁盘，会消耗大量的CPU周期和I/O带宽 12。
- **性能损耗实测**：在极端负载下，使用`perf`进行全量调度追踪可能导致系统吞吐量下降9%到13%，且后期处理（如符号解析）极其耗时 。例如，仅10秒的追踪可能产生数百MB的数据文件，并需要数十秒甚至数分钟进行后处理。

**（2）反馈循环（Feedback Loop）**
追踪工具本身也是运行在系统上的进程。如果追踪工具因为处理大量数据而消耗过多CPU或产生大量I/O，它自身也会引发调度事件，从而被自己追踪到。这种“观察者效应”会进一步加剧系统负载，导致测量结果失真。

### 为什么eBPF是解决方案？
#### 传统方式
```bash
内核发生事件
↓
拷贝数据到用户态
↓
用户态分析
↓
写文件
```


#### eBPF 方式

eBPF 方式，可以在内核里运行自定义程序，流程变成：
```bash
调度发生
↓
eBPF 在内核中统计
↓
只把“聚合后的结果”传给用户态
```

例如：
- 统计某函数累计 Off-CPU 时间
- 聚合成 histogram
- 聚合成 per-stack 延迟

而不是把每一次事件都发出来。

#### 小结

|方式|代价|
|---|---|
|每次调度都传到用户态|极高|
|内核聚合后再传|很低|

eBPF（Extended Berkeley Packet Filter）通过内核态聚合（In-Kernel Aggregation）机制，做到：
**（1）数据缩减（Data Reduction）**：
eBPF允许用户编写程序在内核中直接运行。在Off-CPU分析中，eBPF程序可以在内核态捕获调度事件，直接记录并聚合数据（例如，计算每个堆栈的阻塞时长并存入Map结构），而不是将原始事件逐条发送给用户态。
只传结果：用户态程序只需要定期读取聚合后的统计摘要（Summary），数据量从“每秒数百万条事件”降低到“每秒几KB的统计表”

**(2)极低的开销**
由于避免了频繁的内核-用户态数据拷贝和磁盘I/O，eBPF的开销非常低。
性能对比：实测显示，eBPF在同样的负载下，性能损耗通常控制在**6%左右**（主要集中在初始化和最终数据读取阶段），且不会随着追踪时间的延长而线性增加开销.


## off-cpu和on-cpu对比

|类型|分析什么|
|---|---|
|On-CPU|CPU 在执行什么函数|
|Off-CPU|线程没在 CPU 上时在等什么|

## 优化思路
### 不加锁或者减少锁的粒度
比如：整个hash 表进行加锁，减少为针对hash表的链进行加锁。

### 增大 ring
比如，多生产者，多消费者操作一个ring，可能存在加锁，或者通过CAS减少开销。
如果 ring 太小了，消费者个数大于ring的大小，那么就存在一些消费者获取不到而等待，这个比加锁更加耗时，此时应该增大ring的个数，保证ring的个数大于消费者的个数。

# 参考
```bash
# [火焰图：全局视野的Linux性能剖析]
(https://segmentfault.com/a/1190000023103508)

# 通过IPC指标诊断性能问题
https://aijishu.com/a/1060000000490205
```