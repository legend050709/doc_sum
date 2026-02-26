```table-of-contents
```

# 程序局部性
一个编写良好的计算机程序通常具有程序的局部性，它更倾向于引用最近引用过的数据项，或者这个数据周围的数据——前者是时间局部性，后者是空间局部性。

即：程序在一段时间内访问的数据通常具有局部性，局部性分为“时间局部性”和“空间局部性”；
时间局部性是指当前被访问的数据随后有可能访问到；
空间局部性是指当前访问地址附近的地址可能随后被访问。

## 应用场景
现代操作系统的设计，从硬件到操作系统再到应用程序都利用了程序的局部性原理：
（1）硬件层，通过cache来缓存刚刚使用过的指令或者数据，来提交对内存的访问效率。
（2）在操作系统级别，操作系统利用主存来缓存刚刚访问过的磁盘块；
（3）在应用层，web浏览器将最近引用过的文档放在磁盘上，大量的web服务器将最近访问的文档放在前端磁盘上，这些缓存能够满足很多请求而不需要服务器的干预。



# 处理器的存储体系
计算机体系的储存层次从内到外依次是寄存器、cache（从一级、二级到三级）、主存、磁盘、远程文件系统；
从内到外，访问速度依次降低，存储容量依次增大。这个层次关系，可以用下面这张图来表示：

![](attachments/Pasted%20image%2020250918165714.png)

程序在执行过程中，数据最先在磁盘上，然后被取到内存之中，最后如果经过cache（也可以不经过cache）被CPU使用。如果数据不再cache之中，需要CPU到主存中存取数据，那么这就是cache miss，这将带来相当大的时间开销。

越往下，存储介质速度越快，造价越高容量也越小；越往上，存储介质速度越慢，造价越低但容量也越大。
现代操作系统巧妙的利用cache，以最小的代价获得了最大的性能。但是，注意这里的但是，**要想获得极致性能是有前提的，那就是程序员写的程序必须具有良好的局部性，充分利用缓存**。

## Cache出现的背景
所有计算机程序的本质，是指令的执行过程，从根本上说，也就**是处理器的寄存器(Register)到内存之间的数据交互过程**。
这个交互，无非两个步骤:

- 读内存，也就是加载操作(load), 从内存读到寄存器
- 写内存，就就是存储操作(store)，从寄存器写入到内存

我们知道，CPU和内存之间存在速度差异。
程序的指令，如果需要从CPU Register里面取数据，CPU只需要0 cycles(CPU周期);
 如果需要直接访问内存，则需要几百个cycles。(另外，如果能从高速缓存取到目标数据，需要4-75个cycles）

 一个非常之快的CPU(处理器)，却要和一个更慢的RAM(内存)交换数据。毫无疑问，如果直接交换数据，那么CPU会因为RAM太慢，拖慢自己的速度。
为了解决两者之间数据交换数据的速度差异，计算机科学家们了就引入了高速缓存(Cache)：让CPU和RAM之间使用多层级的缓存，避免两者直接交换数据，而是从临近的缓存区交换数据。


## cache的效果

我们把经常用到的数据放到cache中存储，CPU访问内存时首先查找cache，如果能找到，也就是命中，那么就赚到了，直接返回即可，找不到再去查找内存并更新cache。

我们可以看到，**有了cache，CPU不再直接与内存打交道了**。

![](attachments/Pasted%20image%2020251218150438.png)

但cache的快速读写能力是有代价的，代价就是Money，造价不菲，**因此我们不能把内存完全替换成cache的SRAM，那样的计算机你我都是买不起的**。
因此cache的容量不会很大，但由于程序局部性原理，**因此很小的cache也能有很高的命中率**，从而带来性能的极大提升，有个词叫**四两拨千斤**，用到cache这里再合适不过。

## 多级缓存(cache)
处理器中通常有多级缓存：
- **L1 Cache**（一级缓存） — 每个核心私有，容量很小（几十 KB），延迟极低
- **L2 Cache**（二级缓存） — 通常仍为每核私有，容量更大（几百 KB），延迟稍高
- **L3 Cache / LLC**（三级缓存） — 通常是 **所有核心共享** 的，容量最大（几 MB～几十 MB），延迟最高
- **LLC 就是最靠近内存之前的那一级缓存**，它是 CPU 访问内存数据前的最后一道高速缓存。

# TLB
## 背景

CPU执行机器指令时，指令指示CPU从内存地址A中取出数据，然后CPU执行机器指令时下发命令：“给我从地址A中取出数据”，尽管真的能从地址A中取出数据，但这个地址A不是真的，不是真的，不是真的。
因为这个地址A属于虚拟内存，也就是那个“假的地址空间”，现代CPU内部有一个叫做MMU的模块将这假的地址A转换为真的地址B，将地址A转换为真实的地址B之后才是本文之前讲述的关于cache的那一部分。

![](attachments/Pasted%20image%2020251218173403.png)

## 访问内存(load/store)的整体流程
CPU给出内存地址，此后该地址被转为真正的物理内存地址，接下来查L1 cache，L1 cache不命中查L2 cache，L2 cache不命中查L3 cache，L3 cache不能命中查内存。

以一次 `load` 指令为例（不考虑乱序细节）：
```bash
虚拟地址
   ↓
dTLB 查找（VA → PA）
   ↓
L1 D-cache 查找（PA + cache tag）
   ↓
L2 / L3 
   ↓
内存
```


### TLB和cache的协同工作

当你的一行代码想要读取一个变量时，CPU 的操作顺序通常如下：

1. **查找路标 (TLB)：** CPU 拿的是**虚拟地址**，首先问 TLB：“这个虚拟地址对应的**物理地址**在哪？”
    
    - **TLB Hit：** 瞬间拿到物理地址。
        
    - **TLB Miss：** 完蛋，MMU 必须去内存里翻页表（Page Table Walk）。这就像是在巨大的字典里查一个字在哪一页。
        
2. **查找东西 (Cache)：** 拿到物理地址后，CPU 问 Cache：“这个物理地址里的**数据**在你这吗？”
    
    - **Cache Hit：** 立即拿到数据，执行计算。
        
    - **Cache Miss：** 完蛋，Cache 必须去内存（RAM）里把数据搬回来。


## TLB miss

### perf查看TLB miss
```bash
# Various CPU data TLB statistics for the specified command:
perf stat -e dTLB-load,dTLB-load-misses,dTLB-store,dTLB-store-misses -p PID

perf stat -e dTLB-load,dTLB-load-misses,dTLB-store,dTLB-store-misses COMMAND

```

说明：
```bash
dTLB-loads:  需要进行地址翻译的 load 次数（读次数）。其实每次读内存都要查 dTLB。

dTLB-load-misses：读内存时，dTLB 中 没有该虚拟页的映射。
				 如果TLB miss，就需要访问页表（涉及到多次内存访问！）

```


## 为什么 DPDK 性能对 TLB Miss 更敏感？

在常规程序中，Cache Miss 通常是性能瓶颈。但在 **DPDK 高性能网络转发** 场景下，情况有所不同：

- **内存跨度极大：** 网络缓冲区（mbuf）可能均匀分布在数 GB 的内存空间中。如果使用 4KB 小页，每一页只能覆盖很小的范围，TLB 很快就会被填满并发生频繁替换（这叫 TLB Thrashing）。
    
- **TLB Miss 具有“阻塞性”：** 很多时候，Cache Miss 可以通过“预取”或者指令乱序执行来掩盖延迟，但 **TLB Miss 通常会导致流水线彻底停顿**，因为没有物理地址，CPU 甚至不知道下一步该去哪里预取数据。
    
- **大页的威力：** 使用 2MB 大页时，一个 TLB 条目覆盖的范围是原来的 512 倍。对于同样的内存工作集，TLB Miss 率会大幅下降，从而显著降低长尾时延（Tail Latency）。

# cache miss

## cache miss的影响

cache失效次数，过多的实效会增加IO时间，增加耗时。

### perf 查看
```bash

# Various basic CPU statistics, system wide, for 10 seconds:
perf stat -e cycles,instructions,cache-references,cache-misses,bus-cycles -a sleep 10

# Various CPU level 1 data cache statistics for the specified command:
perf stat -e L1-dcache-loads,L1-dcache-load-misses,L1-dcache-stores command

# Various CPU last level cache statistics for the specified command:
perf stat -e LLC-loads,LLC-load-misses,LLC-stores,LLC-prefetches command
```


## cache miss 和TLB miss的对比

简单来说，**Cache Miss** 是找不到“**东西**”（数据或指令），而 **TLB Miss** 是找不到“**路标**”（虚拟地址到物理地址的映射）。
它们发生在内存访问的不同阶段，对性能的影响也不同。


|**特性**|**TLB Miss (路标缺失)**|**Cache Miss (货物缺失)**|
|---|---|---|
|**全称**|Translation Lookaside Buffer Miss|CPU Cache Miss (L1/L2/L3)|
|**缓存内容**|**页表项 (PTE)**：记录虚拟地址 $\rightarrow$ 物理地址的映射关系。|**数据或指令**：内存中真实存放的 0/1 内容。|
|**功能定位**|属于 **MMU (内存管理单元)**，解决“在哪找”的问题。|属于 **存储层次结构**，解决“拿得快”的问题。|
|**发生时机**|**地址翻译阶段**：CPU 还没拿到物理地址，无法去缓存或内存取数。|**取数阶段**：已经知道物理地址，但在 Cache 层级中没找到。|
|**惩罚开销**|**极高**。通常需要 Page Walk（多级页表查询），可能触发多次内存访问。|**高**。取决于缺失级别（L1/L2/L3），L3 Miss 需要访问 RAM。|
|**优化手段**|**使用大页 (Hugepages)**、增加 TLB 条目、Page Walk Cache。|**增强局部性**、硬件预取 (Prefetch)、增加 Cache 容量。|

### 大页和cache miss以及TLB miss的关系

- **大页的意义主要在于减少 TLB Miss**： 一个 TLB 条目如果对应 4KB，那么 2MB 的内存需要 512 个条目；如果使用 2MB 的大页，只需要 **1 个**条目。这大大提高了 TLB 的利用率，减少了“找路标”的时间。
    
- **Cache Miss 与页大小关系较小**： 无论页多大，CPU Cache（L1/L2/L3）缓存的都是具体的 64 字节数据块（Cache Line）。


#### 使用大页 (Huge Pages)

**思路**：大页减少 TLB miss，间接降低 cache miss 开销。

```bash
# 启用大页
echo 512 > /proc/sys/vm/nr_hugepages

```

然后用 `mmap()` + `MAP_HUGETLB` 分配大页内存。


### perf工具来整体查看cache miss 和TLB miss

我们可以分两步进行排查：**全貌观察**和**代码定点定位**。

#### 第一步：全貌观察（使用 `perf stat`）

这个命令可以让你一眼看出程序在运行期间，总共丢了多少次“路标”（TLB）和“货物”（Cache）。

```bash
# -e 指定我们要观察的事件
# cache-misses: 总的缓存缺失
# dTLB-load-misses: 数据 TLB 查找缺失
# iTLB-load-misses: 指令 TLB 查找缺失
# cycles, instructions: 用来计算 IPC (每周期指令数)
sudo perf stat -e cache-misses,cache-references,dTLB-load-misses,dTLB-loads,iTLB-load-misses,cycles,instructions -p <你的进程PID>
```

- **dTLB-load-miss rate**：计算 `dTLB-load-misses / dTLB-loads`。如果这个比例超过 **1%**，说明 TLB 缺失已经非常严重了。
    
- **Cache-miss rate**：计算 `cache-misses / cache-references`。在 DPDK 转发中，如果这个值很高，通常意味着你的 mbuf 访问跨度太大。
    
- **IPC (Instructions Per Cycle)**：如果 IPC 低于 **1.0**，说明 CPU 大部分时间都在“等内存”，而不是在“算报文”。

#### 第二步：精准定位（使用 `perf record`）

如果你发现 `dTLB-load-misses` 确实很高，你想知道**到底是哪一行代码**触发的，可以用这个组合：

```bash
(1) 采样录制:
# 录制 dTLB 缺失事件，-g 表示记录调用栈
sudo perf record -e dTLB-load-misses -g -p <你的进程PID> -- sleep 10


(2) 生成报告：
sudo perf report

在报告界面中，你可以看到哪些函数贡献了最多的 TLB Miss。你可以直接按回车下钻（Annotate）到汇编级别，看看到底是哪个内存读写指令卡住了。

```

## cache 优化技巧

### 提高空间局部性（Spatial Locality）

**思路**：让访问的数据在内存上尽量连续，这样 CPU 预取能生效，一次取一整 cache line（通常 64 字节）。

#### 数据合并
有两个数据A和B，访问的时候经常是一起访问的，总是会先访问A再访问B。如果A、B是两个长度很大的数组，这样`A[i]`和`B[i]`就距离很远，那么可能`A[i]`和`B[i]`无法同时存在`cache`之中。为了增加程序访问的局部性，需要将`A[i]`和`B[i]`尽量存放在一起。为此，我们可以定义一个结构体，包含A和B的元素各一个。



#### 交换循环位置

C语言中，对于二维数组，同一行的数据是相邻的，同一列的数据是不相邻的。如果在循环中，让依次访问的数据尽量处在内存中相邻的位置，那么程序的局部性将会得到很大的提高。
```c
#define N 1024
int arr[N][N];

// 低命中率：按列访问（内存不连续）
for (int j = 0; j < N; j++)
    for (int i = 0; i < N; i++)
        arr[i][j]++;

// 高命中率：按行访问（内存连续）
for (int i = 0; i < N; i++)
    for (int j = 0; j < N; j++)
        arr[i][j]++;

```


比如对一维数组来说，访问了地址x上的元素，那么以后访问地址`x+1`、`x+2`上元素的可能性就比较高；
现在访问的数据，在不久之后再次被访问的可能性也比较高。
处理器通过在内存和核心之间增加缓存以利用局部性增强程序性能，这样可以用远低于缓存的价格换取接近缓存的速度。
```c
(1) 范例一：
for i=1…n  
for j=1…n  
for k=1…n  
c[i,j] += a[i,k]*b[k,j]  


（2）范例二：
for i=1…n  
for k=1…n  
for j=1…n  
c[i,j] += a[i,k]*b[k,j]


分析：
在cache层面上，代码二要比代码一效率高很多，因为：
代码2的b[k,j]是按行访问的，所以存在良好的空间局部性，cache line被充分利用。
代码1中，b [k，j]由列访问。 由于行的存储矩阵，因此对于每个缓存行加载，只有一个元素用于遍历。
```

#### 循环合并
**定义**：  
将原本**多个遍历相同数据范围的循环**合并成**一个循环**，使得**数据在 cache 中的驻留时间更长**，从而减少 cache miss。


在很多情况下，我们可能使用两个独立的循环来访问数组a和c。由于数组很大，在第二个循环访问数组中元素的时候，第一个循环取进cache中的数据已经被替换出去，从而导致cache失效。如此情况下，可以将两个循环合并在一起。合并以后，每个数组元组在同一个循环体中被访问了两次，从而提高了程序的局部性。

```c
// 假设 N 很大
for (int i = 0; i < N; i++) {
    a[i] = b[i] + 1;      // 第一次访问 b[i]
}

for (int i = 0; i < N; i++) {
    c[i] = b[i] * 2;      // 第二次访问 b[i]
}


- 缓存行为：
    - 第一次循环时把 `b[i]` 从内存加载到缓存，用完后很快被逐出。
    - 第二次循环又要重新把 `b[i]` 读入缓存。
- 导致 cache miss 增多，内存带宽浪费。
```

```c
for (int i = 0; i < N; i++) {
    int tmp = b[i];        // 一次从内存加载
    a[i] = tmp + 1;
    c[i] = tmp * 2;         // 重用缓存中的数据
}


缓存行为：
- `b[i]` 只加载一次，利用缓存行的空间局部性。
- 充分利用缓存中的 `b[i]`，减少内存访问次数 → 降低 cache miss 率。
```

#### AoS 和 SoA

**AoS（Array of Structures）**  
多个结构体对象依次排在内存中，每个结构体包含所有字段。
    
**SoA（Structure of Arrays）**  
把每个字段单独放到一个数组里，同一类数据连续存放。


**(1)AoS（传统写法）**
```c
#include <stdint.h>

struct Data {
    uint64_t id;
    uint64_t timestamp;
    float    value;
};

struct Data arr[100000];


- 内存布局： `id ts val | id ts val | id ts val ...`
- 如果只想访问 `value`，CPU 仍会把 `id`、`timestamp` 一起加载到 cache（浪费带宽）
- 空间局部性差，cache miss 高
```

**(2)SoA（优化写法）**
```c
#include <stdint.h>

struct DataSoA {
    uint64_t id[100000];
    uint64_t timestamp[100000];
    float    value[100000];
};

struct DataSoA data;


- 内存布局： `id id id ...`、`timestamp timestamp ...`、`value value ...`
- 如果只遍历 `value[i]`，会线性加载一整块连续的 `float` 数据
- cache 友好性极好，miss 明显降低
```


**(3)对比**

|项目|AoS|SoA|
|---|---|---|
|内存访问模式|混合字段，跨 cache line|单字段连续线性|
|有效带宽利用率|低|高|
|cache miss 率|高|低|
|适合场景|同时访问结构体多个字段|只访问部分字段、多次遍历|


**(4)实战建议**

- 当**频繁批量访问结构体的单个字段**（如处理大量点/包/记录）时，优先考虑 SoA。
- 如果同时访问所有字段，AoS 也可以，但要注意控制结构体大小（避免跨 cache line）。
- 混合方式：常访问字段放 SoA，冷字段放 AoS 或单独结构体指针。




### 提高时间局部性（Temporal Locality）

**思路**：重复使用相同数据，让它在 cache 中停留更久，避免频繁换出。
优化要点：多次使用同一数据块而不是每次都换新数据。

```c
int a[1024];
// 低命中率：每次都访问不同元素
for (int i = 0; i < 1024; i++)
    use(a[i]);

// 高命中率：多次访问同一小块数据
for (int repeat = 0; repeat < 100; repeat++)
    for (int i = 0; i < 16; i++)
        use(a[i]);

```

### 控制结构体/对象大小，避免跨 cache line

**思路**：让常用字段在同一 cache line，提高命中率，减少 false sharing。

现代 CPU 的 **cache 是以 cache line 为基本单位（通常是 64 字节）** 来加载和管理数据的。
**跨越 cache line** 会导致：
- 一次访问要加载两个 cache line（浪费带宽与 cache 空间）
- 如果多个线程访问不同字段，容易导致 **伪共享（false sharing）**，产生额外 cache coherence 开销


（1）结构体过大跨 cache line
```c
#include <stdint.h>

struct Data {
    uint64_t id;        // 8
    uint64_t timestamp; // 8
    char name[50];      // 50
    // 合计 66 字节，> 64
};

struct Data arr[1000];


- `sizeof(struct Data)` = 72（编译器会做 8 字节对齐补齐）
- 数组的每个元素会跨 cache line，相邻两个元素几乎总是占用两个 cache line
- 顺序访问时，每加载一个元素，可能浪费部分 cache 带宽
```


（2）控制结构体大小 ≤ 64B（避免跨行）

```c
#include <stdint.h>

// 精简并对齐到 cache line 大小（64B）
struct __attribute__((aligned(64))) Data {
    uint64_t id;        // 8
    uint64_t timestamp; // 8
    char name[48];      // 48
    // 总共 64 字节，恰好一个 cache line
};

struct Data arr[1000];


优点：
- 每个对象占 1 条 cache line，不会跨行
- 顺序访问时完全利用带宽
- 多线程访问 `arr[i]` 不会互相干扰，减少伪共享
```

**经验建议**：
理想情况是 **结构体大小 = cache line 大小的整数倍（常见 64B）**。
如果结构体字段非常多，**分离冷热字段**：常访问字段留在主结构体，冷门字段放到单独数组或指针引用。



### Cache 对齐与预取

**思路**：让数据按 cache line 对齐，并主动预取后续数据，减少 cache miss。


```c
#include <emmintrin.h>  // _mm_prefetch

int arr[1024] __attribute__((aligned(64)));

for (int i = 0; i < 1024; i += 16) {
    _mm_prefetch((char*)&arr[i+16], _MM_HINT_T0);  // 预取未来数据
    arr[i]++;
}

```

优化要点：使用 `__attribute__((aligned(64)))` 对齐到 64 字节，必要时用 `_mm_prefetch`。

### 减少随机访问（避免 cache 抖动）

**思路**：避免访问模式高度随机化，否则 cache 无法预测和复用。

```c
// 差：随机访问
int idx[N];
for (int i = 0; i < N; i++)
    use(arr[idx[i]]);

// 优：排序后顺序访问
qsort(idx, N, sizeof(int), cmp);
for (int i = 0; i < N; i++)
    use(arr[idx[i]]);


```


# 建议
## 关掉 inline

### 问题
`inline` 导致查看堆栈的时候，发现代码的调用关系逻辑不太明确。

### 方法

```bash
关掉 inline 看一次：

CFLAGS += -fno-inline -fno-omit-frame-pointer -g
```



# 参考
```bash

```