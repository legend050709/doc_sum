```table-of-contents
```
# EAL线程和non-EAL线程
首先，关于non-EAL，并不是说整个程序完全不使用EAL(这样的话几乎所有DPDK功能都不可用)。
程序本身还是要执行`rte_eal_init()`来对EAL初始化，通过参数`-l`、`-c`或者`--lcores`来指定使用哪些cpu lcore，在每一个管辖的lcore上绑定一个线程待命，这些线程由EAL管理，即为EAL线程。由于EAL线程与lcore的绑定关系，很多地方用lcore指代其上的EAL Thread。

除此之外，也可以手工利用pthread运行新的线程，这些线程不受EAL控制，就是所谓的non-EAL线程。

# non-EAL线程中DPDK的受限功能
在[dpdk-24.11 Environment Abstraction Layer (EAL) Library ](https://doc.dpdk.org/guides-24.11/prog_guide/env_abstraction_layer.html)
中，列举了在non-EAL Thread中受限制的功能，包括`rte_mempool`、`rte_timer`、`rte_log`和一些调试功能。
提及到的理由是，在non-EAL Thread中，变量_lcore_id的值不是一个有效值，因此所有需要提供_lcore_id的DPDK功能均会受限制。

![](attachments/Pasted%20image%2020251208223423.png)

![](attachments/Pasted%20image%2020251208223123.png)

![](attachments/Pasted%20image%2020251208223206.png)


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



# 参考
```bash
# non-EAL线程环境下DPDK 的使用
http://jiangzhuti.me/posts/DPDK-non-EAL%E7%8E%AF%E5%A2%83%E4%B8%8B%E7%9A%84%E4%BD%BF%E7%94%A8
```