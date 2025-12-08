```table-of-contents
```

# NUMA 不平衡问题（NUMA Imbalance）

## 跨numa访问的问题

在多 CPU（多 socket）服务器中，每个 CPU（或称为 **NUMA node**）都有自己**本地内存**。  
虽然所有内存都可以被访问，但访问性能不同：

|类型|含义|延迟|带宽|
|---|---|---|---|
|本地访问 (local access)|访问同节点上的内存|低|高|
|远程访问 (remote access)|访问其他节点上的内存|高|低|


## NUMA 不平衡问题介绍
当进程、线程或数据在 NUMA 节点之间的分布**不匹配**，导致某些节点频繁访问“远程内存”，性能下降，这就叫 **NUMA 不平衡（NUMA imbalance）**。

即：**线程跑在 CPU node A 的 core上，但它用的数据在 node B 的内存中**。

## 原因分析

|原因|说明|
|---|---|
|**内存和线程分配不在同一个 node 上**|比如线程在 node0 执行，但 `malloc` 分配的内存落在 node1|
|**线程/进程迁移**|操作系统调度线程跨 NUMA node 运行，但数据没迁移|
|**内核默认分配策略**|默认使用“first-touch”策略（第一次访问的线程所在节点分配内存），如果初始化阶段和计算阶段跑在不同核上，就出问题|
|**共享数据结构被多个 node 频繁访问**|某个共享队列或缓存区被远程访问过多，导致内存带宽压力不均|


## 后果分析


### QPI/UPI

|名称|全称|厂商|作用|
|---|---|---|---|
|**QPI**|**QuickPath Interconnect**|Intel（老架构，如 Xeon E5/E7）|处理器之间（socket间）及与 I/O hub 的高速互连总线|
|**UPI**|**Ultra Path Interconnect**|Intel（Skylake Xeon 及以后）|QPI 的继任者，带宽更高、延迟更低、协议更高效|

简单来说，**QPI / UPI 就是不同 CPU socket 之间互相通信的物理高速链路**。  
如果主板上有多个物理 CPU（多路系统），每个 CPU 有自己的本地内存控制器，**跨 CPU 访问内存**（即访问另一个 socket 的内存）就必须经过 QPI/UPI 传输。

|参数|QPI（Haswell/Broadwell）|UPI（Skylake 以后）|
|---|---|---|
|速率|6.4 – 9.6 GT/s|10.4 – 11.2 GT/s|
|每方向带宽（单链路）|~12.8 – 19.2 GB/s|~20 – 22 GB/s|
|延迟|约 50~70 ns|约 40~60 ns|
|本地 DRAM 延迟|约 70~80 ns|约 70~80 ns|
|跨 socket 访问延迟|约 120~150 ns|约 110~130 ns|



# 查看
## numactl

![](attachments/Pasted%20image%2020251011115931.png)

### `numactl -H`

![](attachments/Pasted%20image%2020251011120012.png)

### `numactl --show`


## `/proc/PID`查看

Linux 内核为每个进程（或线程）都维护了 NUMA 策略信息，主要包括：
- CPU 亲和性（在哪些 CPU 上运行）
- 内存绑定策略（在哪些 NUMA node 上分配内存）

|内容|文件路径|说明|
|---|---|---|
|CPU 亲和性|`/proc/<pid>/status` 或 `/proc/<pid>/task/<tid>/status`|字段 `Cpus_allowed_list`|
|NUMA 内存策略|`/proc/<pid>/numa_maps`|显示各虚拟内存区域的实际节点分布|


### 查看CPU 绑定

```bash
线程级别的CPU绑定：
cat /proc/<pid>/task/<tid>/status | grep allow

进程级别的CPU绑定：
cat /proc/<pid>/status | grep allow
```

![](attachments/Pasted%20image%2020251011120554.png)

进程级别的CPU绑定：同上

### 查看内存策略
```bash
线程级别的内存策略：
cat /proc/<pid>/task/<tid>/status | grep allow
cat /proc/<pid>/task/<tid>/numa_maps | head -n 20


进程级别的内存策略：
cat /proc/<pid>/numa_maps | head -n 20
cat /proc/<pid>/status | grep allow
```


```bash
# cat /proc/13365/task/13387/numa_maps
00400000 default file=/home/relay/xxx/rdma/kperf mapped=608 N0=608 kernelpagesize_kB=4
007ab000 default file=/home/relay/xxx/rdma/kperf anon=1 dirty=1 mapped=11 N0=10 N1=1 kernelpagesize_kB=4
007b6000 default file=/home/relay/xxx/rdma/kperf anon=8 dirty=8 mapped=39 N0=32 N1=7 kernelpagesize_kB=4
00d17000 default anon=247 dirty=247 N0=37 N1=210 kernelpagesize_kB=4
01cad000 default heap anon=266 dirty=266 N0=176 N1=90 kernelpagesize_kB=4
01db7000 default heap anon=6 dirty=6 N0=6 kernelpagesize_kB=4
01dbd000 default heap anon=1 dirty=1 N0=1 kernelpagesize_kB=4
01dbe000 default heap anon=22 dirty=22 N0=22 kernelpagesize_kB=4
01dd4000 default heap anon=1 dirty=1 N0=1 kernelpagesize_kB=4
01dd5000 default heap anon=16 dirty=16 N0=16 kernelpagesize_kB=4
01de5000 default heap anon=1 dirty=1 N0=1 kernelpagesize_kB=4
01de6000 default heap anon=5 dirty=5 N0=5 kernelpagesize_kB=4
01deb000 default heap anon=1 dirty=1 N0=1 kernelpagesize_kB=4
01dec000 default heap anon=16 dirty=16 N0=16 kernelpagesize_kB=4
...
100000000 default anon=143 dirty=143 N1=143 kernelpagesize_kB=4
100200000 default file=/dev/hugepages/kperfmap_0\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100400000 default file=/dev/hugepages/kperfmap_1\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100600000 default file=/dev/hugepages/kperfmap_2\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100800000 default file=/dev/hugepages/kperfmap_3\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100a00000 default file=/dev/hugepages/kperfmap_4\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100c00000 default file=/dev/hugepages/kperfmap_5\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
100e00000 default file=/dev/hugepages/kperfmap_6\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
101000000 default file=/dev/hugepages/kperfmap_7\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048
101200000 default file=/dev/hugepages/kperfmap_8\040(deleted) huge dirty=1 N0=1 kernelpagesize_kB=2048


```

**字段详细解释**：


|字段|含义|示例|
|---|---|---|
|**地址**|该内存段的起始虚拟地址|`00400000`|
|**策略名**|内存分配策略（NUMA policy）|`default`, `interleave`, `bind`, `preferred`|
|**file=xxx**|如果映射文件，显示文件路径|`file=/usr/bin/bash`|
|**anon=**|匿名页数量（不属于文件映射的页）|`anon=15`|
|**dirty=**|脏页数量|`dirty=5`|
|**mapped=**|映射的页数（一般用于文件映射）|`mapped=250`|
|**mapmax=**|最大映射次数（多个进程共享的页数上限）|`mapmax=10`|
|**active=**|活跃页数（近期被访问）|`active=5`|
|**writeback=**|在回写中的页数|`writeback=2`|
|**N0=, N1=, N2=...**|各个 NUMA 节点上实际分配的页数|`N0=200 N1=50`|
|**kernelpagesize_kB=**|页大小（用于大页或透明大页）|`kernelpagesize_kB=2048`|
|**swapcache=**|在 swap 缓存中的页数|`swapcache=1`|
|**migration=**|正在被迁移的页|`migration=1`|
|**file_mapped=**|文件页数|`file_mapped=1`|
|**stack**|表示这是进程栈区域|`stack`|
|**heap**|表示这是堆区域|`heap`|


（1）NUMA 节点页分布字段：
```bash
N0=200 N1=50

→ 表示这个 VMA 的页：
- 200 页在 NUMA node 0 上；
- 50 页在 node 1 上。

这个就说明 NUMA 不平衡。


```

（2）策略字段：

|策略|含义|来源|
|---|---|---|
|`default`|默认策略，由 `numactl` 或系统调度决定|默认|
|`interleave`|按页轮流分配到不同 NUMA 节点|`numactl --interleave`|
|`bind`|只允许在特定节点上分配|`numactl --membind` / `mbind()`|
|`preferred`|优先使用某节点，不可用则退而求其次|`set_mempolicy(MPOL_PREFERRED)`|


(3) 匿名内存与文件映射区：

- `anon` → 匿名内存（堆、栈、malloc()）
- `file` → 文件映射（mmap文件或共享库）
- `heap` → 程序堆（`brk()` 系统调用管理）
- `stack` → 栈空间

(4) `dirty` / `active`：
这些用于分析内存冷热分布。

|字段|含义|
|---|---|
|`dirty`|页被修改但尚未回写（匿名或文件页）|
|`active`|最近访问的页（内核活动列表页）|



## taskset 查看

```bash
查看 CPU 亲和性（跨线程）
taskset -cp <pid>
```

## numastat 查看
```bash
# numastat -h
numastat: invalid option -- 'h'
Usage: numastat [-c] [-m] [-n] [-p <PID>|<pattern>] [-s[<node>]] [-v] [-V] [-z] [ <PID>|<pattern>... ]
-c to minimize column widths
-m to show meminfo-like system-wide memory usage
-n to show the numastat statistics info
-p <PID>|<pattern> to show process info
-s[<node>] to sort data by total column or <node>
-v to make some reports more verbose
-V to show the numastat code version
-z to skip rows and columns of zeros
```

### 系统内存信息
```bash
# numastat -m

Per-node system memory usage (in MBs):
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
Token Node not in hash table.
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
MemTotal                63560.43        64503.32       128063.76
MemFree                   478.90          517.86          996.77
MemUsed                 63081.53        63985.46       127066.99
Active                   3691.70         2956.68         6648.38
Inactive                32018.84        41287.95        73306.79
Active(anon)               88.56         1023.66         1112.22
Inactive(anon)              5.04           34.93           39.98
Active(file)             3603.14         1933.02         5536.15
Inactive(file)          32013.80        41253.01        73266.81
Unevictable                 0.00            0.00            0.00
Mlocked                     0.00            0.00            0.00
Dirty                       0.58            0.14            0.71
Writeback                   0.00            0.00            0.00
FilePages               35641.75        43319.19        78960.94
Mapped                     95.51          155.43          250.95
AnonPages                  67.49          925.63          993.12
Shmem                      24.82          136.83          161.66
KernelStack                 9.65           15.57           25.22
PageTables                  4.96           13.48           18.45
NFS_Unstable                0.00            0.00            0.00
Bounce                      0.00            0.00            0.00
WritebackTmp                0.00            0.00            0.00
Slab                     1053.97         1202.70         2256.67
SReclaimable              844.93         1013.76         1858.69
SUnreclaim                209.04          188.94          397.98
AnonHugePages               0.00           28.00           28.00
HugePages_Total         16384.00        16384.00        32768.00
HugePages_Free          16384.00        16384.00        32768.00
HugePages_Surp              0.00            0.00            0.00
```

### 系统级别的查看
```bash
# numastat -n

Per-node numastat info (in MBs):
                          Node 0          Node 1           Total
                 --------------- --------------- ---------------
Numa_Hit           5148653259.43   4901805837.69  10050459097.12
Numa_Miss              242647.57    130273620.26    130516267.83
Numa_Foreign        130273620.26       242647.57    130516267.83
Interleave_Hit             96.98           95.62          192.61
Local_Node         5148639223.46   4901717071.98  10050356295.45
Other_Node             256683.53    130362385.96    130619069.49
```


### 进程级别的查看
```bash
numastat -p <pid>

# numastat -p 21984

Per-node process memory usage (in MBs) for PID 21984 (bash)
                           Node 0          Node 1           Total
                  --------------- --------------- ---------------
Huge                         0.00            0.00            0.00
Heap                         0.01            1.55            1.56
Stack                        0.00            0.02            0.03
Private                      3.25            0.22            3.47
----------------  --------------- --------------- ---------------
Total                        3.26            1.80            5.06
```


# 设置

|方法|说明|
|---|---|
|`numactl --cpunodebind=0 --membind=0`|让线程和内存绑定在同一 NUMA 节点上|
|`set_mempolicy()` / `mbind()`|程序内设置内存分配策略|
|`pthread_setaffinity_np()`|绑定线程到特定 CPU|
|`MAP_POPULATE` + `mlock()`|减少缺页带来的跨节点调页|
|NUMA-aware allocator (如 jemalloc、tcmalloc NUMA 支持)|自动在本地节点分配内存|

## numactl 工具：启动时设置CPU亲和性和内存策略
```bash
numactl --cpunodebind=0 --membind=0 ./app
```

## taskset工具：启动时或启动后设置CPU亲和性

## Linux C下的 set_mempolicy / mbind：设置程序的内存分配策略

## Linux C下的 pthread_setaffinity_np ： 设置线程的CPU亲和性

## Linux C下 map + MAP_POPULATE + mlock：减少缺页带来的跨节点调页

## 具有numa亲和性的三方库
比如：`jemalloc`、`tcmalloc` 的 `NUMA` 支持)

# 参考
```bash
```