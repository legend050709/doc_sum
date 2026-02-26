```table-of-contents
```

# 原子操作

## 背景

我们知道无论是何种情况，只要有共享的地方，就离不开同步，也就是concurrency。对共享资源的安全访问，在不使用锁、同步原语的情况下，只能依赖于硬件支持的原子性操作，离开原子操作的保证，无锁编程(lock-free programming)将变得不可能。

我们发现原子性操作可以简单划分为两部分： 

``` bash
1. 原子性读写(atomic read and write)：本例中的原子load(读)、原子store(写) 
2. 原子性交换(Atomic Read-Modify-Write -- RMW)：本例中的 compare_exchange_weak、compare_exchange_strong 
```

## 定义
（==All done or nothing==）。
原子操作可认为是一个不可分的操作；要么发生，要么没发生，我们看不到任何执行的中间过程，不存在部分结果(partial effects)。可以想象的到，原子操作要保证要么全部发生，要么全部没发生，这样原子操作绝对不是一个廉价的消耗低的指令，相反，原子操作是一个较为昂贵的指令。

注：在底层，原子操作是一些特殊的硬件指令，若是硬件不支持是没有办法实现多线程的原子操作的。



## 范例
![](attachments/Pasted%20image%2020250412151120.png)
X 可能会等于 1 吗？答案是会的。
![](attachments/Pasted%20image%2020250412151127.png)
对于这个操作，整个过程并不是原子的，即另一个线程可能看到操作执行的中间状态。
这是典型的 load-modify-store 操作，我们期望这整个过程都要一起完成，即整个操作是原子的。

在 X86 平台上，请看 Intel® 64 and IA-32 Architectures Developer's Manual: Vol. 3A
![](attachments/Pasted%20image%2020250412151216.png)

对齐的 Load 是原子的，对齐的 Store 也是原子的，但 load-modify-store 这整个过程不是原子的。
即使是一个原子变量也是如此：
```cpp
atomic<int> x(0);

x = x + 1;
// `x = x + 1` 这条语句整体不是原子的，但读 x, 写 x 都是原子的。
// 可能得不到你期望的结果
```
C++11 标准中引入了原子操作。
```cpp
#include <atomic>
std::atomic<int> x(0); // NOT atomic<int> x = 0;
```
![](attachments/Pasted%20image%2020250412151326.png)

现在，语言保证了 x 最终会保证 x = 2。
```c
int x = 0x10101010;
--------
// thread 1 
x = 0x11110000;

--------
// thread 2
x = 0x00001111;


// 原子操作保证了 x 只可能是 0x11110000 或者是 0x00001111
// 而不可能是               0x11111111 或者是 0x00000000
```
原子操作本意就是如此，要么都做，要么全都没做，没有中间状态。但现实的原子操作又掺杂了内存序的内容，这两者也确实是密不可分的

## 指令重排

在提内存模型之前，必须要说下指令重排，之后才能理解内存模型是为了什么。再次向这篇文章[2](https://zhuanlan.zhihu.com/p/611868395#ref_2)表示感谢，写的很好，从中受益良多。

## 编译器的指令重排
```cpp
int msg = 0; // assume operation is atomic
int ready = 0;
------------------------------------------
// thread 1
void foo() {
  msg = 10086;
  ready = 1;
}
------------------------------------------
// thread 2
void bar() {
  if (ready == 1) {
    // output may be 0, unexpected
    std::cout << msg;
  }
}
```
以上代码的输出并不一定是 `10086`，也有可能是 `0`。

`thread 1` 单线程执行 `foo()` 先给 a 赋值，还是先给 b 赋值，最终的结果都是正确的, 符合重排的最基本的 `principle`, 所以在不加任何限制的情况下,  compiler 生成的汇编可以自由地将 ready 先于 msg 赋值.

但是, 引入(`thread 2`)多线程之后, 由于业务逻辑上需要有数据的依赖关系, ready 之后才能输出 msg 但是 thread 2并不知道 ready 和 msg 有什么顺序关系, 只负责自己的 `bar()` 执行即可, 在 thread 2看起来自己的执行也是正确的, 但这个时候运行起来就有”bug”了.

可以使用 `asm volatile ("" ::: "memory")` 阻止编译器层面的代码重排。

## 防止重排
在 x86 C 中，可以使用 `asm volatile ("mfence" ::: "memory")` 或者 `__sync_synchronize()`，既阻止编译器，也阻止 CPU 进行重排。

## CPU的指令重排（乱序执行）



# 原子操作API
## 内核的原子操作API
## gcc内置的原子操作API
### `__sync_*` 系列函数
```bash

```
### `__atomic_*`系列函数
```bash

```



## 其他
### DPDK中的原子操作
[DPDK中的原子操作](https://doc.dpdk.org/api/rte__stdatomic_8h_source.html)

# 区分
## 原子变量和 volatile
## 
# 参考
```bash
（1）并发编程：
# Java并发编程—可见性、原子性和有序性问题：并发编程Bug的源头
https://zhangquan.me/2023/05/22/java-bing-fa-bian-cheng-ke-jian-xing-yuan-zi-xing-he-you-xu-xing-wen-ti-bing-fa-bian-cheng-bug-de-yuan-tou/
【总结的很不错；+++++】

# Java 并发编程实战、极客时间
https://time.geekbang.org/column/intro/100023901?tab=catalog

# 并发编程三大特性——原子性、可见性、有序性
https://www.cnblogs.com/yeyang/p/13576636.html


```