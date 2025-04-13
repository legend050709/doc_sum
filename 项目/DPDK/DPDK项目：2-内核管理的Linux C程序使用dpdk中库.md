```table-of-contents
```
# 背景
DPDK中的 `rte_ring` 、`rte_mempool`、`rte_acl` 等库是非常好的库。
如果期望在内核管理的 `Linux C`的项目中使用这些DPDK中的库作为工具，那会不需要重复造轮子，并且这些高性能库可以带来很好的性能。

# 潜在的问题
## rte_eal_init 中的参数
`rte_eal_init`  函数中一般情况下，需要传递 CPU(绑定哪些core，起对应的 线程)，内存（大页内存相关的设置），Pcie设备（网卡相关）等配置参数。
对于  Linux C中的的项目，可能大部分就是期望使用`rte_ring` 、`rte_mempool`，不期望设置Pcie设备，以及`CPU Core`的设置；其中`rte_ring` 、`rte_mempool`这2个工具依赖大页内存。

> ps： 在 `rte_eal_init`  函数调用之前，其他的`dpdk`的库中的函数，可能都无法调用。比如：`rte_rand()` 函数来获取随机数。

![](attachments/Pasted%20image%2020250403204353.png)

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
对于外部的线程，调用 `rte_lcore_id` 得到的线程变量 `per_lcore__lcore_id` 应该没有一个合法的值。

但是dpdk库中很多的变量或者函数的逻辑都是依赖于 线程变量 `per_lcore__lcore_id`的。比如：`rte_rand()` 函数。


# 应用场景
## 类UCX的RDMA统一通讯库
**实现一个类UCX的RDMA统一通讯库**
业务通过这样的统一通讯库提供的类socket编程接口，业务代码就可以按照`TCP/IP socket` 的编程习惯，屏蔽底层RDMA的众多概念，以及实现细节，无缝的享受到RDMA带来的高性能网络通信的能力。
因此，就要求这个统一通讯库提供较高的性能，相较于RDMA IB源语也不会存在大的性能损耗。
因此，就希望在该通讯库中通过DPDK中高性能的库作为一些轮子工具来进行开发。

# 参考
```bash

```