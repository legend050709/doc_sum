```table-of-contents
```


# 数据结构
## GCC 现代原子内建（完整的C11 内存模型写法）
### `__ATOMIC_ACQUIRE`
### `__ATOMIC_RELAXED`

# 函数

## GCC 现代原子内建（完整的C11 内存模型写法）
以`__atomic`开头的函数一般为系统或者编译器内置的函数，在这里即`gcc`的内置函数，主要来实现原子操作。

### `__atomic_fetch_sub`
### `__atomic_fetch_add`
### `__atomic_load_n`
### `__atomic_store_n`
### `__atomic_exchange_n`
### `__atomic_compare_exchange_n`

`__atomic_compare_exchange_n` 是 GCC 提供的 **原子比较并交换（Compare-And-Swap，CAS）** 内建函数，用于在**多线程/多核环境**下无锁地更新共享变量，是实现 lock-free / wait-free 算法的核心原语之一。

```c
bool __atomic_compare_exchange_n(
    type *ptr,
    type *expected,
    type desired,
    bool weak,
    int success_memorder,
    int failure_memorder
);
```
比较`ptr、expected`指向内容，若相同则将`desired`中的值写到`ptr`,否则将`ptr`中的值写入`expected`;

#### 注意
##### CAS 失败会修改 `expected`


### `__atomic_test_and_set`

## GCC 传统原子内建（legacy写法）
`__sync_*` 是 **legacy**，GCC 官方不再推荐新代码使用。

### `__sync_fetch_and_add` 和 `__sync_add_and_fetch`

|函数|返回值|
|---|---|
|`__sync_fetch_and_add(p, v)`|**旧值**|
|`__sync_add_and_fetch(p, v)`|**新值**|

### `__sync_fetch_and_sub` 和 `__sync_sub_and_fetch`
### `__sync_bool_compare_and_swap`


## `__sync_*`和 `__atomic_*` 对比和选择

### `__sync_*`和 `__atomic_*` 对比
`__sync_*` 是旧时代的“大锤”，  
`__atomic_*` 是现代、可控、推荐的新接口。



|维度|`__sync_fetch_and_add`|`__atomic_fetch_add`|
|---|---|---|
|是否推荐|❌ 不推荐新代码|✅ 推荐|
|内存序|固定 `seq_cst`|可选|
|屏障强度|最强|按需|
|性能|较差|可优化|
|标准兼容|非标准|对齐 C11|
|表达能力|弱|强|
|可读性|一般|明确|


### `__sync_*`和 `__atomic_*`的选择

**`__sync_*` 是“我不管，给我最安全的”；  
`__atomic_*` 是“我知道我要什么语义”。**

新代码：一律用 `__atomic_*` 或 C11的 `stdatomic.h`
`__sync_*` 只用于：
- 老代码维护
- 快速验证
- 你明确需要 `seq_cst` 且不在乎性能

如果你能说清楚内存序，就该用 `__atomic_*`； 你说不清楚，用 `__sync_*` 只是暂时安全。

# 参考
```bash

```