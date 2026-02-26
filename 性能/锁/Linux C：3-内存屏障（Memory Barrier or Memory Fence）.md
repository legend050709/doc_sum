```table-of-contents
```

# 访问内存的发展
从硬件的角度来看，最初对内存的访问过程大概是这样：
```bash
CPU <--> 内存
```

相对于 CPU 的执行频率，直接访问内存速度较慢，于是引入存储速度比内存更快、但更贵、更小容量的 `CPU cache` ，用于缓存最近访问的内存数据，硬件拓扑结构如下图（不考虑多级 cache 情形）：

![](attachments/Pasted%20image%2020260118211636.png)

于是访问内存的过程变成了这样：
```bash
1. cahce 命中，直接从 cache 中读取
   CPU <--> CPU cache
2. cahce 未命中，从内存加载到 cache ，再从 cache 读取
   CPU <--> CPU cache <--> 内存
```

有了 `CPU cache`，速度上得到了很大提升，但人的追求永无止境，为了更快的速度，在 `CPU` 和 `cache` 之间， 又加入了更快更贵更小容量的 `store buffer` 存储。此时硬件拓扑如下：

![](attachments/Pasted%20image%2020260118211705.png)

在有 `store buffer` 缓存的情形下，`写操作数据`写入到 `store buffer` ，在需要的时候再写入到 `cache` 。


# 为什么需要内存屏障
## 范例
探讨内存屏障的问题基本都会从如下代码作为示例讲解： 

```c
//假设a和b初始化为0 ，CPU 0执行foo函数，CPU 1执行bar函数。
// 我们再进一步假设a变量, 在CPU 1的cache中，b在CPU 0 cache中;
// 执行的操作序列如下：

CPU0:
void foo(void) {
    a = 1;
    b = 1;
}


CPU1:
void bar(void) {
    while (b != 1); 
    assert (a == 1);
}
```

两方便原因可能导致`assert`失败:

### 编译层面(编译乱序)

如果前后代码无数据依赖关系，如果编译优化选项（-O2或者-O3)级别比较高会发生代码乱序，提升性能。

**注意：c/c++的关键字volatile只保证数据从寄存器回写到主存，没有内存屏障功能，也不保证原子性。**

### CPU层面(执行乱序)

现在的CPU处理器支持乱序执行（out-of-order)。如果刚刚接触这个概念，**十有八九会对乱序执行产生误解，**以为是因为单纯的cpu指令乱序执行导致了上面示例代码的assert失败，其实不然。

**乱序执行的本质**： 
乱序执行是说，给定一串执行指令，cpu为了提升执行效率，会找出没有数据依赖的指令，让他们并行执行，但是，执行执行结果写回寄存器是顺序的。哪怕是先被执行的指令，它的运算结果也是按照指令次序写回到最终的寄存器的。这个和很多程序员理解的乱序执行是有区别的。所以在单核处理器系统中，虽然CPU内部支持乱序执行，但是CPU会保证最终执行结果符合程序员的要求，这种情况下不需要使用内存屏障。这里从linux内核内存屏障函数定义也可以看出：

```c
#if defined(CONFIG_ARM_DMA_MEM_BUFFERABLE) || defined(CONFIG_SMP)
#define mb()        __arm_heavy_mb()
#define rmb()       dsb()
#define wmb()       __arm_heavy_mb(st)
#define dma_rmb()   dmb(osh)
#define dma_wmb()   dmb(oshst)
#else
#define mb()        barrier()                                                                                                                              
#define rmb()       barrier()
#define wmb()       barrier()
#define dma_rmb()   barrier()
#define dma_wmb()   barrier()
#endif
```

可以看到如果是非SMP系统，内存屏障都定义成barrier()函数，这个函数就是编译器层面的内存屏障函数，避免编译器乱序执行代码。


#### 为什么SMP系统会乱序

乱序的核心原因和现代cpu为了追求性能，实现的`memory model`有关，现代cpu内部有`store buffer/cache/invaliate queue`和`缓存一致性协议(MESI)`的方案导致乱序的可能。



# 分类
## 写屏障（Store Barrier / Write Fence）
## 读屏障（Load Barrier / Read Fence）
## 全屏障（Full Barrier / Memory Fence）
###  写法
Linux / DPDK / C11 写法对照如下。

```c
rte_mb();           // DPDK
smp_mb();           // Linux kernel
__sync_*            // GCC legacy atomic
atomic_thread_fence(seq_cst)
```

# load buffer 和 store buffer 以及  Invalid Queue

参考：[# 内存屏障今生之Store Buffer, Invalid Queue](https://blog.csdn.net/wll1228/article/details/107775976)
参考：[# 每个程序员都应该了解的内存知识](https://zhuanlan.zhihu.com/p/611133924)
![](attachments/Pasted%20image%2020260111223724.png)



# 编译器屏障 vs CPU 屏障
## 编译器屏障（Compiler Barrier）
## CPU 内存屏障

# 参考
```bash
# LINUX KERNEL MEMORY BARRIERS
https://www.kernel.org/doc/Documentation/memory-barriers.txt

# Volatile and memory barriers
https://jpbempel.github.io/2015/05/26/volatile-and-memory-barriers.html

# Memory Model and Synchronization Primitive - Part 1: Memory Barrier
https://www.alibabacloud.com/blog/memory-model-and-synchronization-primitive---part-1-memory-barrier_597460
# Memory Model and Synchronization Primitive - Part 2: Memory Model
https://www.alibabacloud.com/blog/memory-model-and-synchronization-primitive---part-2-memory-model_597461

### Volatile, Memory Barriers, and Load/Store Reordering
https://systemtbe.blogspot.com/2014/05/volatile-memory-barriers-and-loadstore.html

# 彻底搞懂内存屏障（上）
https://blog.csdn.net/GetNextWindow/article/details/126565892?spm=1001.2101.3001.10752
# 彻底搞懂内存屏障（下）
https://blog.csdn.net/GetNextWindow/article/details/126679252?spm=1001.2101.3001.10796

```