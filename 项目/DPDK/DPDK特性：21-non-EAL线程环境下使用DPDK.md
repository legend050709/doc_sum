```table-of-contents
```
# EAL线程和non-EAL线程
首先，关于non-EAL，并不是说整个程序完全不使用EAL(这样的话几乎所有DPDK功能都不可用)。
程序本身还是要执行`rte_eal_init()`来对EAL初始化，通过参数`-l`、`-c`或者`--lcores`来指定使用哪些cpu lcore，在每一个管辖的lcore上绑定一个线程待命，这些线程由EAL管理，即为EAL线程。由于EAL线程与lcore的绑定关系，很多地方用lcore指代其上的EAL Thread。

除此之外，也可以手工利用pthread运行新的线程，这些线程不受EAL控制，就是所谓的non-EAL线程。

## 区别
EAL 管理的 lcore 线程：
```bash
- 有 `lcore_id`/ `rte_per_lcore` 变量
- 有 per-lcore TLS
- 有 memory barrier
```


non-EAL 线程：
```bash
- `rte_lcore_id()` = `LCORE_ID_ANY`
- per-lcore 变量 未初始化
```

# non-EAL线程中DPDK的受限功能
在[dpdk-24.11 Environment Abstraction Layer (EAL) Library ](https://doc.dpdk.org/guides-24.11/prog_guide/env_abstraction_layer.html)
中，列举了在non-EAL Thread中受限制的功能，==包括`rte_mempool`、`rte_timer`、`rte_log`和一些调试功能==。
提及到的理由是，在non-EAL Thread中，变量_lcore_id的值不是一个有效值，因此所有需要提供_lcore_id的DPDK功能均会受限制。

![](attachments/Pasted%20image%2020251208223423.png)

![](attachments/Pasted%20image%2020251208223123.png)

![](attachments/Pasted%20image%2020251208223206.png)



思路：在`non-EAL Thread`中通过修改 `RTE_PER_LCORE(_lcore_id)` 并==伪造 EAL 线程==来避免这些限制；

## rte_ring的受限

### rte_ring 的 MP/MC 是“假设不会被抢占”的无锁算法
#### 背景
==一般 使用标准`EAL Thread` 的DPDK应用中，每个线程都会分配不同的 `lcore`，并进行帮核。即：不会在一个core中运行2个线程，因此，可能不涉及到抢占。
但是，如果在 `non-EAL Thread`中，可能会存在2个线程，运行在同一个core上，这个时候，可能就涉及到抢占了，需要注意 `rte_ring`的使用。 ==。

DPDK 常见做法：
```bash
每个 worker：绑死一个 CPU Core（`pthread_setaffinity_np`）+ busy loop；
即：一个CPU Core上通常只有 一个线程；结果是：这个 CPU 上：没有调度竞争，不会被抢占。

你怎么调：
	- `SCHED_FIFO`
	- `SCHED_RR`
	- `nice`

效果几乎一样；

即：在绑核 + 线程独占 CPU 的模型下，调度策略几乎不重要。

即： Linux 的调度策略和优先级，只是“同一个 CPU 内部的竞争规则”；线程一旦分布到不同 CPU，它们之间就不存在调度竞争。
```

#### rte_ring 的约束
`rte_ring` 支持多生产者入队（MP enqueue）和多消费者出队（MC dequeue）。  
但是，`rte_ring` 是非抢占（non-preemptive）的，这一点会连带导致使用`rte_ring`的 `rte_mempool` 也成为非抢占的。

```bash
这里的“非抢占（non-preemptive）” 约束意味着：

- 一个 pthread 在某个 ring 上执行 多生产者入队（MP enqueue） 时，  
    不能被 另一个正在对同一个 ring执行 MP enqueue 的 pthread 抢占。
    
- 一个 pthread 在某个 ring 上执行 多消费者出队（MC dequeue） 时，  
    不能被 另一个正在对同一个 ring执行 MC dequeue 的 pthread 抢占。
```

**如果绕过（违反）这一约束，会发生什么**？
```bash
- 第二个 pthread 可能会 一直自旋（spin），直到第一个 pthread 再次被调度运行。
- 更严重的是，如果第一个 pthread 是被一个 更高优先级的执行上下文 抢占的，  
    甚至可能导致死锁（deadlock）。
```

==这意味着如果两个线程在同一个core上操作，那么2th线程则必须等到1th线程调度后才能访问；因此，尽量不要在同一个core上，多个线程对同一个ring做多生产者同时入队或者出队==。


##### 可以使用的场景（安全）
- 可抢占的单生产者 / 单消费者（SP / SC）
- 非可抢占的多生产者 + 可抢占的单消费者
- 可抢占的单生产者 + 非可抢占的多消费者

##### 有条件可以使用（性能风险）
可抢占的多生产者 和/或 多消费者 pthread + 并且这些 pthread 的调度策略 全部是：
```bash
- `SCHED_OTHER`（CFS）
- `SCHED_IDLE`
- `SCHED_BATCH`
```
注：使用者 必须清楚性能损失，并接受自旋等待带来的开销。

##### 绝对禁止使用的场景
多生产者 / 多消费者 pthread + 并且调度策略是：
```bash
- `SCHED_FIFO`
- `SCHED_RR`
```
注： 这种组合下使用 rte_ring 是不允许的。


##### 替代方案：基于lock-free stack 的 mempool handler

作为替代方案，应用可以使用 基于无锁栈（lock-free stack）的 mempool handler。

在考虑使用它时，需要注意：
```bash
- 当前仅支持 aarch64 和 x86_64 平台  ；（因为它使用了 16 字节 CAS 指令，其它平台尚未支持）
- 它的 平均性能 比非可抢占的 `rte_ring` 要差；但可以通过 软件缓存（例如 mempool cache） 来缓解性能问题，从而减少对栈的访问次数。
```

### 小结
在 `non-EAL Thread`中，可能会存在2个线程，运行在同一个core上，这个时候，可能就涉及到抢占了，需要注意 `rte_ring`的使用。
单生产和单消费都是支持抢占的，但是多生产或者多消费在抢占情况下会有概率出现死锁。

rte_ring 的 MP/MC 是“假设不会被抢占”的无锁算法，只能用在“不会被抢占”的线程上；一旦涉及 pthread + 调度优先级，就要么退化为 SP/SC，要么换实现。

## mempool的受限

### 问题：无法使用 per-core的缓存

`rte_mempool` 是一种高性能的对象池，用于快速分配和释放对象（如数据包 `mbuf`）。它通过利用每个 Lcore（逻辑核心）的**本地缓存**来提高性能。

- **Lcore 缓存机制：** 
DPDK 将每个 CPU 核心抽象为 L-core，并使用 `rte_lcore_id()` 来获取 L-core ID。`rte_mempool` 依赖这个 ID 为每个 L-core 提供一个私有的、无需加锁的快速缓存。
    
- **非 EAL 线程的缺陷：**
    - **定义：** 非 EAL `pthread` 指的是那些没有被 DPDK 框架（EAL）注册和管理的原生 Linux 线程。
    - **结果：** 在这些线程中调用 `rte_lcore_id()` 时，它不会返回有效的 L-core ID*。

### 带来的后果与解决方案

|**方面**|**影响**|**解决方案**|
|---|---|---|
|**性能损失**|由于无法获取 L-core ID，`rte_mempool` 内部的 `put/get` 操作会**绕过默认的 L-core 缓存**，直接访问受保护（通常是自旋锁保护）的 **mempool 主存储区**。这引入了锁开销，导致性能下降。|**使用外部缓存：** 应用程序必须自己管理一个缓存（例如，使用 `rte_mempool_cache` 结构），并在调用 `rte_mempool_generic_put()` 和 `rte_mempool_generic_get()` 时**显式地传递**这个缓存参数。|
|**应用场景**|这种线程模式常用于需要将 DPDK 库集成到传统 Linux 应用程序或需要与非 DPDK 库（如数据库、GUI）交互的场景。|**仅适用于需要极低性能开销不敏感的场景。**|

![](attachments/Pasted%20image%2020251209110624.png)

### 潜在重大风险
由于`non-EAL`线程,其  `rte_lcore_id()` 的起始值为  `LCORE_ID_ANY`； 需要在`non-EAL`线程中手动将`rte_lcore_id()` 设置为一个有效值。

如果多个线程设置的 `rte_lcore_id()` 存在重叠，即2个线程的 `rte_lcore_id()`设置了相同。
那么再操作`rte_mempool` 就会存在严重的问题，即2个线程读写相同的`cache`, 导致了很多非预期的行为。


# DPDK EAL对于lcore的管理
在DPDK中，与lcore相关的数据结构有两个:`rte_config` 和`lcore_config`。

```c

/**
 * Structure storing internal configuration (per-lcore)
 */
struct lcore_config {
    pthread_t thread_id;       /**< pthread identifier */
    int pipe_main2worker[2];   /**< communication pipe with main */
    int pipe_worker2main[2];   /**< communication pipe with main */

    lcore_function_t * volatile f; /**< function to call */
    void * volatile arg;       /**< argument of function */
    volatile int ret;          /**< return value of function */

    volatile enum rte_lcore_state_t state; /**< lcore state */
    unsigned int socket_id;    /**< physical socket id for this lcore */
    unsigned int core_id;      /**< core number on socket for this lcore */
    int core_index;            /**< relative index, starting from 0 */
    uint8_t core_role;         /**< role of core eg: OFF, RTE, SERVICE */

    rte_cpuset_t cpuset;       /**< cpu set which the lcore affinity to */
};

extern struct lcore_config lcore_config[RTE_MAX_LCORE];

/**
 * The global RTE configuration structure.
 */
struct rte_config {
    uint32_t main_lcore;         /**< Id of the main lcore */
    uint32_t lcore_count;        /**< Number of available logical cores. */
    uint32_t numa_node_count;    /**< Number of detected NUMA nodes. */
    uint32_t numa_nodes[RTE_MAX_NUMA_NODES]; /**< List of detected NUMA nodes. */
    uint32_t service_lcore_count;/**< Number of available service cores. */
    enum rte_lcore_role_t lcore_role[RTE_MAX_LCORE]; /**< State of cores. */

    /** Primary or secondary configuration */
    enum rte_proc_type_t process_type;

    /** PA or VA mapping mode */
    enum rte_iova_mode iova_mode;

    /**
     * Pointer to memory configuration, which may be shared across multiple
     * DPDK instances
     */
    struct rte_mem_config *mem_config;
} __rte_packed;


/**
 * The lcore role (used in RTE or not).
 */
enum rte_lcore_role_t {
    ROLE_RTE,
    ROLE_OFF,
    ROLE_SERVICE,
    ROLE_NON_EAL,
};

int rte_lcore_is_enabled(unsigned int lcore_id)
{
    struct rte_config *cfg = rte_eal_get_configuration();

    if (lcore_id >= RTE_MAX_LCORE)
        return 0;
    return cfg->lcore_role[lcore_id] == ROLE_RTE;
}
```
## `rte_config`

`rte_config`用来保存EAL命令行参数，`lcore_role`数组是lcore的角色，以lcore_id作为下标，根据命令行参数(`-l`,`-c`,`--lcores`)，EAL thread会被赋为`ROLE_RTE`，非EAL thread被赋为其他值。其实在DPDK中，这个`lcore_role`唯一的作用只是用在函数`rte_lcore_is_enabled()`，，该函数判断给定的`lcore_id`是否是`ROLE_RTE`，进一步被用于`rte_get_next_lcore()`和宏`RTE_LCORE_FOREACH`和`RTE_LCORE_FOREACH_SLAVE`中。

## `lcore_config`

`lcore_config`则包含了关于lcore的绝大部分重要信息。`lcore_config[RTE_MAX_LCORE]`该数组由`rte_eal_init()`在EAL初始化时填写，每一个`EAL Thread`都对应一个`lcore_config`。可以说`EAL Thread`和`non-EAL Thread`根本性区别就在于是否有一个`lcore_config`结构。在一个线程中，如果要用到EAL特性，首先需要得到该线程的`lcore_config`结构。为此，每一个线程都有一个`_lcore_id`，范围是`[0. RTE_MAX_LCORE)`，对应于`lcore_config`数组的下标。
`_lcore_id`定义于
```c
RTE_DEFINE_PER_LCORE(unsigned, _lcore_id) = LCORE_ID_ANY;

其中，`RTE_DEFINE_PER_LCORE`的宏展开为：

#define RTE_DEFINE_PER_LCORE(type, name)			\
	__thread __typeof__(type) per_lcore_##name

```

## 其他

对于一个`EAL Thread`，可以通过自身`_lcore_id`找到对应的`lcore_config`结构;
而`non-EAL Thread`中，`_lcore_id`为`LCORE_ID_ANY`。
在DPDK中，凡是用到线程相关的功能，均需要提供`_lcore_id`（通常由`rte_lcore_id()`获得）。
因此，所有要求提供`_lcore_id`的函数在`non-EAL Thread`中都会受到影响。

# non-EAL线程 分类

![](attachments/Pasted%20image%2020260130112441.png)

`non-EAL` thread 如果注册的话，最好是通过 `rte_thread_register()`；
如果为了简便，直接设置 `valid __lcore_id_` 可能也是可以的。


# 参考
```bash
# non-EAL线程环境下DPDK 的使用
http://jiangzhuti.me/posts/DPDK-non-EAL%E7%8E%AF%E5%A2%83%E4%B8%8B%E7%9A%84%E4%BD%BF%E7%94%A8
```