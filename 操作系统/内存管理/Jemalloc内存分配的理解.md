```table-of-contents
```

# Jemalloc和dpdk的mempool
## dpdk的mempool
### 优点
### dpdk的mempool移植到Linux C程序中
**目标**
将DPDK的mempool以及依赖工具整体移植到通用的Linux C的程序中.

> 希望在普通Linux环境下使用类似DPDK的高效内存管理机制，但不需要依赖DPDK的特定环境，比如内核旁路或大页内存。这可能用于需要高性能内存管理的应用，比如网络服务器、数据处理程序等，但又不希望引入DPDK的复杂性。



## Jemalloc和dpdk的mempool对比
### 性能特征对比
|**指标**|Jemalloc|DPDK Mempool|
|---|---|---|
|单对象分配耗时|15-30 ns|5-12 ns|
|批量分配(64对象)|400-600 ns|80-150 ns|
|内存碎片率|<5% (优化后)|0% (固定对象池)|
|线程扩展性|线性扩展到32核|近乎线性扩展到128核|


### 内存布局对比


## Jemalloc和dpdk的mempool对比
要考虑用户的实际应用场景。如果用户在处理大量的网络数据包，比如DPDK应用，那么mempool会更适合，因为它减少了内存分配的开销，支持批量操作和对象重用。
而如果是通用服务器应用，需要处理各种内存分配请求，Jemalloc可能更合适。



# 参考
```bash
# Jemalloc内存分配与优化实践
https://mp.weixin.qq.com/s/U3uylVKZ-FsMjdeX3lymog
```
