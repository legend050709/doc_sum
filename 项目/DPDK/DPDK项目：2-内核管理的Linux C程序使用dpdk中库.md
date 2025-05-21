```table-of-contents
```
# 背景
DPDK中的 `rte_ring` 、`rte_mempool`、`rte_acl` 等库是非常好的库。
如果期望在内核管理的 `Linux C`的项目中使用这些DPDK中的库作为工具，那会不需要重复造轮子，并且这些高性能库可以带来很好的性能。

# 潜在的问题
## rte_eal_init 中的参数
`rte_eal_init`  函数中一般情况下，需要传递 CPU(绑定哪些core，起对应的 线程)，内存（大页内存相关的设置），Pcie设备（网卡相关）等配置参数。
对于  Linux C中的的项目，可能大部分就是期望使用`rte_ring` 、`rte_mempool`，不期望设置`Pcie`设备，以及`CPU Core`的设置；其中`rte_ring` 、`rte_mempool`这2个工具依赖大页内存。

> ps： 在 `rte_eal_init`  函数调用之前，其他的`dpdk`的库中的函数，可能都无法调用。比如：`rte_rand()` 函数来获取随机数。

![](attachments/Pasted%20image%2020250403204353.png)


### 使用范例

```c
    char *argv[] = {"appName", "-l", "1", "--socket-mem", socket_mem_str,
    			    "--socket-limit", socket_limit_str,  "--no-telemetry",
                    "--no-shconf", "--file-prefix", file_prefix_str,   NULL};
    int argc = sizeof(argv) / sizeof(char *) - 1;


    if (rte_eal_init(argc, (char **)argv) < 0) {
        return -1;
    }
```


## per_lcore__lcore_id 的值异常
```c
/**
 * Read/write the per-lcore variable value
 */
#define RTE_PER_LCORE(name) (per_lcore_##name)

static inline unsigned rte_lcore_id(void)
{
	return RTE_PER_LCORE(_lcore_id);
}
```

通过 `rte_eal_init` 传递的 `core 或 lcore `等 core 参数创建的线程，才会有 合法的 `RTE_PER_LCORE(_lcore_id)` 的值。
对于外部的线程，调用 `rte_lcore_id` 得到的线程变量 `per_lcore__lcore_id` 应该没有一个合法的值， 应该默认值是 u32_max (即：4294967295U)。
![](attachments/Pasted%20image%2020250509120831.png)

但是dpdk库中很多的变量或者函数的逻辑都是依赖于 线程变量 `per_lcore__lcore_id`的。比如：`rte_rand()` 函数, 以及  mempool 中 get元素时，如果使用了per-core的cache。如下所示：

![](attachments/Pasted%20image%2020250509122525.png)

### 原因
在 Linux C 内核管理的程序，调用dpdk的`rte_eal_init`的时候，一般只需要通过 `-l` 来指定 master core 线程，比如：`-l 1`,  为了防止创建无用的slave线程，并不多指定多个core，来创建 slave线程。

后续在多线程中，调用 `rte_mempool_get`, 每个线程的 线程变量 `per_lcore__lcore_id`应该都是初始化为 `LCORE_ID_ANY,  即 u32_t_max`的值。
此时是无法通过  `rte_mempool_get`来使用cache的，如果无法使用cache，那么就会通过`rte_ring`的多生产者多消费者来获取元素,这个就涉及到`cas`锁的问题，对性能会存在影响。

### 解决方法
在多线程中，每个线程通过 `RTE_PER_LCORE(_lcore_id) = xxx` 来重新设置自身的  线程变量 `per_lcore__lcore_id`。后续在调用`rte_mempool_get`就可以使用 到 mempool 的 cache了。如下所示：
```c
if (rte_lcore_id() == LCORE_ID_ANY） {
	RTE_PER_LCORE(_lcore_id) = rte_atomic32_add_return(&thread_lcore_id, 1);
}
```

### rte_mempool 的 cache 的使用
```c
static inline unsigned rte_lcore_id(void)
{
	return RTE_PER_LCORE(_lcore_id);
}

static __rte_always_inline int rte_mempool_get_bulk(struct rte_mempool *mp, void **obj_table, unsigned int n)
{
	struct rte_mempool_cache *cache;
	cache = rte_mempool_default_cache(mp, rte_lcore_id());
	rte_mempool_trace_get_bulk(mp, obj_table, n, cache);
	return rte_mempool_generic_get(mp, obj_table, n, cache);
}


static __rte_always_inline struct rte_mempool_cache * rte_mempool_default_cache(struct rte_mempool *mp, unsigned lcore_id)
{
	if (mp->cache_size == 0)
		return NULL;

	if (lcore_id >= RTE_MAX_LCORE)
		return NULL;

	rte_mempool_trace_default_cache(mp, lcore_id,
		&mp->local_cache[lcore_id]);
	return &mp->local_cache[lcore_id];
}


struct rte_mempool * rte_mempool_create_empty(const char *name, unsigned n, unsigned elt_size,
	unsigned cache_size, unsigned private_data_size,
	int socket_id, unsigned flags)
{
	...
	
	/* Init all default caches. */
	if (cache_size != 0) {
		for (lcore_id = 0; lcore_id < RTE_MAX_LCORE; lcore_id++)
			mempool_cache_init(&mp->local_cache[lcore_id],
					   cache_size);
	}

	...
}
```

# 应用场景
## 类UCX的RDMA统一通讯库
**实现一个类UCX的RDMA统一通讯库**
业务通过这样的统一通讯库提供的类socket编程接口，业务代码就可以按照`TCP/IP socket` 的编程习惯，屏蔽底层RDMA的众多概念，以及实现细节，无缝的享受到RDMA带来的高性能网络通信的能力。
因此，就要求这个统一通讯库提供较高的性能，相较于RDMA IB源语也不会存在大的性能损耗。
因此，就希望在该通讯库中通过DPDK中高性能的库作为一些轮子工具来进行开发。

# 参考
```bash

```