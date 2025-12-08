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

## 缓存
处理器中通常有多级缓存：
- **L1 Cache**（一级缓存） — 每个核心私有，容量很小（几十 KB），延迟极低
- **L2 Cache**（二级缓存） — 通常仍为每核私有，容量更大（几百 KB），延迟稍高
- **L3 Cache / LLC**（三级缓存） — 通常是 **所有核心共享** 的，容量最大（几 MB～几十 MB），延迟最高
- **LLC 就是最靠近内存之前的那一级缓存**，它是 CPU 访问内存数据前的最后一道高速缓存。

## cache miss的影响

cache失效次数，过多的实效会增加IO时间，增加耗时。


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

### 使用大页 (Huge Pages)

**思路**：大页减少 TLB miss，间接降低 cache miss 开销。

```bash
# 启用大页
echo 512 > /proc/sys/vm/nr_hugepages

```

然后用 `mmap()` + `MAP_HUGETLB` 分配大页内存。



# 参考
```bash

```