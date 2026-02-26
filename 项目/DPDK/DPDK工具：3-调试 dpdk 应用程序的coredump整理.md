```table-of-contents
```
# mempool相关的问题
## 背景
DPDK作为高性能的应用程序，一般在转发线程中使用`mempool`来进行内存的申请以及释放，而不是通过`rte_malloc`，以及 `rte_free`.
注：`mbufpool`的其实也是一种`mempool`。


## mempool或mbufpool内存泄漏问题
### 背景
在`DPVS`的工作中，曾经遇到过`mbufpool`的内存泄漏，程序运行了七个月，最终`mbufpool`完全泄漏，进而导致 `failover`的探测流量，`ping lan-gw`不通，才发现了`mbufpool`的完全泄漏。

> 后来查询了原因是：在`Synproxy`开启的情况下，收到了`ttl<=1`的`syn`包，会发送`icmp`差错报文，并进行丢弃，但是丢弃存在问题，导致泄漏。

### 分析

在不知道具体哪里泄漏时，分析，程序可以运行七八个月，并且通过`mbufpool`中可用元素个数的监控统计, 可以发现一直在非常缓慢的下降，就证明了存在泄漏。
然后存在泄漏，又是缓慢下降，就说明，是一种非常规的代码逻辑中存在了泄漏。

当时是一头雾水，其实也想过，查看`mbufpool`中各个已经使用的`mbuf`，将其`mbuf`信息打印出来，看看是否存在共同点。但是没有这样的接口。后来是通过看代码，才获取到了泄漏点。

### 思路
在`mempool`内存泄漏时，为了找到泄漏的元素是否存在共性。
可以考虑实现一个**通用的**打印`mempool`中各个**已经使用的成员信息的接口**；
主要是打印元素的地址信息即可，得到了地址，后续可以通过`gdb`，来打印具体的信息。

> 注：(1) 对于`mbufpool`这种`mempool`，其每个成员是`mbuf`，`mbuf`又是分层的，通过`gdb`又不太容易看，可以考虑直接在程序中实现打印`mbufpool`中`mbuf`元素的接口。
> （2） 对于其他的`mempool`，打印元素信息，直接打印地址，然后在`gdb`中查看即可。


# DPDK程序占用的大页
## dpdk参数`--file-prefix`
## dpdk参数`--huge-unlink`
## dpdk参数`--no-shconf`


# 大页内存的coredump
## 问题
默认DPDK程序启动之后，通过`rte_malloc`, `rte_mempool`, `rte_ring` 分配的内存都是从共享大页中分配来的。
程序的 `coredump_filter` 默认是`0x33`, 那么就会出现一个问题：
```bash
(1) 共享大页内存，不会被写入到coredump中；
(2) 通过gdb 查看从共享大页内存中申请的内存(rte_malloc的内容, rte_mempool的内容，rte_ring的内容)，查看内存空间都是0;
    但是程序中非从大页内存中申请的全局变量的内容不是0；
```


## 分析

### coredump_filter 各个bit 的含义

`coredump_filter` 是一个 `bitmask`，控制哪些 VMA 会被 dump。

`#man core`, 如下所示：

![](attachments/Pasted%20image%2020260121124719.png)

```bash
(1) File-backed Mapping (文件映射):

当你使用 `mmap` 系统调用时，指定了一个打开的文件描述符（fd）。内核会将文件内容映射到进程的地址空间。
此时，这块内存的“后端（Backing Store）”就是磁盘上的那个文件。

- 读取：如果内存不在物理内存中，触发缺页异常，内核从磁盘文件读取。
- 回写：如果是共享映射（Shared），对内存的修改最终会写回磁盘文件。

(2) Anonymous Mapping (匿名映射):
没有关联任何文件（如通过 `malloc` 分配的内存或 `mmap` 时使用 `MAP_ANONYMOUS` 标志）。它的“后端”是系统的交换分区 (Swap)。

```

```bash
（1）bit 0  Dump anonymous private mappings: 
	该bit位被设置，默认可以打印 匿名私有mmap的非大页内存，比如：malloc 申请的内存；
	
（2）bit 1  Dump anonymous shared mappings：
	该bit位被设置，默认可以打印 匿名共享mmap的非大页内存；

（3）bit 2  Dump file-backed private mappings：
	该bit位被设置，默认可以打印 文件私有mmap的非大页内存；
	
（4）bit 3  Dump file-backed shared mappings：
	该bit位被设置，默认可以打印 文件共享mmap的非大页内存；

（5）bit 4 (since Linux 2.6.24) Dump ELF headers：

（6）bit 5 (since Linux 2.6.28) Dump private huge pages：
	该bit位被设置，默认可以打印 私有mmap的大页内存；

（7）bit 6 (since Linux 2.6.28) Dump shared huge pages：
	该bit位被设置，默认可以打印 共享mmap的大页内存；

```


### 默认 `coredump_filter = 0x33`的表项

在默认 `coredump_filter = 0x33` 的情况下，  通过 `rte_malloc / rte_mempool` 分配的 DPDK 内存（==`rte_eal_init` 不加特殊参数情况下，默认`mmap`的大页是file backed 共享大页==）几乎必然在 core 中表现为 “全 0” 或“不可用”。

这是 预期行为。
其实==并不是因为内存真的被清空了，而是因为内核在生成 Core 文件时，只记录了虚拟地址范围，但没有把对应的大页物理内容dump进去；
即：gdb 看到的“全 0”只是空洞填充，不是真实数据。==；


在 `0x33` 过滤器下，内核的行为如下：
**元数据保留**：`rte_malloc` 指针、`mempool` 的管理头（Metadata）被赋值给某个全局变量（或者作为全局变量的结构体），这样的全局变量通常位于普通内存段（Anonymous Private），这些在 `0x33` 下会被 Dump。所以你可以看到指针地址。

**大页内容缺失**：`rte_malloc` 实际分配出的“肉”以及 `mempool` 存储对象的内存位于 共享大页（shared hugepage） 中。由于 `0x33` 默认不包含共享大页（Bit 5 为 0），内核在写 Core 文件时会跳过这些区域。

```bash
DPDK 的大页内存来源是：hugetlbfs + MAP_SHARED
在 `/proc/PID/maps` 里通常长这样：
7f8000000000-7f8200000000 rw-s 00000000 00:05 12345 /dev/hugepages/rtemap_0
```

**GDB 的表现**：当你用 GDB 查看这些被跳过的地址时，GDB 无法从 Core 文件中读到数据，默认会显示为 `0`。
```bash
core 文件里：没有这段内存;
gdb：知道“理论上这个地址区间存在”; 但 core 中没有数据; 就用 0 填充 / 显示为 0;

所以你看到的是：

(gdb) p *mbuf
$1 = { ... data = 0x0 ... }

这不是“被清零”，而是 “从未被 dump”。
```

### 影响
`coredump_filter` 默认是`0x33`，你无法调试：
```bash
无法查看：
- mbuf 内容
- mempool 状态
- rte_ring 中的数据
- rte_malloc 中的数据
```
    
因此，**默认 `coredump_filter`  对 DPDK 数据面几乎没用**。


## 解决方法
### `-in-memory`参数

```bash
- `--in-memory`
    Do not create any shared data structures and run entirely in memory. Implies `--no-shconf` and (if applicable) `--huge-unlink`.
```

![](attachments/Pasted%20image%2020260122174907.png)


DPDK Linux 中的大页映射, 支持以下：
```bash
1. 文件backed 大页映射 (默认方式) , 比如： /proc/PID/maps中看到 /dev/hugepages/FILE_PREFIXmap_xx, 比如：/dev/hugepages/kperfmap_13
2. 匿名 大页映射（--in-memory方式）
```

#### DPDK 匿名映射与 hugetlbfs 的区别


#### 存在`-in-memory`参数和不存在 `-in-memory`参数的效果对比

（1）不存在 `-in-memory`参数：



（2）存在 `-in-memory`参数：


### `echo 0xff > /proc/pid/coredump_filter`

设置 `/proc/PID/coredump_filter` 的参数；
程序崩溃 或 使用gcore工具 生成进程的core文件时，可以通过设定内核转储掩码来筛选需要dump的部分内存。
```bash
echo 0xff > coredump_filter 
不要通过vim更改，也不要写为 ​echo 000000ff > coredump_filter; 否则会失败；
```



#### coredump 文件爆炸问题
`0xff` 能解决调试可见性，但一定会引入“core 爆炸”的风险。
如果一个进程同时使用 DPDK + SPDK，并且你把 `coredump_filter` 设为 `0xff`：几乎必然会 dump 所有共享 `hugepage`, core 文件很容易是几十 GB 甚至上百 GB，磁盘不够用是非常现实的风险；

#### 分析

#### 验证
设置完了 `coredump_filter`之后，通过`kill -6 PID` 或者 `gcore pid` 可以产生`coredump`文件。
然后查看`coredump`文件的内容即可。

#### 解决1
```bash
echo 0x73 > /proc/PID/coredump_filter
```

动态打开：
```c
static void prepare_core_dump(void)
{
    int fd = open("/proc/self/coredump_filter", O_WRONLY);
    write(fd, "0x73\n", 5);
}

```
#### 解决2： 限制 Core 文件大小
```bash
ulimit -c 10485760  # 限制为 10GB

```

Core 文件会被截断。虽然不完整，但通常能保留栈帧信息（Backtrace），这对于解决大部分 Segment Fault 已经足够了。

#### 解决3：代码级精细化控制
如果你担心方案 1 导致磁盘爆炸，可以在代码里对“不重要”的大块内存池进行标记。 
比如：在 `rte_mempool_create` 之后，获取其虚拟地址，调用：
```bash
madvise(mempool_addr, mempool_size, MADV_DONTDUMP);

- 效果：
	即使你设置了 `0x53` 准备 Dump 大页，内核也会跳过标记了 `DONTDUMP` 的区域。 
	
- 调试思路：
	你可以只对包含重要业务逻辑的小内存池保留 Dump，对巨大的、只存原始报文的数据池标记 `DONTDUMP`。
```

# 参考
```bash
# 调试 dpdk 应用程序的coredump整理
https://blog.csdn.net/legend050709/article/details/108822350
```