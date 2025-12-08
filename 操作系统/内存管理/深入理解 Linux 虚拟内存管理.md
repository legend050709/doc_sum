```table-of-contents
```

# 地址空间范围
x86_64 架构使用 64 位地址空间，但并不是所有的地址都是可用的。以下是关于用户态指针的一些关键点：

## 地址空间划分

在绝大多数 64 位 Linux 系统上使用的都是 **x86_64 架构**，它虽然是“64 位”，但实际上**虚拟地址没有使用满 64 位**。

**用户态与内核态**: 在 x86_64 架构上，虚拟地址空间通常被划分为用户态和内核态。用户态的虚拟地址空间通常是从 `0x0000000000000000` 到 `0x00007fffffffffff`（即 47 位有效地址），而内核态的虚拟地址空间通常从 `0xffff800000000000` 开始。

```bash
用户态：  0x0000_0000_0000_0000  ~  0x0000_7fff_ffff_ffff
内核态：  0xffff_8000_0000_0000  ~  0xffff_ffff_ffff_ffff
```

 **可用地址范围**: 由于用户态的有效地址范围是 `0x0000000000000000` 到 `0x00007fffffffffff`，这意味着用户态指针的最高 17 位（从第 48 位到第 63 位）必然是 0。这是因为在 64 位地址空间中，x86_64 架构规定了高位地址（大于 `0x00007fffffffffff`）是内核空间，因此在用户态下返回的指针值的最高几位（即第 48 位到第 63 位）始终为 0。

### 新一代CPU（5-level paging）
- 有效位：**57 位**
- 地址范围：
```bash
用户态：  0x0000_0000_0000_0000  ~  0x007f_ffff_ffff_ffff
内核态：  0xff80_0000_0000_0000  ~  0xffff_ffff_ffff_ffff
```

是否启用 5-level paging 取决于：
- CPU 支持 `LA57`（即 57-bit linear address feature）
- 内核启动参数 `la57` 或 `no5lvl`；以及编译时配置

注：目前主流 Linux 发行版（如 Ubuntu、RHEL、Debian）**通常仍默认 48 位 VA**，除非显式开启 5-level paging。

### 查看方法
#### 查看 `/proc/self/maps`
通过`cat /proc/self/maps`，可以看到你当前进程使用的虚拟地址区间（heap、stack、mmap 等）。例如：
```bash
00400000-00452000 r-xp 00000000 08:01 123456 /bin/bash
7fff5d3e0000-7fff5d400000 rw-p 00000000 00:00 0 [stack]
```
如果最高地址在 `0x7fff_xxxx_ffff` 附近，那通常是 48 位。

#### 在程序中打印指针
```c
#include <stdio.h>
int main() {
    void *p = malloc(1);
    printf("Address: %p\n", p);
    return 0;
}

一般输出的地址范围可以帮你判断是 48 位还是 57 位。
```

#### 查看 CPU 是否支持 LA57
```bash
lscpu | grep la57

若有 `la57` 字样，则硬件支持 57 位虚拟地址。实际上，还需要 内核开启 `la57`。
```

### 总结

在 Linux x86_64 架构中，用户态程序返回的指针值的最高 16 位（第 48 位到第 63 位）必然是 0。这是因为用户态的地址空间被限制在 `0x0000000000000000` 到 `0x00007fffffffffff` 之间，任何高于此范围的地址都属于内核态。
因此，用户态指针的有效范围是 `0x0000000000000000` 到 `0x00007fffffffffff`，并且高位的 16 位（第 48 位到第 63 位）始终为 0。

### 应用
**可以只用一个`u64`类型的变量，其中`低48`位保持一个指针的值（比如用户空间的某个结构体的地址，比如`conn`的地址），
`高16`位另外保留其他的标记位。这样，通过一个`u64`类型的变量就可以快速的获取多个信息**。

常用于：如下的 `union epoll_data` 的 `u64`的设置。
```c
#include <sys/epoll.h>

struct epoll_event {
   uint32_t      events;  /* Epoll events */
   epoll_data_t  data;    /* User data variable */
};

union epoll_data {
   void     *ptr;
   int       fd;
   uint32_t  u32;
   uint64_t  u64;
};
```


# 参考
```bash
## 深入理解 Linux 虚拟内存管理

https://xiaolincoding.com/os/3_memory/linux_mem.html#_4-6-%E6%B7%B1%E5%85%A5%E7%90%86%E8%A7%A3-linux-%E8%99%9A%E6%8B%9F%E5%86%85%E5%AD%98%E7%AE%A1%E7%90%86

```