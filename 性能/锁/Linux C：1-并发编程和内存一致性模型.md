```table-of-contents
```

# 并发编程
## 背景1：对于共享数据的访问和并行编程的矛盾

现代计算机，即使很小的智能机亦或者平板电脑，都是一个多核(多CPU)处理设备，如何充分利用多核CPU资源，以达到单机性能的极大化成为我们码农进行软件开发的痛点和难点。在多核服务器中，采用多进程或多线程来并行处理任务，俨然成为了大家性能调优的标准解决方案。**多进程(多线程)的并行编程**方式，必然要面对**共享数据**的访问问题，如何**并发、高效、安全地访问共享数据**资源，成为并行编程的一个重点和难点。

### 共享数据的同步源语和并行编程(parallel programming)的对立
传统的共享数据访问方式是采用同步原语(临界区、锁、条件变量等)来达到共享数据的安全访问，然而，同步恰恰和并行编程是对立的，很容易成为并行程序中的瓶颈。

一方面，有些同步原语是操作系统的内核对象，调用该原语会带来昂贵的上下文切换(用户态切换到内核态)代价，同时，内核对象是一个比较有限的资源。
另一方面，同步杜绝了并行操作，一个线程在访问共享数据的时候，其他的多个线程必须在排队空闲等待，同时，同步可扩展性很弱，随着并行线程的增加，很容易成为程序的一个瓶颈，甚至出现，服务性能吞吐量并没随CPU核数增加或并发线程的增加呈现线性增长，相反出现下降的情况。

### 思路
人们开始研究**对共享数据进行并发访问的数据结构和算法**，通常有以下几方面：

```bash
1. Transactional memory --- 事务性内存 
2. Fine-grained algorithms --- 细粒度(锁)算法 
3. Lock-free data structures --- 无锁数据结构
```

#### 事务内存（Transactional memory）

事务内存（Transactional memory）TM是一个软件技术，简化了并发程序的编写。 TM借鉴了在数据库社区中首先建立和发展起来的概念， 基本的想法是要申明一个代码区域作为一个事务。一个事务（transaction ） 执行并原子地提交所有结果到内存（如果事务成功），或中止并取消所有的结果（如果事务失败）。 TM的关键是提供原子性（Atomicity），一致性（Consistency ）和隔离性（Isolation ）这些要素。 
事务可以安全地并行执行，以取代现有的痛苦和容易犯错误(下面几点)的技术，如锁和信号量。
```bash
4. 因为忘记使用锁而导致条件竞争(race condition) 
5. 因为不正确的加锁顺序而导致死锁(deadlock) 
6. 因为未被捕捉的异常而造成程序崩溃(corruption) 
7. 因为错误地忽略了通知，造成线程无法正常唤醒(lost wakeup)
```

事务内存 还有一个潜在的性能优势。 我们知道锁是悲观的（pessimistic ），并假设上锁的线程将写入数据，因此，其他线程的进展被阻塞。 然而访问锁定值的两个事务可以并行地进行，且回滚只发生在当事务之一写入数据的时候。但是，目前还没有嵌入式的事务内存，比较难和传统代码集成，需要软件做出比较大的变化，同时，软件TM性能开销极大，2-10倍的速度下降是常见的，这也限制了软件TM的广泛使用

#### 细粒度锁
细粒度(锁)算法是一种基于另类的同步方法的算法，它通常基于“轻量级的”原子性原语(比如自旋锁)，而不是基于系统提供的昂贵消耗的同步原语。细粒度(锁)算法适用于任何锁持有时间少于将一个线程阻塞和唤醒所需要的时间的场合，由于锁粒度极小，在此类原语之上构建的数据结构，可以并行读取，甚至并发写入。Linux 4.4以前的内核就是采用`_spin_lock`自旋锁这种细粒度锁算法来安全访问共享的`listen socket`，在并发连接相对轻量的情况下，其性能和无锁性能相媲美。然而，在高并发连接的场景下，细粒度(锁)算法就会成为并发程序的瓶颈所在。

#### 无锁数据结构

无锁数据结构，为解决在高并发场景下，细粒度锁无法避免的性能瓶颈，**将共享数据放入无锁的数据结构中，采用原子修改的方式来访问共享数据**。 
目前，常见的无锁数据结构主要有：无锁队列(lock free queue)、无锁容器(b+tree、list、hashmap等)。

我们知道无论是何种情况，只要有共享的地方，就离不开同步，也就是concurrency。对共享资源的安全访问，在不使用锁、同步原语的情况下，只能依赖于硬件支持的原子性操作，离开原子操作的保证，无锁编程(lock-free programming)将变得不可能。


## 背景2：并发编程的背景

这些年，我们的 CPU、内存、I/O 设备都在不断迭代，不断朝着更快的方向努力。但是，在这个快速发展的过程中，有一个**核心矛盾一直存在，就是这三者的速度差异**。CPU 和内存的速度差异可以形象地描述为：CPU 是天上一天，内存是地上一年（假设 CPU 执行一条普通指令需要一天，那么 CPU 读写内存得等待一年的时间）。内存和 I/O 设备的速度差异就更大了，内存是天上一天，I/O 设备是地上十年。

程序里大部分语句都要访问内存，有些还要访问 I/O，根据**木桶理论**（一只水桶能装多少水取决于它最短的那块木板），程序整体的性能取决于最慢的操作——读写 I/O 设备，也就是说单方面提高 CPU 性能是无效的。

### 解决思路
为了合理利用 CPU 的高性能，平衡这三者的速度差异，计算机体系机构、操作系统、编译程序都做出了贡献，主要体现为：

(1)CPU 增加了缓存，以均衡与内存的速度差异；
(2)操作系统增加了进程、线程，以分时复用 CPU，进而均衡 CPU 与 I/O 设备的速度差异；
(3) 编译程序优化指令执行次序，使得缓存能够得到更加合理地利用。


### 并发编程的问题
#### 缓存导致的可见性问题

**在单核时代**：所有的线程都是在一颗 CPU 上执行，CPU 缓存与内存的数据一致性容易解决。因为所有线程都是操作同一个 CPU 的缓存，一个线程对缓存的写，对另外一个线程来说一定是可见的。例如在下面的图中，线程 A 和线程 B 都是操作同一个 CPU 里面的缓存，所以线程 A 更新了变量 V 的值，那么线程 B 之后再访问变量 V，得到的一定是 V 的最新值（线程 A 写过的值）。

![](attachments/Pasted%20image%2020240912105837.png)

**可见性的定义**：
一个线程对共享变量的修改，另外一个线程能够立刻看到，我们称为可见性。


**多核时代**：每颗 CPU 都有自己的缓存，这时 CPU 缓存与内存的数据一致性就没那么容易解决了，当多个线程在不同的 CPU 上执行时，这些线程操作的是不同的 CPU 缓存。比如下图中，线程 A 操作的是 CPU-1 上的缓存，而线程 B 操作的是 CPU-2 上的缓存，很明显，这个时候线程 A 对变量 V 的操作对于线程 B 而言就不具备可见性了。这个就属于硬件程序员给软件程序员挖的“坑”。

![](attachments/Pasted%20image%2020240912105938.png)


**验证可见性的范例：**
下面我们再用一段代码来验证一下多核场景下的可见性问题。
下面的代码，每执行一次 add10K() 方法，都会循环 10000 次 count+=1 操作。在 calc() 方法中我们创建了两个线程，每个线程调用一次 add10K() 方法，我们来想一想执行 calc() 方法得到的结果应该是多少呢？
```java
public class Test {
	private int count = 0;

	private void add10K() {
		int idx = 0;
		while(idx++ < 10000) {
			count += 1;
		}
	}

	public static int calc() throws Exception {
		final Test test = new Test();
		Thread th1 = new Thread(()->{
			test.add10K();
		});
		Thread th2 = new Thread(()->{
			test.add10K();
		});

		th1.start();
		th2.start();
		th1.join();
		th2.join();
		return test.count;
	}

	public static void main(String[] args) throws Exception {
		long c = calc();
		System.out.println(c);
	}
}
```

直觉告诉我们应该是 20000，因为在单线程里调用两次 add10K() 方法，count 的值就是 20000，但实际上 calc() 的执行结果是个 10000 到 20000 之间的随机数。为什么呢？

我们假设线程 A 和线程 B 同时开始执行，那么第一次都会将 count=0 读到各自的 CPU 缓存里，执行完 count+=1 之后，各自 CPU 缓存里的值都是 1，同时写入内存后，我们会发现内存中是 1，而不是我们期望的 2。之后由于各自的 CPU 缓存里都有了 count 的值，两个线程都是基于 CPU 缓存里的 count 值来计算，所以导致最终 count 的值都是小于 20000 的。这就是缓存的可见性问题。

循环 10000 次 count+=1 操作如果改为循环 1 亿次，你会发现效果更明显，最终 count 的值接近 1 亿，而不是 2 亿。如果循环 10000 次，count 的值接近 20000，原因是两个线程不是同时启动的，有一个时差。

#### 线程切换带来的原子性问题

**多进程分时复用单核CPU**：

由于 IO 太慢，早期的操作系统就发明了多进程，即便在单核的 CPU 上我们也可以一边听着歌，一边写 Bug，这个就是多进程的功劳。

操作系统允许某个进程执行一小段时间，例如 50 毫秒，过了 50 毫秒操作系统就会重新选择一个进程来执行（我们称为“任务切换”），这个 50 毫秒称为“**时间片**”。

![](attachments/Pasted%20image%2020240912113321.png)

在一个时间片内，如果一个进程进行一个 IO 操作，例如读个文件，这个时候该进程可以把自己标记为“休眠状态”并出让 CPU 的使用权，待文件读进内存，操作系统会把这个休眠的进程唤醒，唤醒后的进程就有机会重新获得 CPU 的使用权了。

这里的进程在等待 IO 时之所以会释放 CPU 使用权，是为了让 CPU 在这段等待时间里可以做别的事情，这样一来 CPU 的使用率就上来了；此外，如果这时有另外一个进程也读文件，读文件的操作就会排队，磁盘驱动在完成一个进程的读操作后，发现有排队的任务，就会立即启动下一个读操作，这样 IO 的使用率也上来了。

是不是很简单的逻辑？但是，虽然看似简单，**支持多进程分时复用单核CPU**在操作系统的发展史上却具有里程碑意义，Unix 就是因为解决了这个问题而名噪天下的。


早期的操作系统基于进程来调度 CPU，不同进程间是不共享内存空间的，所以进程要做任务切换就要切换内存映射地址，而一个进程创建的所有线程，都是共享一个内存空间的，所以线程做任务切换成本就很低了。现代的操作系统都基于更轻量的线程来调度，现在我们提到的“任务切换”都是指“线程切换”。

Java 并发程序都是基于多线程的，自然也会涉及到任务切换，也许你想不到，**任务切换竟然也是并发编程里诡异 Bug 的源头之一。任务切换的时机大多数是在时间片结束的时候**，我们现在基本都使用高级语言编程，**高级语言里一条语句往往需要多条 CPU 指令完成，例如上面代码中的`count += 1`，至少需要三条 CPU 指令**。
```text
- 指令 1：首先，需要把变量 count 从内存加载到 CPU 的寄存器；
- 指令 2：之后，在寄存器中执行 +1 操作；
- 指令 3：最后，将结果写入内存（缓存机制导致可能写入的是 CPU 缓存而不是内存）。
```

**操作系统做任务切换，可以发生在任何一条CPU 指令执行完，是的，是 CPU 指令，而不是高级语言里的一条语句**。
对于上面的三条指令来说，我们假设 count=0，如果线程 A 在指令 1 执行完后做线程切换，线程 A 和线程 B 按照下图的序列执行，那么我们会发现两个线程都执行了 count+=1 的操作，但是得到的结果不是我们期望的 2，而是 1。

![](attachments/Pasted%20image%2020240912113730.png)

我们潜意识里面觉得 count+=1 这个操作是一个不可分割的整体，就像一个原子一样，线程的切换可以发生在 count+=1 之前，也可以发生在 count+=1 之后，但就是不会发生在中间。**我们把一个或者多个操作在 CPU 执行的过程中不被中断的特性称为原子性**。

==CPU 能保证的原子操作是 CPU 指令级别的，而不是高级语言的操作符，这是违背我们直觉的地方==。因此，很多时候我们需要在高级语言层面保证操作的原子性。

#### 编译优化带来的有序性问题

那并发编程里还有没有其他有违直觉容易导致诡异 Bug 的技术呢？
有的，就是有序性。顾名思义，有序性指的是程序按照代码的先后顺序执行。编译器为了优化性能，有时候会改变程序中语句的先后顺序。

例如程序中：“a=6；b=7；”编译器优化后可能变成“b=7；a=6；”，在这个例子中，编译器调整了语句的顺序，但是不影响程序的最终结果。不过有时候编译器及解释器的优化可能导致意想不到的 Bug。


在 Java 领域一个经典的案例就是利用双重检查创建单例对象，例如下面的代码：在获取实例 getInstance() 的方法中，我们首先判断 instance 是否为空，如果为空，则锁定 Singleton.class 并再次检查 instance 是否为空，如果还为空则创建 Singleton 的一个实例。

```java
public class Singleton {
  private Singleton() { }
  private static Singleton instance;
  public static Singleton getInstance(){
    if (instance == null) {
      synchronized(Singleton.class) {
        if (instance == null)
          instance = new Singleton();
        }
    }
    return instance;
  }
}
```

假设有两个线程 A、B 同时调用 getInstance() 方法，他们会同时发现 `instance == null` ，于是同时对 Singleton.class 加锁，此时 JVM 保证只有一个线程能够加锁成功（假设是线程 A），另外一个线程则会处于等待状态（假设是线程 B）；线程 A 会创建一个 Singleton 实例，之后释放锁，锁释放后，线程 B 被唤醒，线程 B 再次尝试加锁，此时是可以加锁成功的，加锁成功后，线程 B 检查 `instance == null` 时会发现，已经创建过 Singleton 实例了，所以线程 B 不会再创建一个 Singleton 实例。

这看上去一切都很完美，无懈可击，但实际上这个 getInstance() 方法并不完美。问题出在哪里呢？出在 new 操作上，我们以为的 new 操作应该是有3条CPU指令：
```text
8. 分配一块内存 M；
9. 在内存 M 上初始化 Singleton 对象；
10. 然后 M 的地址赋值给 instance 变量。
```

但是实际上优化后的执行路径却是这样的：
```text
11. 分配一块内存 M；
12. 将 M 的地址赋值给 instance 变量；
13. 最后在内存 M 上初始化 Singleton 对象。
```

优化后会导致什么问题呢？我们假设线程 A 先执行 getInstance() 方法，当执行完指令 2 时恰好发生了线程切换，切换到了线程 B 上；如果此时线程 B 也执行 getInstance() 方法，那么线程 B 在执行第一个判断时会发现 `instance != null` ，所以直接返回 instance，不用去获取锁。而此时的 instance 是没有初始化过的，如果我们这个时候访问 instance 的成员变量就可能触发空指针异常。

![](attachments/Pasted%20image%2020240912114931.png)


#### 小结

要写好并发程序，首先要知道并发程序的问题在哪里，只有确定了“靶子”，才有可能把问题解决，毕竟所有的解决方案都是针对问题的。并发程序经常出现的诡异问题看上去非常无厘头，但是深究的话，无外乎就是直觉欺骗了我们，**只要我们能够深刻理解可见性、原子性、有序性在并发场景下的原理，很多并发 Bug 都是可以理解、可以诊断的**。

在介绍可见性、原子性、有序性的时候，特意提到**缓存**导致的可见性问题，**线程切换**带来的原子性问题，**编译优化**带来的有序性问题，其实缓存、线程、编译优化的目的和我们写并发程序的目的是相同的，都是提高程序性能。但是技术在解决一个问题的同时，必然会带来另外一个问题，所以**在采用一项技术的同时，一定要清楚它带来的问题是什么，以及如何规避**。


## 并发编程的三大问题

|问题类型|描述|举例|
|---|---|---|
|原子性|操作是否被打断|i++|
|可见性|一个核写，另一个核能否看到|flag|
|顺序性|指令执行顺序是否符合代码|data/flag|



### 原子性
#### 原子变量(C11 atomic)
```c
#include <stdatomic.h>

atomic_int flag = 0;
int data = 0;

// producer
data = 100;
atomic_store_explicit(&flag, 1, memory_order_release);

// consumer
if (atomic_load_explicit(&flag, memory_order_acquire)) {
    printf("%d\n", data);
}



分析：
atomic_store(..., memory_order_release); 约等于 store + 写屏障
```


|能力|atomic|
|---|---|
|原子性|✅|
|可见性|✅|
|顺序性|✅（取决 memory_order）|

|类型|含义|
|---|---|
|relaxed|只有原子性|
|acquire|读屏障|
|release|写屏障|
|seq_cst|全屏障（最强）|

#### 读写锁/自旋锁/Mutex 等锁
读写锁/自旋锁 保护了 临界区的操作，要么执行，要么不执行。

```c
pthread_mutex_lock(&m);
// 临界区
pthread_mutex_unlock(&m);

等价于

acquire barrier
critical section
release barrier
```

|能力|锁|
|---|---|
|原子性|✅|
|可见性|✅|
|顺序性|✅|
|互斥|✅|



### 可见性
#### volatile
### 有序性 （顺序性）
#### 内存屏障

内存屏障：解决“顺序性 + 可见性”，但不解决“竞争（原子性）”。


```c
#define rte_mb() _mm_mfence()
#define rte_wmb() _mm_sfence()
#define rte_rmb() _mm_lfence()
#define rte_smp_wmb() rte_compiler_barrier()
#define rte_smp_rmb() rte_compiler_barrier()
/**
 * Compiler barrier.
 *
 * Guarantees that operation reordering does not occur at compile time
 * for operations directly before and after the barrier.
 */
#define rte_compiler_barrier() do {     \
    asm volatile ("" : : : "memory");   \
} while(0)
```


### 原子变量/锁/Volatile关键字和 原子性/可见性/顺序性的关系

`volatile`、`barrier()`、`smp_mb()`、`atomic_t`、`mutex/spinlock` 这五种机制经常被混用，但它们解决的问题完全不同：

![](attachments/Pasted%20image%2020260421235010.png)


![](attachments/Pasted%20image%2020260422001017.png)

**原子变量 = 原子性 +（内置）内存屏障**  
**锁 = 原子变量 + 更强语义（互斥 + 顺序 + 可见性）**  
**内存屏障 = 只解决顺序性和可见性**  
**volatile = 只约束编译器（几乎不能用于线程同步）**


|特性|volatile|内存屏障|atomic|锁|
|---|---|---|---|---|
|防编译器重排|✅|✅|✅|✅|
|防 CPU 乱序执行|❌|✅|✅（取决 memory_order）|✅|
|可见性（多核）|❌|✅|✅|✅|
|原子性|❌|❌|✅|✅|
|互斥|❌|❌|❌|✅|
|易用性|高|低|中|高|
|性能|高|高|高|低|



# 内存一致性(memory model/Memory consistency model)

指令重排虽然一定程度加快了程序的执行，但却带来了额外的问题，快是快了，但程序的正确性却没了。
这个时候就需要有另外的技术来解决的问题, 就是我们要说的 `memory order/barrier/fence`. 这里提到3个词 `memory order`, `memory barrier`, `memory fence`, 其实本质上说的都是一个问题(内存可见性).

## 介绍

`Memory consistency model`又称`Memory model` (内存模型)，定义了使用`Shared memory`(共享内存)执行多线程(Multithread)程序所允许的行为规范。
`Memory model`定义了软硬件接口规范，以便程序员预料硬件会有什么行为，而硬件实现者知道可以使用什么样的优化，消除软硬件在配合上的歧义。

### 共享内存的问题
要了解为什么需要定义内存模型规范，我们先举个例子，如表1所示。系统中有两个cores在执行各自的代码，假设所有变量的初始值都是0，那么最终Core C2中寄存器r2的值应该为多少呢？

![](attachments/Pasted%20image%2020260118211007.png)


## `Store Buffer` 

同时，为了更好的理解内存模型，请看之前文章 [《# 每个程序员都应该了解的内存知识（What every programmer should know about memory）》](https://zhuanlan.zhihu.com/p/611133924) 的 ### `Load/Store Buffer` 部分，先对 `Store Buffer` 这个东西有个初步的印象。

此外，为了更好的理解内存模型，应该从硬件视角去理解，推荐看《Memory Barriers: a Hardware View for Software Hackers》

![](attachments/Pasted%20image%2020250412151904.png)

CPU 在向 Cache 写入的时候，会先向 Store Buffer 里写入，Store Buffer 累积一些写入，然后再写到 Cache 。每个 CPU 有属于它自己的 Cache ，这就带来了缓存一致性的问题，CPU 之间通过 MESI 等协议解决这个问题。
对于 Store Buffer 而言，它写到自己的 Cache 就等于写到内存了，其他 CPU 就应该能看见了，至于复杂的一致性问题，交给 MESI.
MESI 协议用于实现缓存一致性，其定义了四种状态“modified”, “exclusive”, “shared”, “invalid”。

# 参考
```bash
# 原子操作与内存模型/序/屏障 （Atomic operation & Memory model）
https://zhuanlan.zhihu.com/p/611868395

# 每个程序员都应该了解的内存知识（What every programmer should know about memory）
https://zhuanlan.zhihu.com/p/611133924

# Memory Consistency Models（内存一致性模型）
https://zhuanlan.zhihu.com/p/644601840

# 内存一致性(Memory Consistency)模型简介
https://blog.csdn.net/JiMoKuangXiangQu/article/details/130702950

# 上篇 | 说说无锁(Lock-Free)编程那些事（+++++++）
https://cloud.tencent.com/developer/article/1516818

```