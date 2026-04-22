```table-of-contents
```
# 基本概念
## 临界区
当一个进程进入**临界区**时, 其他想进入临界区的进程只能**等待**, 也就是说, 临界区是**互斥**的, 而**临界区**内的资源, 则是**临界资源**

### 访问临界区
访问临界区有几种情况:
- **空闲让进**: 当没有进程在临界区时, 代表临界区是空闲的, 此时一个进程可以立即进入临界区, 访问临界资源
- **忙等**: 当已经有进程在临界区内时, 代表临界区有人正在访问, 所以其他想要进入临界区的进程必须等待
- **有限等待**: 当进程等待访问临界区时, 设置一定的超时时间, 避免一直等待
- **让权**: 当进程访问临界区发现需要等待时, 立刻释放资源, 避免夯住

## 信号量
那么在一个进程结束后, 怎么通知其他进程进入呢? 这就需要信号量

信号量分为四种类型:
- 整形信号量
- 记录型信号量
- AND 型信号量
- 信号量集

以整形信号量为例, 核心是设置一个用于表示资源数量的整形 A, 这个 A 有两种操作,
一个是减小(wait), 一个是增大(signal)；
在进入资源区时进行`wait`操作, 在退出时进行`signal`操作。


# 锁的分类
## 互斥锁
互斥量（Mutex）， 又称为互斥锁， 是一种用来保护临界区的特殊变量， 它可以处于锁定（locked） 状态， 也可以处于解锁（unlocked） 状态。

互斥锁是一种「独占锁」，比如当线程 A 加锁成功后，此时互斥锁已经被线程 A 独占了，只要线程 A 没有释放手中的锁，线程 B 加锁就会失败，于是就会释放 CPU 让给其他线程，**既然线程 B 释放掉了 CPU，自然线程 B 加锁的代码就会被阻塞**。

### 互斥锁加锁失败
**对于互斥锁加锁失败而阻塞的现象，是由操作系统内核实现的**。
当加锁失败时，内核会将线程置为「睡眠」状态，等到锁被释放后，内核会在合适的时机唤醒线程，当这个线程成功获取到锁后，于是就可以继续执行。如下图：

![](attachments/Pasted%20image%2020240206171741.png)

#### 失败的开销
所以，互斥锁加锁失败时，会从用户态陷入到内核态，让内核帮我们切换线程，虽然简化了使用锁的难度，但是存在一定的性能开销成本。

那这个开销成本是什么呢？会有**两次线程上下文切换的成本**：
- 当线程加锁失败时，内核会把线程的状态从「运行」状态设置为「睡眠」状态，然后把 CPU 切换给其他线程运行；
- 接着，当锁被释放时，之前「睡眠」状态的线程会变为「就绪」状态，然后内核会在合适的时间，把 CPU 切换给该线程运行。

#### 线程的上下文切换
线程的上下文切换的是什么？
当两个线程是属于同一个进程，**因为虚拟内存是共享的，所以在切换时，虚拟内存这些资源就保持不动，只需要切换线程的私有数据、寄存器等不共享的数据。**

上下切换的耗时有大佬统计过，大概在几十纳秒到几微秒之间，如果你锁住的代码执行时间比较短，那可能上下文切换的时间都比你锁住的代码执行时间还要长。

所以，**如果你能确定被锁住的代码执行时间很短，就不应该用互斥锁，而应该选用自旋锁，否则使用互斥锁。**

## 自旋锁
自旋锁是通过 CPU 提供的 `CAS` 函数（_Compare And Swap_），在「用户态」完成加锁和解锁操作，不会主动产生线程上下文切换，所以相比互斥锁来说，会快一些，开销也小一些。

一般加锁的过程，包含两个步骤：
- 第一步，查看锁的状态，如果锁是空闲的，则执行第二步；
- 第二步，将锁设置为当前线程持有；

CAS 函数就把这两个步骤合并成一条硬件级指令，形成**原子指令**，这样就保证了这两个步骤是不可分割的，要么一次性执行完两个步骤，要么两个步骤都不执行。

使用自旋锁的时候，当发生多线程竞争锁的情况，加锁失败的线程会「忙等待」，直到它拿到锁。这里的「忙等待」可以用 `while` 循环等待实现，不过最好是使用 CPU 提供的 `PAUSE` 指令来实现「忙等待」，因为可以减少循环等待时的耗电量。


### 自旋锁和互斥锁的区别
互斥锁和自旋锁对于加锁失败后的处理方式是不一样的：
自旋锁与互斥锁使用层面比较相似，但实现层面上完全不同：**当加锁失败时，互斥锁用「线程切换」来应对，自旋锁则用「忙等待」来应对**。

- **互斥锁**加锁失败后，线程会**释放 CPU** ，给其他线程；
- **自旋锁**加锁失败后，线程会**忙等待**，直到它拿到锁；


### 自旋锁可能存在的问题
- **试图递归地获得自旋锁必然会引起死锁**：  
在递归程序中使用自旋锁应遵守下列策略：递归程序决不能在持有自旋锁时调用它自己，也决不能在递归调用时试图获得相同的自旋锁。
    
- **过多占用cpu资源。**
如果不加限制，由于申请者一直在循环等待，因此自旋锁在锁定的时候,如果不成功,不会睡眠,会持续的尝试,单cpu的时候自旋锁会让其它process动不了。
因此，一般自旋锁实现会有一个参数限定最多持续尝试次数. 超出后, 自旋锁放弃当前`time slice`时间片， 等下一次机会。

由此可见，**自旋锁比较适用于锁使用者保持锁时间比较短的情况**。正是由于自旋锁使用者一般保持锁时间非常短，即 占用CPU不断尝试的时间比唤醒任务上下文切换的时间短，才适合使用自旋锁。


### 自旋的公平性问题
如果每一个新来的进程都在获取不到锁时, 进行自旋, 就会出现一个问题。
比如之前有3个进程在自旋后依旧没有获取到锁, 进入到了等待状态, 此时一个新的进程来了, 他在获取不到锁时, 进入了自旋状态, 而此时占用锁的进程退出了, 那么这一个新的进程就会最先获取到锁, 类似于插队的问题, 这样显然是不公平的。

### 注意事项
自旋锁是最比较简单的一种锁，一直自旋，利用 CPU 周期，直到锁可用。

**需要注意，在单核 CPU 上，需要抢占式的调度器（即不断通过时钟中断一个线程，运行其他线程）。否则，自旋锁在单 CPU 上无法使用，因为一个自旋的线程永远不会放弃 CPU。**

### 适应场景
自旋锁开销少，在多核系统下一般不会主动产生线程切换，适合异步、协程等在用户态切换请求的编程方式。
但如果被锁住的代码执行时间过长，自旋的线程会长时间占用 CPU 资源，所以自旋的时间和被锁住的代码执行的时间是成「正比」的关系，我们需要清楚的知道这一点。


## 读写锁
读写锁从字面意思我们也可以知道，它由「读锁」和「写锁」两部分构成，如果只读取共享资源用「读锁」加锁，如果要修改共享资源则用「写锁」加锁。

所以，**读写锁适用于能明确区分读操作和写操作的场景**。

### 工作原理
读写锁的工作原理是：

- 当「写锁」没有被线程持有时，多个线程能够并发地持有读锁，这大大提高了共享资源的访问效率，因为「读锁」是用于读取共享资源的场景，所以多个线程同时持有读锁也不会破坏共享资源的数据。
- 但是，一旦「写锁」被线程持有后，读线程的获取读锁的操作会被阻塞，而且其他写线程的获取写锁的操作也会被阻塞。

所以说，写锁是独占锁，因为任何时刻只能有一个线程持有写锁，类似互斥锁和自旋锁；而读锁是共享锁，因为读锁可以被多个线程同时持有。

### 使用场景
知道了读写锁的工作原理后，我们可以发现，**读写锁在读多写少的场景，能发挥出优势**。

### 公平性
根据实现的不同，读写锁可以分为「读优先锁」和「写优先锁」。

读优先锁对于读线程并发性更好，但也不是没有问题。我们试想一下，如果一直有读线程获取读锁，那么写线程将永远获取不到写锁，这就造成了写线程「饥饿」的现象。

写优先锁可以保证写线程不会饿死，但是如果一直有写线程获取写锁，读线程也会被「饿死」。

#### 读优先
期望的是，读锁能被更多的线程持有，以便提高读线程的并发性，它的工作方式是：当读线程 A 先持有了读锁，写线程 B 在获取写锁的时候，会被阻塞，并且在阻塞过程中，后续来的读线程 C 仍然可以成功获取读锁，最后直到读线程 A 和 C 释放读锁后，写线程 B 才可以成功获取写锁。

![](attachments/Pasted%20image%2020240206175949.png)

#### 写优先
写优先是优先服务写线程，其工作方式是：当读线程 A 先持有了读锁，写线程 B 在获取写锁的时候，会被阻塞，并且在阻塞过程中，后续来的读线程 C 获取读锁时会失败，于是读线程 C 将被阻塞在获取读锁的操作，这样只要读线程 A 释放读锁后，写线程 B 就可以成功获取写锁。
![](attachments/Pasted%20image%2020240206180032.png)

#### 公平读写锁
不管优先读锁还是写锁，对方可能会出现饿死问题，那么我们就不偏袒任何一方，搞个「公平读写锁」。

**公平读写锁比较简单的一种方式是：用队列把获取锁的线程排队，不管是写线程还是读线程都按照先进先出的原则加锁即可，这样读线程仍然可以并发，也不会出现「饥饿」的现象。**

## CAS
CAS（compare and swap）是为了解决多线程并行情况下使用锁造成性能损耗的一种机制。
**CAS操作包含三个操作数——内存位置（V）、预期原值（A）和新值(B)**。如果内存位置的值与预期原值相匹配，那么处理器会自动将该位置值更新为新值。否则，处理器不做任何操作。
CAS有效地说明了“我认为位置V应该包含值A；如果包含该值，则将B放到这个位置；否则，不要更改该位置，只告诉我这个位置现在的值即可。

### CAS机制执行流程
![](attachments/Pasted%20image%2020240206163941.png)

### CAS的问题
#### （1）ABA问题
并发编程 中的 ABA 问题：  
因为 CAS 需要在操作值的时候，检查某地址的内容有没有发生变化（和旧值进行比较），如果没有发生变化则更新为新的值。但是如果一个值原来是A，变成了 B，又变成了 A，那么使用 CAS 进行检查时会发现它的值没有发生变化，但是实际上却变化了。即：这个线程在操作的时候，其他线程也在进行操作。

##### 原因

CAS算法 实现一个重要前提需要取出内存中某时刻的数据，而在下时刻比较并替换，那么在这个==时间差==中出现数据的变化。

##### ABA的影响

Q: CAS的ABA问题，有什么影响，可以举个例子吗？ 
```bash
简单理解：
	一个线程将某个值从A变为B之后，又变为A；另外一个线程将A变为C，即使这个A是老的A，好像也没有问题吧。什么场景下会有问题呢。
```

**分析**：
你这个直觉只在**值本身就是全部语义**的情况下成立，但一旦**值背后代表的是某种状态、结构或时序**，ABA 就会埋雷，而且是那种**非常隐蔽、极难复现**的问题。
如果 A 只是一个普通变量（比如 int 值 100），那确实通常没问题——因为你只关心“现在是不是 A”，而不关心中间发生过什么。

**结论：**
- 如果变量是**纯值（stateless）**
- 不依赖“历史变化过程”
那么 ABA 基本无害。

##### 无锁链表栈（Lock-Free Stack）的ABA问题
假设有一个无锁栈，用链表实现，栈顶指针 `top` 指向节点 A：
```bash
初始状态：top → A → B → C
```

**（1）线程 1（慢线程）想执行 pop：准备 pop A**
```bash
old_top = A
next = B
// 准备 CAS(top, A, B), 此时线程1被挂起，线程2开始执行
```

**（2）线程 2（快线程）连续执行多次操作**
```bash
pop A → top = B → C （A被弹出） 
pop B → top = C （B被弹出，B的内存被释放或复用） 
push A → top = A → C （A被重新压入，但 A.next 现在指向 C！）

此时状态：top → A → C
```

**（3）线程 1 恢复执行：**
```bash
CAS(top, A, B) 
// top 当前是 A，和期望值 A 相同，CAS 成功！ 
// top 被设置为 B
```

**（4）问题：**

```bash
1》影响：
top 现在指向 B 但 B 已经被释放了！→ 野指针 / 内存错误 
节点 C 也从链表中凭空消失了！

2》整个流程：

初始:        T1读快照:       T2操作后:        T1 CAS成功后:
top          top             top              top
 ↓            ↓               ↓                ↓
[A]→[B]→[C]  记住A           [A]→[C]          [B] ← 野指针！
                              (B已释放)         [C] 丢失！
```

**（5）本质总结（为什么会出问题）**

ABA 的问题不在“值”，而在：这个值是否代表一个“时间点的状态”。

在无锁数据结构中：
- 指针 A ≠ 同一个 A
- A 可能已经：
    - 被删除
    - 被复用（malloc/free）
    - 被重新插入

```bash
A == A（值相等）  
但 != 同一个“历史状态”
```

##### lock-free 的 内存池的ABA问题

比如 lock-free + 内存池：
```bash
1. 线程1读取指针 A
2. 线程2：
    - free(A)
    - malloc 新对象，刚好复用地址 A
3. 线程1 CAS 成功
```

结果：
- 指针“没变”（还是 A）
- 但对象已经是**完全不同的数据**

##### 解决ABA问题

**(1) 添加版本号**
ABA问题的解决思路其实也很简单，就是使用版本号。在变量前面追加上版本号，每次变量更新的时候把版本号加1，那么A→B→A就会变成1A→2B→3A了。

```bash
struct {
    pointer ptr;
    int version;
}

CAS 比较：(ptr == old_ptr && version == old_version)

每次修改 version++
```

**(2)Hazard Pointer（更高级）**

核心思想：
- 标记“哪些节点正在被使用”
- 防止被提前 free

**(3)Epoch-based reclamation（RCU 类似）**

延迟回收：
- 确保没有线程再访问旧数据


##### 添加版本号实现ABA的范例
核心思路其实很简单：**把“指针 + 版本号”当成一个整体，一次 CAS 同时比较它们**。

在 Linux C 里实现带版本号的 CAS，核心思路是将**指针 + 版本号**打包进一个 128-bit 的结构中，然后借助 x86-64 的 `CMPXCHG16B` 指令一次性原子地比较和替换两者。在 x86_64 上可以用 **128-bit CAS（cmpxchg16b）**，GCC 提供了原子内建：
```c
__atomic_compare_exchange()
```

完整的范例：
```c
// gcc -O2 -mcx16 -pthread -o aba_demo aba_demo.c
// -mcx16 启用 CMPXCHG16B 指令（x86-64 上 128-bit 原子操作）

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>
#include <pthread.h>

/* ── 1. 核心数据结构 ── */

typedef struct Node {
    int value;
    struct Node *next;
} Node;

// 指针 + 版本号打包成 16 字节，必须 16 字节对齐
// 这样 __atomic_compare_exchange 才能映射到 CMPXCHG16B
typedef struct {
    Node     *ptr;
    uintptr_t version;
} __attribute__((aligned(16))) StampedPtr;

/* ── 2. 128-bit CAS 封装 ── */

static bool stamped_cas(volatile StampedPtr *target,
                        StampedPtr expected,
                        StampedPtr desired)
{
    // __atomic_compare_exchange 在 16 字节对齐结构上
    // 会被编译器翻译成 CMPXCHG16B 一条原子指令
    return __atomic_compare_exchange(
        (StampedPtr *)target,
        &expected,
        &desired,
        /*weak=*/false,
        __ATOMIC_SEQ_CST,
        __ATOMIC_SEQ_CST
    );
}

// 原子读取整个 StampedPtr（两个 64-bit 字段不能分开读）
static StampedPtr stamped_load(volatile StampedPtr *src)
{
    StampedPtr result;
    __atomic_load((StampedPtr *)src, &result, __ATOMIC_SEQ_CST);
    return result;
}

/* ── 3. 无锁栈实现 ── */

volatile StampedPtr top = { .ptr = NULL, .version = 0 };

void push(int value)
{
    Node *node = malloc(sizeof(Node));
    node->value = value;

    StampedPtr old_top, new_top;
    do {
        old_top = stamped_load(&top);
        node->next = old_top.ptr;

        new_top.ptr     = node;
        new_top.version = old_top.version + 1; // 每次操作版本号 +1
    } while (!stamped_cas(&top, old_top, new_top));
    // CAS 失败说明 top 已被其他线程修改（ptr 或 version 变了），重试
}

Node *pop(void)
{
    StampedPtr old_top, new_top;
    do {
        old_top = stamped_load(&top);
        if (old_top.ptr == NULL) return NULL; // 栈空

        new_top.ptr     = old_top.ptr->next;
        new_top.version = old_top.version + 1;
    } while (!stamped_cas(&top, old_top, new_top));

    return old_top.ptr; // 调用者负责 free
}

/* ── 4. 演示 ABA 场景被正确拦截 ── */

void *thread_aba_trigger(void *arg)
{
    // 模拟"快线程"：pop A → pop B → push A
    // 这会让地址 A 重新出现在栈顶，但 version 已经不同
    Node *a = pop(); // version: 1→2
    Node *b = pop(); // version: 2→3

    // 重新压入 A（version 变为 4）
    // 注意：a->next 现在可能指向旧的节点
    a->next = top.ptr; // 手动设置以模拟 ABA
    StampedPtr cur = stamped_load(&top);
    StampedPtr new_top = { .ptr = a, .version = cur.version + 1 };
    __atomic_store((StampedPtr *)&top, &new_top, __ATOMIC_SEQ_CST);

    printf("[快线程] pop A(v=%lu) → pop B(v=%lu) → push A，version 现在 = %lu\n",
           (unsigned long)1, (unsigned long)2, (unsigned long)new_top.version);

    free(b);
    return NULL;
}

int main(void)
{
    // 初始化栈: C → B → A（A 在栈顶）
    push(3); // C, version=1
    push(2); // B, version=2
    push(1); // A, version=3 (top)

    printf("初始栈顶: ptr=%p, version=%lu\n",
           (void *)top.ptr, (unsigned long)top.version);

    // 慢线程：记录此刻的 top（version=3, ptr=A）
    StampedPtr slow_snapshot = stamped_load(&top);
    printf("[慢线程] 快照: ptr=%p, version=%lu\n",
           (void *)slow_snapshot.ptr, (unsigned long)slow_snapshot.version);

    // 快线程运行：A 被 pop、B 被 pop、A 再 push → version 变为 4
    pthread_t t;
    pthread_create(&t, NULL, thread_aba_trigger, NULL);
    pthread_join(t, NULL);

    // 慢线程尝试 CAS：用旧快照（version=3）去替换
    // ptr 虽然相同（都是 A），但 version 不同（3 vs 4）→ CAS 失败
    StampedPtr desired = { .ptr = slow_snapshot.ptr->next, .version = slow_snapshot.version + 1 };
    bool ok = stamped_cas(&top, slow_snapshot, desired);

    printf("[慢线程] CAS 结果: %s（version %lu → %lu 不匹配，ABA 被拦截！）\n",
           ok ? "成功（有问题）" : "失败（正确）",
           (unsigned long)slow_snapshot.version,
           (unsigned long)top.version);

    // 清理
    Node *n;
    while ((n = pop()) != NULL) free(n);
    return 0;
}
```

**为什么必须 `-mcx16`？**
`__atomic_compare_exchange` 作用在 16 字节结构上时，编译器会生成 `CMPXCHG16B` 指令。这条指令在 x86-64 上默认不启用（历史兼容原因），`-mcx16` 告诉编译器目标 CPU 支持它。


**如果是用uint64_t可不可以，低48bit存储指针，高16位存储版本号？**
完全可以，这就是经典的 指针标记（Tagged Pointer） 技术。x86-64 用户空间指针实际只用低 48 位，高 16 位天然为零，可以借来存版本号。

||高16位标记（uint64）|StampedPtr（__int128）|
|---|---|---|
|CAS 指令|`LOCK CMPXCHG`（64-bit）|`CMPXCHG16B`（128-bit）|
|编译参数|无需特殊|`-mcx16`|
|版本号位数|**16 bit**（0–65535，会绕回）|64 bit（几乎不绕回）|
|解引用前|必须 mask 掉高16位|直接用|
|5级分页（LA57）风险|⚠️ 只剩 7 位可用|无影响|

```c
// gcc -O2 -pthread -o tagged_ptr tagged_ptr.c
// 无需 -mcx16，标准 64-bit CAS 即可

#include <stdio.h>
#include <stdlib.h>
#include <stdint.h>
#include <stdbool.h>
#include <stdatomic.h>
#include <pthread.h>

/* ── 1. 位域定义 ──────────────────────────────
 *
 *  63     48 47                              0
 *  ┌────────┬────────────────────────────────┐
 *  │version │         pointer (48 bit)       │
 *  └────────┴────────────────────────────────┘
 */
#define PTR_MASK  0x0000FFFFFFFFFFFFULL   // 低 48 位
#define VER_SHIFT 48
#define VER_MASK  0xFFFFULL               // 高 16 位

typedef struct Node {
    int value;
    struct Node *next;
} Node;

// _Atomic uint64_t：保证 lock cmpxchg 原子操作
typedef _Atomic uint64_t TaggedPtr;

/* ── 2. 打包 / 解包 ── */

static inline uint64_t pack(Node *ptr, uint16_t version)
{
    // 确保指针真的只用了 48 位（debug 用）
    // assert(((uint64_t)ptr & ~PTR_MASK) == 0);
    return ((uint64_t)version << VER_SHIFT)
         | ((uint64_t)(uintptr_t)ptr & PTR_MASK);
}

static inline Node *unpack_ptr(uint64_t tagged)
{
    // ⚠️ 解引用前必须 mask，否则高16位非零会导致段错误
    return (Node *)(uintptr_t)(tagged & PTR_MASK);
}

static inline uint16_t unpack_ver(uint64_t tagged)
{
    return (uint16_t)(tagged >> VER_SHIFT);
}

/* ── 3. Tagged CAS ── */

static bool tagged_cas(TaggedPtr *target, uint64_t expected, uint64_t desired)
{
    // 编译成：LOCK CMPXCHG qword ptr [target], desired
    return atomic_compare_exchange_strong_explicit(
        target, &expected, desired,
        memory_order_seq_cst,
        memory_order_seq_cst
    );
}

/* ── 4. 无锁栈 ── */

static TaggedPtr top = 0; // ptr=NULL, version=0

void push(int value)
{
    Node *node = malloc(sizeof(Node));
    node->value = value;

    uint64_t old_val, new_val;
    do {
        old_val = atomic_load_explicit(&top, memory_order_acquire);
        node->next = unpack_ptr(old_val);                  // 必须 mask
        new_val = pack(node, unpack_ver(old_val) + 1);     // 版本 +1
    } while (!tagged_cas(&top, old_val, new_val));
}

Node *pop(void)
{
    uint64_t old_val, new_val;
    do {
        old_val = atomic_load_explicit(&top, memory_order_acquire);
        Node *cur = unpack_ptr(old_val);
        if (!cur) return NULL;

        new_val = pack(cur->next, unpack_ver(old_val) + 1);
    } while (!tagged_cas(&top, old_val, new_val));

    return unpack_ptr(old_val);
}

/* ── 5. 验证版本号绕回风险 ── */

void check_wraparound(void)
{
    uint16_t ver = 0xFFFF;
    printf("\n[版本号绕回演示]\n");
    printf("  当前版本: 0x%04X (%u)\n", ver, ver);
    printf("  +1 之后:  0x%04X (%u)  ← 绕回到 0！\n",
           (uint16_t)(ver + 1), (uint16_t)(ver + 1));
    printf("  高频场景下 65536 次操作即绕回一轮，需评估 ABA 窗口\n");
}

/* ── 6. 主程序演示 ── */

int main(void)
{
    push(30); push(20); push(10);

    printf("=== Tagged Pointer 无锁栈 ===\n");

    // 显示初始状态的 64-bit 打包值
    uint64_t raw = atomic_load(&top);
    printf("raw uint64  = 0x%016lX\n", raw);
    printf("  高16位 version = %u\n", unpack_ver(raw));
    printf("  低48位 ptr     = %p\n\n", (void *)unpack_ptr(raw));

    // 弹出全部元素
    Node *n;
    while ((n = pop()) != NULL) {
        printf("pop → %d\n", n->value);
        free(n);
    }

    check_wraparound();

    return 0;
}
```



#### （2）循环时间长开销大
自旋CAS如果长时间不成功，会给CPU带来非常大的执行开销

#### （3）只能保证一个共享变量的原子操作

当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，但是对多个共享变量操作时，循环CAS就无法保证操作的原子性，这个时候就可以用锁。

## RCU

## 小结
**互斥锁**
开发过程中，最常见的就是互斥锁的了，互斥锁加锁失败时，会用「线程切换」来应对，当加锁失败的线程再次加锁成功后的这一过程，会有两次线程上下文切换的成本，性能损耗比较大。

**自旋锁**
如果我们明确知道被锁住的代码的执行时间很短，那我们应该选择开销比较小的自旋锁，因为自旋锁加锁失败时，并不会主动产生线程切换，而是一直忙等待，直到获取到锁，那么如果被锁住的代码执行时间很短，那这个忙等待的时间相对应也很短。

**读写锁**
如果能区分读操作和写操作的场景，那读写锁就更合适了，它允许多个读线程可以同时持有读锁，提高了读的并发性。根据偏袒读方还是写方，可以分为读优先锁和写优先锁，读优先锁并发性很强，但是写线程会被饿死，而写优先锁会优先服务写线程，读线程也可能会被饿死，那为了避免饥饿的问题，于是就有了公平读写锁，它是用队列把请求锁的线程排队，并保证先入先出的原则来对线程加锁，这样便保证了某种线程不会被饿死，通用性也更好点。

> 互斥锁和自旋锁都是最基本的锁，读写锁可以根据场景来选择这两种锁其中的一个进行实现。



# 乐观锁 & 悲观锁
乐观锁其实主要就是一种思想，因为乐观锁的操作过程中其实没有没有任何锁的参与。
乐观锁只是和悲观锁相对，严格的说乐观锁不能称之为锁。

独占锁：是一种悲观锁；独占锁，会导致其它所有需要锁的线程挂起，等待持有锁的线程释放锁。  
乐观锁：每次不加锁，假设没有冲突去完成某项操作，如果因为冲突失败就重试，直到成功为止。

## 悲观锁

前面提到的互斥锁、自旋锁、读写锁，都是属于悲观锁。
悲观锁做事比较悲观，它认为**多线程同时修改共享资源的概率比较高，于是很容易出现冲突，所以访问共享资源前，先要上锁**。  

## 乐观锁
相反的，如果多线程同时修改共享资源的概率比较低，就可以采用乐观锁。
乐观锁做事比较乐观，它假定冲突的概率很低，它的工作方式是：**先修改完共享资源，再验证这段时间内有没有发生冲突，如果没有其他线程在修改资源，那么操作完成，如果发现有其他线程已经修改过这个资源，就放弃本次操作**。
放弃后如何重试，这跟业务场景息息相关，虽然重试的成本很高，但是冲突的概率足够低的话，还是可以接受的。
可见，乐观锁的心态是，不管三七二十一，先改了资源再说。另外，你会发现**乐观锁全程并没有加锁，所以它也叫无锁编程**。

### 使用场景
**在线文档**：
我们都知道在线文档可以同时多人编辑的，如果使用了悲观锁，那么只要有一个用户正在编辑文档，此时其他用户就无法打开相同的文档了，这用户体验当然不好了。

那实现多人同时编辑，实际上是用了乐观锁，它允许多个用户打开同一个文档进行编辑，编辑完提交之后才验证修改的内容是否有冲突。

怎么样才算发生冲突？这里举个例子，比如用户 A 先在浏览器编辑文档，之后用户 B 在浏览器也打开了相同的文档进行编辑，但是用户 B 比用户 A 提交早，这一过程用户 A 是不知道的，当 A 提交修改完的内容时，那么 A 和 B 之间并行修改的地方就会发生冲突。

服务端要怎么验证是否冲突了呢？通常方案如下：
- 由于发生冲突的概率比较低，所以先让用户编辑文档，但是浏览器在下载文档时会记录下服务端返回的文档版本号；
- 当用户提交修改时，发给服务端的请求会带上原始文档版本号，服务器收到后将它与当前版本号进行比较，如果版本号不一致则提交失败，如果版本号一致则修改成功，然后服务端版本号更新到最新的版本号。

## 其他

### AS 不是乐观锁吗，为什么基于 CAS 实现的自旋锁是悲观锁？
乐观锁是先修改同步资源，再验证有没有发生冲突。
悲观锁是修改共享数据前，都要先加锁，防止竞争。

CAS 是乐观锁没错，但是 CAS 和自旋锁不同之处，自旋锁基于 CAS 加了while 或者睡眠 CPU 的操作而产生自旋的效果，加锁失败会忙等待直到拿到锁，自旋锁是要需要事先拿到锁才能修改数据的，所以算悲观锁。

## 小结
实际上，我们常见的 SVN 和 Git 也是用了乐观锁的思想，先让用户编辑代码，然后提交的时候，通过版本号来判断是否产生了冲突，发生了冲突的地方，需要我们自己修改后，再重新提交。

乐观锁虽然去除了加锁解锁的操作，但是一旦发生冲突，重试的成本非常高，所以**只有在冲突概率非常低（即写少多读场景），且加锁成本非常高的场景时，才考虑使用乐观锁。**



# 死锁（deadlock）
**死锁是指两个或两个以上的线程在执行过程中，由于竞争资源或者由于彼此通信而造成的一种阻塞的现象，若无外力作用，它们都将无法推进下去。**此时称系统处于死锁状态或系统产生了死锁，这些永远在互相等待的进程称为死锁进程。


## 死锁的发生条件
虽然进程在运行过程中，可能发生死锁，但死锁的发生也必须具备一定的条件，**死锁的发生必须具备以下四个必要条件**：

- **互斥条件**：指进程对所分配到的资源进行排它性使用，即在一段时间内某资源只由一个进程占用。如果此时还有其它进程请求资源，则请求者只能等待，直至占有资源的进程用毕释放。
- **请求和保持条件**：指进程已经保持至少一个资源，但又提出了新的资源请求，而该资源已被其它进程占有，此时请求进程阻塞，但又对自己已获得的其它资源保持不放。
- **不剥夺条件**：指进程已获得的资源，在未使用完之前，不能被剥夺，只能在使用完时由自己释放。
- **环路等待条件**：指在发生死锁时，必然存在一个进程——资源的环形链，即进程集合{P0，P1，P2，···，Pn}中的P0正在等待一个P1占用的资源；P1正在等待P2占用的资源，……，Pn正在等待已被P0占用的资源。

# 锁的公平性
锁从公平性上通常会分为公平锁和非公平锁。
主要取决于在锁获取的过程中，先进行锁获取的线程是否比后续的线程更先获得锁，如果是则就是公平锁：多个线程按照获取锁的顺序依次获得锁；否则就是非公平性。

## 锁饥饿(hungry)
锁饥饿是指因为大量线程都同时进行获取锁，某些线程可能在获取锁的过程中一直失败，从而长时间获取不到锁。


# 参考
```bash

# 并发编程系列：谈谈锁的实现机制
https://www.cyub.vip/2022/07/28/%E8%B0%88%E8%B0%88%E9%94%81%E7%9A%84%E5%AE%9E%E7%8E%B0%E6%9C%BA%E5%88%B6/

# [Linux同步机制 - 基本概念(死锁,活锁,饿死,优先级反转,护航现象)](https://www.cnblogs.com/linuxbug/p/4840148.html)

# 什么是悲观锁、乐观锁
https://xiaolincoding.com/os/4_process/pessim_and_optimi_lock.html
```