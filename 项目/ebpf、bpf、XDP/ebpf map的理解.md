```table-of-contents
```

# 简介
## BPF Map是什么

![](attachments/Pasted%20image%2020240912120542.png)

如上图，简单来说，BPF Map就是内核空间和用户空间之间用于数据交换、信息传递的桥梁。是eBPF程序中使用的主要数据结构。
BPF Map本质上是以「键/值」方式存储在内核中的数据结构，它们可以被任何知道它们的BPF程序访问。

在内核空间的程序创建BPF Map并返回对应的文件描述符，在用户空间运行的程序就可以通过这个文件描述符来访问并操作BPF Map，从而实现kernel内核空间与user用户空间的数据交换。

## BPF Map数据结构

每个BPF Map由四个值定义：类型、元素的最大个数、值（value）大小(以字节为单位)和键（key）大小(以字节为单位)。

```c
# BPF Map 定义示例
struct {
    __uint(type, BPF_MAP_TYPE_HASH);   // 类型
    __uint(max_entries, 8192);         // 元素的最大个数
    __type(key, pid_t);                // 键大小(以字节为单位)
    __type(value, u64);                // 值大小(以字节为单位)
} rb SEC(".maps");     // SEC(".maps") 声明并创建了一个名为rb的BPFMap
```

## bpf map和 bpf程序的关系

当前一个BPF程序可以直接访问多达64个不同的MAP，并且不同的 eBPF 程序也可以通过相同的 BPF 映射来共享它们的状态，例如，跟踪程序可以与网络程序共享MAP。

![](attachments/Pasted%20image%2020240910170058.png)

# ebpf map的作用

## eBPF内核态程序与用户态程序的交互

在eBPF中，映射（Maps）是一种用于在用户态程序和内核态程序之间交换数据的重要机制。映射类似于键值对存储，允许用户态和内核态程序在共享的数据结构中读取和写入数据。

### 详细步骤

下面是eBPF用户态和内核态如何通过 map 交互的详细步骤：
1. 创建和管理Map:
使用 bpf_create_map 或SEC("maps")宏创建一个 map。

2. 内核态的操作:
在 eBPF 程序中，可以使用 BPF helper 函数，如 bpf_map_update_elem, bpf_map_lookup_elem 和 bpf_map_delete_elem 来操作 map 的内容。

3. 用户态的操作:
在用户空间，同样可以使用对应的helper，如 bpf_map_lookup_elem, bpf_map_update_elem 和 bpf_map_delete_elem，通过前面创建 map 时获得的文件描述符来访问和管理 map 的内容。

4. 双向交互:
内核中的 eBPF 程序可能会根据数据包内容或其他条件更新 map 的值。
同时，用户空间应用程序可能会读取这些 map 来获得统计信息、状态或其他数据，或者更新 map 以影响 eBPF 程序的行为。

5. 共享和引用:
eBPF map 还可以被多个 eBPF 程序引用，或者被用作“固定(pin)”map，使其在多个程序或用户空间应用程序之间持久共享。

# map的类型

## 总体介绍
所有 map 类型的定义：

```c
// https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L130

enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,
    BPF_MAP_TYPE_ARRAY,
    BPF_MAP_TYPE_PROG_ARRAY,
    BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    BPF_MAP_TYPE_PERCPU_HASH,
    BPF_MAP_TYPE_PERCPU_ARRAY,
    BPF_MAP_TYPE_STACK_TRACE,
    BPF_MAP_TYPE_CGROUP_ARRAY,
    BPF_MAP_TYPE_LRU_HASH,
    BPF_MAP_TYPE_LRU_PERCPU_HASH,
    BPF_MAP_TYPE_LPM_TRIE,
    BPF_MAP_TYPE_ARRAY_OF_MAPS,
    BPF_MAP_TYPE_HASH_OF_MAPS,
    BPF_MAP_TYPE_DEVMAP,
    BPF_MAP_TYPE_SOCKMAP,
    BPF_MAP_TYPE_CPUMAP,
    BPF_MAP_TYPE_XSKMAP,
    BPF_MAP_TYPE_SOCKHASH,
    BPF_MAP_TYPE_CGROUP_STORAGE,
    BPF_MAP_TYPE_REUSEPORT_SOCKARRAY,
    BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE,
    BPF_MAP_TYPE_QUEUE,
    BPF_MAP_TYPE_STACK,
    BPF_MAP_TYPE_SK_STORAGE,
    BPF_MAP_TYPE_DEVMAP_HASH,
    BPF_MAP_TYPE_STRUCT_OPS,
    BPF_MAP_TYPE_RINGBUF,
    BPF_MAP_TYPE_INODE_STORAGE,

};
```

你可以使用如下的 bpftool 命令，来查询当前系统支持哪些映射类型：

```bash
# bpftool feature probe | grep map_type
eBPF map_type hash is available
eBPF map_type array is available
eBPF map_type prog_array is available
eBPF map_type perf_event_array is available
eBPF map_type percpu_hash is available
eBPF map_type percpu_array is available
eBPF map_type stack_trace is available
eBPF map_type cgroup_array is available
eBPF map_type lru_hash is available
eBPF map_type lru_percpu_hash is available
eBPF map_type lpm_trie is available
eBPF map_type array_of_maps is available
eBPF map_type hash_of_maps is available
eBPF map_type devmap is available
eBPF map_type sockmap is available
eBPF map_type cpumap is available
eBPF map_type xskmap is available
eBPF map_type sockhash is available
eBPF map_type cgroup_storage is available
eBPF map_type reuseport_sockarray is available
eBPF map_type percpu_cgroup_storage is available
eBPF map_type queue is available
eBPF map_type stack is available
```


在下面的表格中，我给你整理了几种最常用的映射类型及其功能和使用场景：

![](attachments/Pasted%20image%2020240910171006.png)


## 每种 map 引入时的内核版本

见 bcc [维护的文档](https://github.com/iovisor/bcc/blob/master/docs/kernel-versions.md#tables-aka-maps)， 记录了哪个内核版本引入的，以及对应的 patch。


## hash map

### 介绍
Hash map 的实现见 [kernel/bpf/hashtab.c](https://github.com/torvalds/linux/blob/v5.8/kernel/bpf/hashtab.c)。 五种类型共用一套代码：

- `BPF_MAP_TYPE_HASH`
- `BPF_MAP_TYPE_PERCPU_HASH`
- `BPF_MAP_TYPE_LRU_HASH`
- `BPF_MAP_TYPE_LRU_PERCPU_HASH`
- `BPF_MAP_TYPE_HASH_OF_MAPS`

### Hash map 的特点
(1) **key 的长度没有限制**
但显然应该大于 0。

(2) 给定 key 查找 value 时，内部通过哈希实现，而非数组索引

(3) **key/value 是可删除的**
作为对比，**Array 类型**的 map 中，key/value 是**不可删除的**（但用空值覆盖掉 value ，可实现删除效果）。

原因其实也很简单：哈希表是链表，可以删除链表中的元素；array 是内存空间连续的 数组，即使某个 index 处的 value 不用了，这段内存区域还是得留着，不可能将其释放掉。

### percpu hash 和 hash的区别
type hash是 global 的，只有一个实例；percpu hash 是 cpu-local 的，每个 CPU 上都有一个 map 实例；

多核并发访问时，global map 要加锁；per-cpu map 无需加锁，每个核上的程序访问 local-cpu 上的 map，最后将所有 CPU 上的 map 汇总。

### BPF_MAP_TYPE_HASH
基础的哈希表类型，申请时类型如下：
```bash
struct {
    __uint(type, BPF_MAP_TYPE_HASH);
    __type(key, type1);         // key的类型
    __type(value, type2);       // value类型
    __uint(max_entries, 1024);  // 最大 entry 数量
} hash_map SEC(".maps");
```


### BPF_MAP_TYPE_PERCPU_HASH
### BPF_MAP_TYPE_LRU_HASH
### BPF_MAP_TYPE_LRU_PERCPU_HASH

## array map

### BPF_MAP_TYPE_ARRAY

```bash
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __type(key, u32);
    __type(value, type1);
    __uint(max_entries, 256);
} my_map SEC(".maps");

```
可以发现和hash的声明没啥区别，但是在操作的时候key的语义是index。

### BPF_MAP_TYPE_PERCPU_ARRAY

#### 介绍
`BPF_MAP_TYPE_PERCPU_ARRAY`是eBPF 中的一种特殊类型的映射（map）。它用于在eBPF程序中创建一个数组，每个CPU都有自己的副本，可以在不同的CPU上并发地进行读写操作。`BPF_MAP_TYPE_PERCPU_ARRAY`是一种用于多核系统的映射类型，它提供了一种在不同CPU上进行并发读写的机制，避免了竞争条件和锁的使用。

#### 使用场景

#### 范例

## cgroup map
## tracing map
## socket map

## 其他
### BPF_MAP_TYPE_LPM_TRIE
### BPF_MAP_TYPE_XSKMAP

## 对比
### 性能对比
（1）查询性能
一般来说 eBPF 程序作为数据面更多是查询，常用的 map 的查询性能：
**array > percpu array > hash > percpu hash > lru hash > lpm**。
map 查询对 eBPF 性能有不少的影响，比如：lpm 类型 map 的查询在我们测试发现最大影响 20% 整体性能、lru hash 类型 map 查询影响 10%。

（2）**array 的查询性能比 percpu array 更好，hash 的查询性能也比 percpu hash 更好**
这是由于 array 和 hash 的 lookup helper 层面在内核有更多的优化。对于数据面读多写少情况下，使用 array 比 percpu array 更优（hash、percpu hash 同理）；
而对于需要数据面写数据的情况使用 percpu 更优，比如统计计数。

**（3）尽可能在一个 map 中查询到更多的数据，减少 map 查询次数**

（4）尽可能使用 array map
在控制面实现更复杂的逻辑，如分配一个 index，将一些 hash map 查询转换为 array map 查询。

（5）eBPF map 也可以指定 numa 创建，另外不同类型的 map 也会有一些额外的 flags 可以用来调整特性。
比如：lru hash map 有 no_common_lru 选项来优化写入性能。


# map 的操作

bpf map的操作，分为用户态的操作以及内核态的操作，如下所示：

 bpf_map_lookup_elem()是ebpf中查找map中特定key对应value的函数。
 这个函数存在于两个不同的形态：
 1) 内核中提供的helper function，供ebpf程序调用；
 2) libbpf库中提供的用户态接口，最终是通过系统调用来实现查找key对应的value。

```bash

// 用户态
int bpf_map_lookup_elem(int fd, const void *key, void *value)

// 内核态
void *bpf_map_lookup_elem(struct bpf_map *map, const void *key)

```

```c
bpf_map_update_elem

// 内核态：源文件定义 kernel/bpf/helpers.c
BPF_CALL_4(bpf_map_update_elem, struct bpf_map *, map, void *, key,
	   void *, value, u64, flags)

// 用户态 LIBBPF_API：tools/lib/bpf/bpf.h
 int bpf_map_update_elem(int fd, const void *key, const void *value,  __u64 flags);

内核程序访问的函数与用户程序访问的函数是不同的。

- 内核程序可以直接访问映射，并原子性地更新元素。
- 用户程序需要使用文件描符来引用映射，所以更新操作不是原子性的。
```

## 用户态操作

在命令行中输入 man bpf ，就可以查询到 BPF 系统调用的调用格式：

```c
#include <linux/bpf.h>

int bpf(int cmd, union bpf_attr *attr, unsigned int size);
```

BPF 系统调用接受三个参数：
- 第一个，cmd ，代表操作命令
- 第二个，attr，代表 bpf_attr 类型的 eBPF 属性指针，不同类型的操作命令需要传入不同的属性参数；
- 第三个，size ，代表属性的大小。

可以通过`strace`命令看到具体的系统调用（不建议使用`strace`，因为会影响性能）：

![](attachments/Pasted%20image%2020240912200933.png)

注意，不同版本的内核所支持的 BPF 命令是不同的，具体支持的命令列表可以参考内核头文件 include/uapi/linux/bpf.h 中 bpf_cmd 的定义。比如，v5.13 内核已经支持 36个BPF命令
```bash
enum bpf_cmd {
  BPF_MAP_CREATE,
  BPF_MAP_LOOKUP_ELEM,
  BPF_MAP_UPDATE_ELEM,
  BPF_MAP_DELETE_ELEM,
  BPF_MAP_GET_NEXT_KEY,
  BPF_PROG_LOAD,
  BPF_OBJ_PIN,
  BPF_OBJ_GET,
  BPF_PROG_ATTACH,
  BPF_PROG_DETACH,
  BPF_PROG_TEST_RUN,
  BPF_PROG_GET_NEXT_ID,
  BPF_MAP_GET_NEXT_ID,
  BPF_PROG_GET_FD_BY_ID,
  BPF_MAP_GET_FD_BY_ID,
  BPF_OBJ_GET_INFO_BY_FD,
  BPF_PROG_QUERY,
  BPF_RAW_TRACEPOINT_OPEN,
  BPF_BTF_LOAD,
  BPF_BTF_GET_FD_BY_ID,
  BPF_TASK_FD_QUERY,
  BPF_MAP_LOOKUP_AND_DELETE_ELEM,
  BPF_MAP_FREEZE,
  BPF_BTF_GET_NEXT_ID,
  BPF_MAP_LOOKUP_BATCH,
  BPF_MAP_LOOKUP_AND_DELETE_BATCH,
  BPF_MAP_UPDATE_BATCH,
  BPF_MAP_DELETE_BATCH,
  BPF_LINK_CREATE,
  BPF_LINK_UPDATE,
  BPF_LINK_GET_FD_BY_ID,
  BPF_LINK_GET_NEXT_ID,
  BPF_ENABLE_STATS,
  BPF_ITER_CREATE,
  BPF_LINK_DETACH,
  BPF_PROG_BIND_MAP,
};
```

为了方便你掌握，我把用户程序中常用的命令整理成了一个表格，你可以在需要时参考：

![](attachments/Pasted%20image%2020240910172843.png)

### map的创建


BPF 辅助函数中并没有 BPF 映射的创建函数，BPF 映射只能通过用户态程序的系统调用来创建。比如，你可以通过下面的示例代码来创建一个 BPF 映射，并返回映射的文件描述符：

```c
int bpf_create_map(enum bpf_map_type map_type,
       unsigned int key_size,
       unsigned int value_size, unsigned int max_entries)
{
  union bpf_attr attr = {
    .map_type = map_type,
    .key_size = key_size,
    .value_size = value_size,
    .max_entries = max_entries
  };
  return bpf(BPF_MAP_CREATE, &attr, sizeof(attr));
}
```

### map的删除

BPF 系统调用中并没有删除映射的命令，这是因为 BPF 映射会在用户态程序关闭文件描述符的时候自动删除（即close(fd) ）。

如果你想在程序退出后还保留映射，就需要调用 BPF_OBJ_PIN 命令，将映射挂载到 /sys/fs/bpf 中。


### map 的更新

### map的查找
#### BPF_MAP_TYPE_PERCPU_ARRAY type 的map查询的注意事项

BPF_MAP_TYPE_PERCPU_ARRAY  type  的 bpf_map_look_elem()函数

(1)  **内核中的 helper function的实现 **

![](attachments/Pasted%20image%2020240912194951.png)

可以很清楚的看到，内核helper function在根据key查找元素时最终是通过this_cpu_ptr()获取元素指针。

（2） **用户态实现 **
![](attachments/Pasted%20image%2020240912195059.png)

可以看到，这里最终是拷贝了`num_possible_cpus()*元素size` 大小的内存到用户态。
上面内核态的 helper function只访问了一个元素大小的内存，而这里用户态的则是num_possible_cpus()个元素。

即： 对于BPF_MAP_TYPE_PERCPU_ARRAY这种percpu 类型的map，用户态通过系统调用bpf_map_look_elem()去访问时，需要传递num_possible_cpus()个元素大小的内存来存放内核拷贝的数据 。 如果只是按照普通MAP方式，传递一个元素大小的内存，则会发生越界。

**其实在Linux中BPF_MAP_TYPE_PERCPU_ARRAY只是其中一种percpu map，还有其他若干种percpu map，其元素的访问原理也是一样，helper function与用户态系统调用是有区别的**。




## BPF Map 内核态操作
### 不同 map 类型支持的操作（xx_map_ops）

所有 map 类型的定义：

```c
// https://github.com/torvalds/linux/blob/v5.10/include/uapi/linux/bpf.h#L130

enum bpf_map_type {
    BPF_MAP_TYPE_UNSPEC,
    BPF_MAP_TYPE_HASH,
    BPF_MAP_TYPE_ARRAY,
    BPF_MAP_TYPE_PROG_ARRAY,
    BPF_MAP_TYPE_PERF_EVENT_ARRAY,
    BPF_MAP_TYPE_PERCPU_HASH,
    BPF_MAP_TYPE_PERCPU_ARRAY,
    BPF_MAP_TYPE_STACK_TRACE,
    BPF_MAP_TYPE_CGROUP_ARRAY,
    BPF_MAP_TYPE_LRU_HASH,
    BPF_MAP_TYPE_LRU_PERCPU_HASH,
    BPF_MAP_TYPE_LPM_TRIE,
    BPF_MAP_TYPE_ARRAY_OF_MAPS,
    BPF_MAP_TYPE_HASH_OF_MAPS,
    BPF_MAP_TYPE_DEVMAP,
    BPF_MAP_TYPE_SOCKMAP,
    BPF_MAP_TYPE_CPUMAP,
    BPF_MAP_TYPE_XSKMAP,
    BPF_MAP_TYPE_SOCKHASH,
    BPF_MAP_TYPE_CGROUP_STORAGE,
    BPF_MAP_TYPE_REUSEPORT_SOCKARRAY,
    BPF_MAP_TYPE_PERCPU_CGROUP_STORAGE,
    BPF_MAP_TYPE_QUEUE,
    BPF_MAP_TYPE_STACK,
    BPF_MAP_TYPE_SK_STORAGE,
    BPF_MAP_TYPE_DEVMAP_HASH,
    BPF_MAP_TYPE_STRUCT_OPS,
    BPF_MAP_TYPE_RINGBUF,
    BPF_MAP_TYPE_INODE_STORAGE,

};
```


[include/linux/bpf_types.h](https://github.com/torvalds/linux/blob/v5.8/include/linux/bpf_types.h) 中定义了不同类型的 BPF map 所支持的操作：

```c
// include/linux/bpf_types.h

BPF_MAP_TYPE(BPF_MAP_TYPE_ARRAY,            array_map_ops)             // kernel/bpf/arraymap.c
BPF_MAP_TYPE(BPF_MAP_TYPE_PERCPU_ARRAY,     percpu_array_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_PROG_ARRAY,       prog_array_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_PERF_EVENT_ARRAY, perf_event_array_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_CGROUP_ARRAY,     cgroup_array_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_CGROUP_STORAGE,   cgroup_storage_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_HASH,             htab_map_ops)              // kernel/bpf/hashtab.c
BPF_MAP_TYPE(BPF_MAP_TYPE_PERCPU_HASH,      htab_percpu_map_ops)       // kernel/bpf/hashtab.c
BPF_MAP_TYPE(BPF_MAP_TYPE_LRU_HASH,         htab_lru_map_ops)          // kernel/bpf/hashtab.c
BPF_MAP_TYPE(BPF_MAP_TYPE_LRU_PERCPU_HASH,  htab_lru_percpu_map_ops)   // kernel/bpf/hashtab.c
BPF_MAP_TYPE(BPF_MAP_TYPE_LPM_TRIE,         trie_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_STACK_TRACE,      stack_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_ARRAY_OF_MAPS,    array_of_maps_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_HASH_OF_MAPS,     htab_of_maps_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_DEVMAP,           dev_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_SOCKMAP,          sock_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_SOCKHASH,         sock_hash_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_CPUMAP,           cpu_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_XSKMAP,           xsk_map_ops)
BPF_MAP_TYPE(BPF_MAP_TYPE_REUSEPORT_SOCKARRAY, reuseport_array_ops)
```



### 查看
执行下面的命令，可以查看每种类型的辅助函数。
```bash
bpftool feature probe
```

### map的创建
```c
#define SEC(NAME) __attribute__((section(NAME),  used))

struct bpf_map_def SEC("maps") my_bpf_map = {
  .type       = BPF_MAP_TYPE_HASH, 
  .key_size   = sizeof(int),
  .value_size   = sizeof(int),
  .max_entries = 100,
  .map_flags   = BPF_F_NO_PREALLOC,
};


```


说明：
1. 声明`ELF section`属性
2. `bpf_load`扫描目标文件里定义’maps’ section, 通过BPF系统调用创建BPF Map.

```c
/* parses elf file compiled by llvm .c->.o
 * . parses 'maps' section and creates maps via BPF syscall // 就是这里
 * . parses 'license' section and passes it to syscall
 * . parses elf relocations for BPF maps and adjusts BPF_LD_IMM64 insns by
 *   storing map_fd into insn->imm and marking such insns as BPF_PSEUDO_MAP_FD
 * . loads eBPF programs via BPF syscall
 *
 * One ELF file can contain multiple BPF programs which will be loaded
 * and their FDs stored stored in prog_fd array
 *
 * returns zero on success
 */
int load_bpf_file(char *path);

```

`SEC("maps")`没有看到与映射(map)相关联的文件描述符。内核使用`map_data`全局变量来保存BPF程序映射(map)信息。`map_data`是数组结构, 按照程序中指定映射(map)的顺序进行排序。比如：获取程序中第一个映射的文件描述符, 示例:
```c
fd = map_data[O].fd;
```

### BPF Map更新
```c
// 源文件定义kernel/bpf/helpers.c

BPF_CALL_4(bpf_map_update_elem, struct bpf_map *, map, void *, key,
	   void *, value, u64, flags)
```



## 范例

### 用户态获取内核中定义的多个map

```c
// xdp驱动程序：
// 内核中定义的 多个 bpf map

/* Don't fragment flag. */
#define IP_DF       0x4000
#define AF_INET     2
#define AF_INET6    10

/* Define maximum reasonable number of NIC queues supported. */
#define QUEUE_MAX   256

/* A map of configuration options. */
struct {
    __uint(type, BPF_MAP_TYPE_ARRAY);
    __uint(max_entries, QUEUE_MAX);
    __uint(key_size, sizeof(__u32)); /* Must be 4 bytes. */
    __uint(value_size, sizeof(knot_xdp_opts_t));
} opts_map SEC(".maps");

/* A map of AF_XDP sockets. */
struct {
    __uint(type, BPF_MAP_TYPE_XSKMAP);
    __uint(max_entries, QUEUE_MAX);
    __uint(key_size, sizeof(__u32)); /* Must be 4 bytes. */
    __uint(value_size, sizeof(int));
} xsks_map SEC(".maps");

struct ipv6_frag_hdr {
    unsigned char nexthdr;
    unsigned char whatever[7];
} __attribute__((packed));

SEC("xdp")
int xdp_redirect_dns_func(struct xdp_md *ctx)
{
    
    ...
}
```


xdp 用户态程序, 如下所示：
```c
// xdp 用户态程序：
// 基于xpd 程序的 fd 获取信息（bpf_obj_get_info_by_fd），
// 获取到多个 maps，再基于map id 获取 map fd(bpf_map_get_fd_by_id),
// 基于 map fd信息获取 map info(bpf_obj_get_info_by_fd);
// 从而过滤得到需要的 map 的信息.
static int get_bpf_maps(int prog_fd, struct kxsk_iface *iface)
{
    uint32_t *map_ids = calloc(NO_BPF_MAPS, sizeof(*map_ids));
    if (map_ids == NULL) {
        return KNOT_ENOMEM;
    }

    struct bpf_prog_info prog_info = {
        .nr_map_ids = NO_BPF_MAPS,
        .map_ids = (__u64)(unsigned long)map_ids,
    };

    uint32_t prog_len = sizeof(struct bpf_prog_info);
    int ret = bpf_obj_get_info_by_fd(prog_fd, &prog_info, &prog_len);
    if (ret != 0) {
        free(map_ids);
        return ret;
    }

    for (int i = 0; i < NO_BPF_MAPS; ++i) {
        int fd = bpf_map_get_fd_by_id(map_ids[i]);
        if (fd < 0) {
            continue;
        }

        struct bpf_map_info map_info = { 0 };
        uint32_t map_len = sizeof(struct bpf_map_info);
        ret = bpf_obj_get_info_by_fd(fd, &map_info, &map_len);
        if (ret != 0) {
            close(fd);
            continue;
        }

        if (strcmp(map_info.name, "opts_map") == 0) {
            iface->opts_map_fd = fd;
            continue;
        }

        if (strcmp(map_info.name, "xsks_map") == 0) {
            iface->xsks_map_fd = fd;
            continue;
        }

        close(fd);
    }

    if (iface->opts_map_fd < 0 || iface->xsks_map_fd < 0) {
        unget_bpf_maps(iface);
        free(map_ids);
        return KNOT_ENOENT;
    }

    free(map_ids);
    return KNOT_EOK;
}
```


# bpftool 操作 map

在调试 BPF 映射相关的问题时，你还可以通过 bpftool 来查看或操作映射的具体内容。比如，你可以通过下面这些命令创建、更新、输出以及删除映射：

```bash
//创建一个哈希表映射，并挂载到/sys/fs/bpf/stats_map(Key和Value的大小都是2字节)
bpftool map create /sys/fs/bpf/stats_map type hash key 2 value 2 entries 8 name stats_map

//查询系统中的所有映射
bpftool map
//示例输出
//340: hash  name stats_map  flags 0x0
//        key 2B  value 2B  max_entries 8  memlock 4096B

//向哈希表映射中插入数据
bpftool map update name stats_map key 0xc1 0xc2 value 0xa1 0xa2

//查询哈希表映射中的所有数据
 
bpftool map dump name stats_map
//示例输出
//key: c1 c2  value: a1 a2
//Found 1 element

//删除哈希表映射
rm /sys/fs/bpf/stats_map
```


# ebpf map的并发管理

参考：[# Concurrency management in BPF](https://lwn.net/Articles/779120/?spm=a2c6h.12873639.article-detail.13.12df275abmUqw4)

## 背景

由于 eBPF 映射可以发生许多程序同时并发访问同一个 Map；另外，用户态进程和内核中的ebpf程序都会操作map，这些可能会产生竞争条件。

用户态程序和eBPF程序或许是并发的，多个eBPF程序之间也是并发的，eBPF Map是线程安全的吗？

## 思路
多个线程对映射表`map`进行查找和更新可能会造成“丢失修改”问题, 所以

- (1) percpu类型的 map
> 前端使用映射类型的时候最好使用`perl-CPU`的哈希和数组映射类型，最小化冲突（这也是`BCC`和`bpftrace`前端的做法）

- (2) 互斥相加操作`BPF_XADD`
- (3) `BPF`自旋锁机制，可以通过`bpf_spin_lock()`和`bpf_spin_unlock()`实现控制;
- (4) `bpf_map_update_elem()`对常规的`Hash`和`LRU`操作都是原子性的。



## bpf_spin_lock

### 背景

多个线程对映射表`map`进行查找和更新可能会造成“丢失修改”问题, 所以前端使用映射类型的时候最好使用`perl-CPU`的哈希和数组映射类型，最小化冲突（这也是`BCC`和`bpftrace`前端的做法）


### 介绍

`linux`内核5.1增加了`bpf spin lock`之后才有了并发控制，所以需要满足内核版本最低要求。BPF通过BPF自旋锁（`bpf_spin_lock`和`bpf_spin_unlock`）来防止竞争条件, 可以在操作映射元素时对访问的映射元素进行锁定, 自旋锁仅适用于数组、哈希、cgroup存储映射

在 eBPF 程序中，我们可以使用`BPF_FUNC_spin_lock()`和`BPF_FUNC_spin_unlock()`这两个辅助函数对数据加锁解锁，释放锁后其它程序就可以安全地访问该元素。而在用户空间，我们在更新或读取元素时可以使用`BPF_F_LOCK`标志，从而避免数据竞争。


# 参考


```bash
# BPF 进阶笔记（二）：BPF Map 类型详解：使用场景、程序示例
https://arthurchiao.art/blog/bpf-advanced-notes-2-zh/

# BPF 进阶笔记（三）：BPF Map 内核实现
https://arthurchiao.art/blog/bpf-advanced-notes-2-zh/

# 极客时间：编程接口：eBPF程序是怎么跟内核进行交互的？
https://time.geekbang.org/column/article/482459


# ebpf-map的使用介绍
https://blog.csdn.net/qq_18643341/article/details/125233822

# EBPF原子操作避坑指南
https://www.xxfe.com/posts/20231031-ebpf-atomic/

```
