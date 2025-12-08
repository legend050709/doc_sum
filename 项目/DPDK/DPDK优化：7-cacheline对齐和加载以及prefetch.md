```table-of-contents
```
# CPU对于数据的访问
## 数据访问的原理

![](attachments/Pasted%20image%2020250923103437.png)

比如，有一个 `int array[100]` 的数组，当载入 `array[0]` 时，由于这个数组元素的大小在内存只占 4 字节，不足 64 字节，CPU 就会**顺序加载**数组元素到 `array[15]`，意味着 `array[0]~array[15]` 数组元素都会被缓存在 CPU Cache 中了，因此当下次访问这些数组元素时，会直接从 CPU Cache 读取，而不用再从内存中读取，大大提高了 CPU 读取数据的性能。

事实上，CPU 读取数据的时候，无论数据是否存放到 Cache 中，CPU 都是先访问 Cache，只有当 Cache 中找不到数据时，才会去访问内存，并把内存中的数据读入到 Cache 中，CPU 再从 CPU Cache 读取数据。

这样的访问机制，跟我们使用「内存作为硬盘的缓存」的逻辑是一样的，如果内存有缓存的数据，则直接返回，否则要访问龟速一般的硬盘。



## 数据访问的成本差距

![](attachments/Pasted%20image%2020250923103619.png)

![](attachments/Pasted%20image%2020250923103245.png)

![](attachments/Pasted%20image%2020250923103309.png)
![](attachments/Pasted%20image%2020250923103651.png)

![](attachments/Pasted%20image%2020250923103818.png)



# prefetch 预取
## cache prefetch
提示 CPU 把某个内存地址对应的数据，**提前加载到 L1/L2/L3 cache**。
这样当代码真正访问这个地址时，数据已经在 cache 里，不会触发昂贵的 DRAM 访问（通常要 100ns+）。

## 函数介绍
### DPDK中函数
```c
static inline void rte_prefetch0(const volatile void *p)
{
    asm volatile ("prefetcht0 %[p]" : : [p] "m" (*(const volatile char *)p));
}

static inline void rte_prefetch1(const volatile void *p)
{
    asm volatile ("prefetcht1 %[p]" : : [p] "m" (*(const volatile char *)p));
}

static inline void rte_prefetch2(const volatile void *p)
{
    asm volatile ("prefetcht2 %[p]" : : [p] "m" (*(const volatile char *)p));
}

static inline void rte_prefetch_non_temporal(const volatile void *p)
{
    asm volatile ("prefetchnta %[p]" : : [p] "m" (*(const volatile char *)p));
}
```

**rte_prefetch0(p)**：Locality=3（最强局部性） → 尽量拉到 L1 cache（最常用）。意味着数据 马上要用。

**rte_prefetch1(p)**： Locality=2 → 适合放到 L2 cache。数据 不久后就用（但不是马上）。

**rte_prefetch2(p)**：Locality=1 → 适合放到 L3 cache。数据 很久以后才用。
        
**rte_prefetch_non_temporal(p)**：非时间局部性预取（数据用一次就丢，不污染 cache）。

这些底层在 x86 上通常映射到 `__builtin_prefetch` 或 `prefetcht0`/`prefetcht1`/`prefetcht2` 指令。


#### 如何选择prefetch

|访问时间|建议 prefetch|
|---|---|
|立刻就用（下一次迭代就访问）|`prefetch0`|
|稍后用（隔几次迭代访问）|`prefetch1`|
|很久之后才用|`prefetch2`|

大多数情况下，**默认用 `prefetch0`** 就能有收益，


**推荐场景 1：遍历对象队列（rte_ring, mbuf 数组）**
```c
for (i = 0; i < nb_pkts; i++) {
    if (i + 1 < nb_pkts)
        rte_prefetch0(pkts[i+1]->buf_addr);
    process(pkts[i]);
}

```

为什么用 `prefetch0`？
- 下一次循环迭代马上要访问 `pkts[i+1]`。
- Prefetch0 最合适（到 L1 cache）。
- 这种情况在 **fast-path**（简单收发包）最常见。


**推荐场景 2：循环体很大（复杂包处理）**
```c
for (i = 0; i < nb_pkts; i++) {
    if (i + 2 < nb_pkts)
        rte_prefetch1(pkts[i+2]->buf_addr);  // 提前两步
    process_complex(pkts[i]); // 包解析、ACL 查表、NAT 等
}

```

为什么用 `prefetch1`？
- 当前包处理很复杂（几十到上百 cycles）。
- 提前 2 个迭代做预取，数据在处理完时就落到 L2，接着被拉到 L1。
- 避免 L1 过早被污染。



### Linux C中的函数
```c
__builtin_prefetch (const void *addr, int rw, int locality)
```

`__builtin_prefetch`是GCC编译器提供的一个内置函数，用于预取数据到CPU的缓存中，以便提高程序的执行效率。

其中，`addr`是一个指向要预取数据的地址的指针，`rw`是一个表示读写属性的整数，`locality`是一个表示预取数据的局部性的整数。
`__builtin_prefetch`的返回值是`void`类型。

#### 原理
**`__builtin_prefetch`只是告诉CPU预取数据到缓存中，而不会等待数据被加载到缓存中**。
`__builtin_prefetch`的内部原理是，它会向CPU发送一个预取数据的请求，然后CPU会将请求加入到预取队列中。当CPU空闲时，它会从预取队列中取出请求，并将请求的数据预取到缓存中。每次抓多少是有具体的CPU实现决定的，但是至少会抓32字节。



## prefetch 为什么可以提升性能

**prefetch 指令本身开销很小（几 cycles，通常 1 发射即可，不会阻塞流水线）**
它不会立刻用到数据，只是向 CPU 提示：“请提前把这个地址对应的 cache line 放到 cache 里”。
这样 CPU 可以在**当前指令执行时**，**异步地**去内存取数据。等你真正访问该数据时，它**大概率已经在 cache 里**，避免 stall。

所以，prefetch 的关键作用是 **把必然发生的长延迟，转化为“后台慢慢做”的并行工作**。

### 为什么不能到时候“直接访问”而不 prefetch？

- 如果等到真正访问时，数据还在 DRAM → CPU pipeline 必须停下来，啥也干不了，浪费 200 cycles。
有了 prefetch，你在处理当前数据时，就把**未来会访问的数据**放到 cache，  等需要访问它时 → **零等待 or 低等待**。
换句话说：**prefetch 是用小开销去掩盖大延迟**。



## prefetch 的开销和收益对比

**prefetch 指令**：只要几 cycles
 **一次 DRAM miss**：200+ cycles stall
当你批量处理时（如 32 个包一批），prefetch 可以显著降低平均延迟，提升吞吐率。

所以它提升性能的原因是：  
**prefetch 把不可避免的长延迟（200 cycles）变成提前发射的低成本 hint（几 cycles），并用当前的有用计算来掩盖它。**



## prefetch 使用场景
prefetch 最关键的应用场景：**批量循环访问数据结构时**。

### 循环遍历链表

**链表特点**
- 链表的访问模式 = 逐节点跳跃，下一个地址依赖当前节点。
- 访问模式不连续，局部性（cache 友好性）差。

**适合使用 prefetch 的情况**：
- 如果你只处理当前节点，没办法提前知道下一个节点 → **prefetch 无法提前发射**。
- 如果链表节点里存储了 `next` 指针，可以在访问当前节点时，**prefetch 下一个节点**。


**范例**：
```c
struct Node {
    struct Node *next;
    int value;
};

void traverse(struct Node *head) {
    struct Node *cur = head;
    while (cur) {
        if (cur->next) {
            rte_prefetch0(cur->next);  // 提前预取下一个节点
        }
        process(cur->value);
        cur = cur->next;
    }
}

```




### 循环遍历rte_ring

**特点**
- `rte_ring` 底层是一个 **环形数组**。
- 批量 dequeue/enqueue 时访问模式是**线性连续**的。

**为什么适合 prefetch**
- 环形数组访问模式和普通数组一样，顺序可预测，非常 cache-friendly。
- 可以预取未来几个元素。


**范例**：
```c
unsigned n = rte_ring_dequeue_burst(r, (void **)pkts, BURST, NULL);
for (unsigned i = 0; i < n; i++) {
    if (i + 1 < n) {
        // 显式预取下一个包的数据区
        rte_prefetch0(rte_pktmbuf_mtod(pkts[i+1], void *));
    }
    process_packet(pkts[i]); // 使用当前包
}

```

### 循环遍历数组

**特点**
- 顺序访问模式，连续存储，cache 友好性最好。
- CPU 硬件 prefetcher 本身就能预测并提前加载下一条 cache line。


**是否需要显式 prefetch**
- 对于**顺序访问**的数组：大多数情况下 **不需要手工 prefetch**，硬件预取足够聪明。
对于 顺序访问数组 这种简单的线性模式，它非常聪明：你访问 `arr[0]`、`arr[1]`，CPU 就会自动把 `arr[2]`、`arr[3]` 提前加载进 cache。所以在这种场景下，显式调用 `rte_prefetch0()` 反而可能多余。

- 对于**跨 stride（步长很大）的访问**，硬件 prefetch 可能不工作，此时手工 prefetch 有帮助。


**范例**：大步长访问数组
```c
#define STRIDE 128

void traverse_array(int *arr, int n) {
    for (int i = 0; i < n; i += STRIDE) {
        if (i + STRIDE < n) {
            rte_prefetch0(&arr[i + STRIDE]);
        }
        process(arr[i]);
    }
}

说明：大步长访问（稀疏访问）时，手工 prefetch 可以避免 cache miss。
```


### rte_ring 和 数组对比

|场景|硬件 prefetch|显式 prefetch|
|---|---|---|
|顺序访问数组|✅ 足够聪明，通常不需要|❌ 一般不推荐|
|大步长/稀疏访问数组|❌ 预测不到|✅ 可以显式 prefetch|
|rte_ring（访问指针数组 + mbuf 数据）|❌ 预测不到解引用后的内存|✅ 强烈推荐|



现代 CPU 内建了 **硬件预取器 (hardware prefetcher)**，它会监测你访问的内存地址模式。
对于 **顺序访问数组** 这种简单的线性模式，它非常聪明：你访问 `arr[0]`、`arr[1]`，CPU 就会自动把 `arr[2]`、`arr[3]` 提前加载进 cache。所以在这种场景下，**显式调用 `rte_prefetch0()` 反而可能多余**。


**rte_ring** 的本质确实是一个数组，但有两个特点：
**(1) 访问粒度不同**：
数组遍历通常是访问连续的 `int`、`struct` 元素，单个元素比较小。`rte_ring` 存的通常是 **指针 (`struct rte_mbuf *`)**，解引用后要访问 **mbuf 里的数据包头/数据**，而这些内存位置可能分散、不连续。硬件 prefetcher **只能预测数组下标地址的访问，不会跟踪指针解引用**。所以这里需要软件显式 prefetch，把 **mbuf 数据区** 拉进 cache。

**(2) 批量处理模式**：
DPDK 的收发包 API 通常一次处理 16/32/64 个包。这种 **批处理循环** 特别适合在处理当前包时，预取下一个包的数据。显式 prefetch 可以很好地掩盖 L3 miss → DRAM 的大延迟。





### 小结

|数据结构|是否适合 prefetch|典型场景|代码实现方式|
|---|---|---|---|
|链表 (list)|✅（有限）|顺序遍历长链表|在访问当前节点时预取 `next`|
|`rte_ring`|✅（推荐）|批量收发包|在处理当前元素时预取下一个元素|
|数组 (array)|⚠️ 一般不用|硬件预取已足够|仅在大步长/不规则访问时手工 prefetch|







# cacheline

## cpu的cache查看 

![](attachments/Pasted%20image%2020250923102154.png)

如上所示:
L3 cache 大小「28M」大概是L2 cache「1M」大小的 30倍。
L2 cache 大小「1M」大概是L1 cache「32k」大小的 30倍。

## cacheline对齐

## CPU 访问内存和 cache line 的关系
```bash
背景问题：

Linux C中一个结构体是cacheline对齐的，并且长度大于一个cacheline；
如果读取结构体中间的一个成员变量，此时cpu加载到cache中的是这个成员开始算起的cacheline，还是包含这个成员在中间的cacheline？
```

**（1）cache line 是 CPU cache 的最小传输单位**
- 大多数现代 CPU 的 cache line 大小是 64B。        
- 内存到 cache 的加载是以 **cache line 对齐** 的，即地址必须是 64B 边界对齐。

**（2）访问结构体某个成员变量时**
- CPU 会计算出该成员在内存中的 **物理地址**。        
- 然后找到该地址所在的 **cache line 起始地址**（按 64B 对齐）。
- CPU 会把这条完整的 64B cache line 从内存加载到 cache。

**（3）并不会只加载成员变量本身**
- 即使你访问的只是 `int`（4B），CPU 仍然加载整个 64B cache line。
- 这是为了保证后续访问相邻数据的时间局部性（temporal locality）。

因此：
CPU 加载的是“包含该成员的整个 cache line”，而不是“从该成员开始的 cache line”。Cache line 总是按固定大小（通常 64B）对齐。结构体跨多个 cache line 时，某些成员变量访问会导致加载不同的 cache line，甚至可能跨两个 cache line。

### 范例
```c
#include <stdint.h>
#include <stdio.h>

struct __attribute__((aligned(64))) Data {
    uint64_t a;    // 偏移 0
    uint64_t b;    // 偏移 8
    char buf[100]; // 偏移 16 ~ 115
    uint64_t c;    // 偏移 120
};

```

内存布局（两个 cache line）如下：
```bash
地址: 0x1000 -------------------------- 0x103F  (cache line 0)
        a(偏移0) b(偏移8) buf[0..47](偏移16..63)

地址: 0x1040 -------------------------- 0x107F  (cache line 1)
        buf[48..99](偏移64..115) c(偏移120)

```

如果访问 `d.buf[60]`：
- `buf[60]` 的地址大约在 `0x1040 + 12`
- CPU 会加载从 `0x1040` 开始的整条 cache line（64B），而不是只加载 `buf[60]` 或从 `buf[60]` 开始算的 64B。


# 参考
```bash

```