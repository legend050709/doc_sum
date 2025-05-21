```table-of-contents
```

# mempool的per-core的缓存cache
## 背景
在多核系统中，如果多个核心频繁访问同一个内存池的“空闲对象环形队列（ring）”，**每次访问都需要执行原子操作（比如 CAS）**，这会带来很高的 CPU 开销。

## 思路
如何优化？
为了避免这种频繁的共享访问，**DPDK 为每个核心维护了一个本地缓存（per-core cache）**：
- 内存池分配器不再每次都从全局 ring 获取对象
- 而是先从当前核心自己的 cache 中取/还对象
- 只有当本地 cache **空了或满了**，才进行一次批量交互（bulk get/put）

### 总结
==带`cache`的`mempool` = 独立资源池+公共资源池==：**独立资源池为了高性能，公共资源池为了容量扩展**。

每个线程(每个线程绑定到不同的core上)都有自己的独立资源池(即：cache)，这样性能很高。
当某个线程的自身的独立资源池（cache）的资源不足时，还有一个公共资源池。

### 拓展
**问题**
由于`mempool`的`cache`，设置的是每个`core`最大是`512`，不能更大了，然后某个`core`的`cache`不足之后，就需要从`mempool`的公共资源池中申请，对于多生产者多消费者而言，公共资源池中的元素通过`rte_ring`来管理，那么就存在`CAS`加锁的问题。

**思路**
依照：带`cache`的`mempool` = 独立资源池+公共资源池的思路。
可以每个线程再单独创建一个不带`cache`的单生产者、单消费者的`mempool`「前提： 元素的生产、释放都只能在一个线程中」。
独占的`mempool`的资源不足时，还有一个共享的带有`cache`的`mempool`。

## 缓存结构
本地缓存结构：

- 是一个**小型数组**，保存对象指针（用于后续复用）
- 用一个栈结构（top 指针）维护
- 每个核心独占，不共享
- 在创建内存池时可以选择**启用或禁用**

## 自定义外部缓存（External Cache）

除了默认的 per-lcore 缓存外，你也可以使用 **用户自定义的外部缓存**，通过以下 API 实现：
```c
rte_mempool_cache_create() 
rte_mempool_cache_free() 
rte_mempool_cache_flush()
```

# mempool的debug
## 调试模式中的“Cookie”（Cookies）
当开启 **调试模式（debug mode）** 时，DPDK 会在每个分配的对象前后加上一些特殊字段，称为 **cookie**。
这些字段的作用是：用来检测内存越界或覆盖错误（例如缓冲区溢出、未对齐访问）。

## 内存池统计（Stats）
如果启用了 **统计模式（stats mode）**，DPDK 会记录关于从内存池获取/归还对象的统计数据。
这些统计信息保存在 `mempool` 结构体中：
- 例如：每次 `rte_mempool_get()` 和 `rte_mempool_put()` 的次数；
- **每个 lcore 单独维护统计数据**，避免多核并发更新同一个计数器时出现锁竞争；

# mempool的ops（Mempool Handler）
在 `DPDK` 中，每个内存池通过一个名字进行标识，并通过一种称为 **mempool handler** 的机制来管理空闲对象。

## 默认的ops
`rte_mempool_ring`
默认的 handler 是基于 **环形队列（ring-based）** 的实现。参考：`drivers/mempool/ring/rte_mempool_ring.c` 文件。


## 自定义的ops
添加自定义 `ops/handler` 的代码（实现 `mempool` 操作集）；这就相当于**注册一个新的内存池类型**，告诉 `DPDK` 如何分配/释放对象。

你需要：
- 实现自己的 **mempool ops 操作结构体**（比如实现 `.alloc`, `.put`, `.get` 等函数）
- 用宏注册它：`RTE_MEMPOOL_REGISTER_OPS(my_mempool_ops);`


## 使用新的 ops 创建内存池
使用新的 `ops` 创建内存池，你需要使用如下 API：
```c
rte_mempool_create_empty() 
rte_mempool_set_ops_byname()
```
调用顺序是：

1. 使用 `rte_mempool_create_empty()` 创建一个空内存池
2. 使用 `rte_mempool_set_ops_byname()` 指定你要使用的 `handler`（通过名字）

这样这个内存池就会使用你实现的 `ops` 回调逻辑。


### 多个 ops 同时用
你可以在同一个程序中使用多个不同的 `ops`，比如：

- 一个`mempool` 使用默认 `ring` 实现
- 另一个 `mempool` 使用你自定义的 `NUMA-aware` 实现


只要分别设置 `ops` 就可以：
```c
rte_mempool_set_ops_byname(pool1, "ring_mp_mc"); 
rte_mempool_set_ops_byname(pool2, "my_custom_ops");
```
老版本程序一般调用的是：

```c
rte_mempool_create()
```

这个会默认使用 `ring-based handler`。

如果你想切换到新的 `ops`，你需要修改老程序，改用：

```c
rte_mempool_create_empty() + set_ops_byname()
```


### 默认ops和自定义ops
#### 旧接口 `rte_mempool_create()` 的特点
##### 优点
- 简洁，一行搞定：创建 + 绑定 handler + 初始化 + 填充
- 默认使用 **ring-based handler**（适用于大多数通用场景）
- 对于多数 mbuf 使用者足够方便

##### 缺点

|局限点|说明|
|---|---|
|handler 写死|默认使用 ring_mp_mc，不支持自由选择|
|不支持外部内存|比如不能直接用 hugepage file、DMA 区、设备共享内存等|
|扩展性差|不能参与高级定制（NUMA-aware 分配、自定义缓存结构、共享跨设备 buffer 等）|
|某些 flags 不好设置|比如想禁用缓存、启用自定义对齐、物理地址映射控制等受限|

#### 新接口（组合式 API）
##### 使用流程

```c
struct rte_mempool *mp = rte_mempool_create_empty("my_pool", ...);
rte_mempool_set_ops_byname(mp, "ring_mp_mc");     // 可选任何 handler
rte_mempool_populate_default(mp);                // 填充内存
```

##### 新接口的优势

|优势点|描述|
|---|---|
|**高度灵活**|handler 不写死，你可以随时切换，比如使用 DPDK 自带的 `stack`, `bucket`, 或你自定义的|
|**支持外部内存（external memory）**|适配 zero-copy、DMA 显存、设备共享内存等场景|
|**更适合高级应用**|比如网络设备的 buffer 共享、zero-copy、多 NUMA 节点、PMD 驱动协同管理内存等|
|**分步控制，利于调试与扩展**|每一步可插入调试信息、限制策略，构建更复杂的 mempool 创建逻辑|
|**更好支持 testing 和插件机制**|可以动态加载 handler `.so` 文件，插件式扩展内存管理|


##### 适用场景
在哪些情况下使用新接口有 **“显著性能收益”**？

|场景类型|是否建议用新接口|提升点|
|---|---|---|
|NUMA 优化|强烈推荐|降低远程访问开销|
|外部内存 / DMA 显存|必须|Zero-copy、设备共享|
|启动优化|可选|批量填充更快|
|多 PMD 共享 mempool|推荐|内存节省、性能更稳定|
|软件测试、模拟|推荐|更轻更快更可控|
|简单 mbuf 分配|可继续用老接口|无需切换|

#### 新老接口对比
老接口：
```c
struct rte_mempool *mp = rte_mempool_create("mypool", 1024,
                   2048, 512, sizeof(struct rte_pktmbuf_pool_private),
                   NULL, NULL, NULL, NULL, SOCKET_ID_ANY, 0);
```


新接口：
```c
struct rte_mempool *mp = rte_mempool_create_empty("mypool", 1024,2048, 512, SOCKET_ID_ANY, 0);
rte_mempool_set_ops_byname(mp, "ring_mp_mc");
rte_mempool_populate_default(mp);
```

## pktmbuf的 ops 设置
如果你用的是 `rte_pktmbuf_create()`，可以通过配置宏来指定默认使用的 `mempool ops`：
```c
#define RTE_MBUF_DEFAULT_MEMPOOL_OPS "my_ops_name"
```

这样就不用每次都手动 `set ops`，配置文件统一控制即可。

## ops 动态库
如果你的程序使用了 DPDK 的 **共享库（shared libraries）**；你可以通过 EAL 参数 `-d` 指定要加载的 ops `.so` 动态库
```c
./app -d my_handler.so
```

 如果是多进程程序，所有子进程使用的 `-d` 参数 **必须顺序一致**，否则会导致 ops 注册失败或行为错乱。

## 小结
通俗总结：

|内容|说明|
|---|---|
|handler 是什么？|指定一个内存池具体怎么分配/释放的实现方式（比如 ring、stack、GPU buffer）|
|怎么用？|注册 ops 结构体 + `rte_mempool_create_empty()` + `set_ops_byname()`|
|能多个用吗？|一个程序可以同时用多个不同 handler|
|老程序能用吗？|默认用 ring handler，要改代码才能换 handler|
|动态加载注意什么？|所有进程的 `-d` 参数顺序必须一致|

# mempool的内存对齐
在 x86 架构上，根据硬件内存配置的不同，通过在对象之间添加**特定的内存填充（padding）**，可以显著提升性能。

原因是：
通常只访问数据包的**前 64 字节**（例如只看头部做决策），如果所有包的起始地址都落在同一个通道，那 CPU 就只能从一个通道拉数据，造成瓶颈。而如果起始地址均匀分布在多个通道上，**多个通道可以并行工作，访问带宽显著提升**。

## 内存对齐的目标
优化的目标是：
确保每个对象的起始地址**落在不同的内存通道（channel）和 rank 上**，  
从而使所有内存通道被**均衡地使用**，避免瓶颈。特别是在执行 **L3 转发（forwarding）** 或 **流分类（flow classification）** 的时候，性能差异明显。

## 启用内存的channel对齐
如何启用通道（channel）/Rank 优化？
运行 DPDK 应用时，可以通过 **EAL 启动参数** 指定使用的 **内存通道（memory channels）和 rank 数量**。
```bash
./my_app --memory-channels=4
```

## 小结
通俗总结：

|概念|说明|
|---|---|
|内存通道（channel）|CPU 与内存之间的“高速通道”，多通道可并行传输|
|Rank|每个 DIMM 上可被单独访问的 DRAM 组，但它们共享通道|
|优化方式|给每个对象加合适的 padding，使它们分布在不同的通道/rank 上|
|目标|让 CPU 同时利用多个内存通道并行加载数据，提升吞吐|
|实用场景|包处理（L3、分类）只读前 64 字节时效果最明显|
|启用方式|使用 EAL 启动参数 `--memory-channels=N` 显式设置|


# API
## rte_mempool_create
```c
struct rte_mempool * rte_mempool_create(const char *name, unsigned n, unsigned elt_size,
	unsigned cache_size, unsigned private_data_size,
	rte_mempool_ctor_t *mp_init, void *mp_init_arg,
	rte_mempool_obj_cb_t *obj_init, void *obj_init_arg,
	int socket_id, unsigned flags);
```

## rte_mempool_get
## rte_mempool_put

## 其他
### 一个core中进行`rte_mempool_get`，另外一个core进行`rte_mempool_put`是否有问题？
#### 背景
比如：在一个线程中通过 `rte_mempool_get` 获取到元素，然后将元素通过 `rte_ring` 传递到另外一个线程中，然后进行释放。

如果使用了cache，是否存在问题。
如果不适用cache，是否存在问题。

#### 分析
**（1）整体来说，是没有问题的**：

因为：释放的时候，是将元素释放到当前线程所在的`core`的`cache`数组中，而不是基于元素原本所在的`cache`。
而 当前线程所在的`core`的`cache`数组只有当前线程才会操作，其他的线程根本不会操作，因此没有问题。

**（2）额外的影响**：
由于在一个线程中申请，在其他的线程中释放，那么申请的线程的`cache`会很快用完，那么就需要从公共资源池中申请，如果是多消费者，那么可能涉及到`cas`来操作`rte_ring`。

### 存在cache时，mempool的总的元素的个数

# 参考
```bash
# DPDK内存管理 —— mempool介绍
https://zhuanlan.zhihu.com/p/654772667
```