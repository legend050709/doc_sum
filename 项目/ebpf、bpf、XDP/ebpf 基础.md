```table-of-contents
```
# 概述
# bpf架构
## 指令集
## helper辅助函数
### 定义
什么是 eBPF 辅助函数？eBPF 辅助函数是内核提供给开发者的接口。

为什么要有 eBPF 辅助函数呢？为什么不能像驱动一样直接调用内核函数呢？
这主要是为了保证系统安全。由于 eBPF 程序运行在内核态，为了防止不当调用内核函数导致系统崩溃或安全漏洞，eBPF 程序只能调用内核提供的 eBPF 辅助函数。

截止目前内核共提供了 210 多个 eBPF 辅助函数，具体详细列表可见内核源码文件：include/uapi/linux/bpf.h。

参见：[# 一文讲解eBPF helper 函数的设计与实现](https://blog.csdn.net/youzhangjing_/article/details/134533095)

### 作用

![](attachments/Pasted%20image%2020240808200434.png)

辅助函数是面向开发者的，提供操作BPF程序和BPF Map的工具类函数。由于内核本身会有不定期更新迭代，如果直接调用内核模块，那天可能就不能用了，因此通过定义和维护BPF辅助函数，由BPF辅助函数来去面对后端的内核函数的变化，对开发者透明，形成稳定API接口。

例如，BPF程序不知道如何生成一个随机数，有一个BPF辅助函数会可以帮你检索并询问内核，完成”给我一个随机数”的任务，或者”从BPF Map中读取某个值”等等。任何一种与操作系统内核的交互都是通过BPF辅助函数来完成的，由于这些都是稳定的API，所以BPF程序可以跨内核版本进行移植。

### 可使用的helper函数
可用的辅助函数可能因BPF程序类型而有所不同，例如，与附加到tc层的BPF程序相比，附加到套接字的BPF程序仅允许调用一部分辅助函数。

下图是部分BPF辅助函数的列表：bpf 的helper 函数都是以小写bpf_开头的。
![](attachments/Pasted%20image%2020240808200303.png)

### eBPF 辅助函数的设计
在内核中，struct bpf_func_proto 描述了 eBPF 辅助函数的定义、入参类型、返回值类型等重要信息。这些信息的指定主要是为了通过 eBPF verifier 的安全验证，确保传入数据的可靠性，避免传入错误的参数导致系统崩溃。struct bpf_func_proto 的具体形式的代码片段如下所示：
```bash
struct bpf_func_proto {
 //eBPF 辅助函数具体实现
 u64 (*func)(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5);
 bool gpl_only;
 bool pkt_access;
 bool might_sleep;
 
 // 返回类型
 enum bpf_return_type ret_type;
 
 // 参数类型
 union {
	  struct {
	   enum bpf_arg_type arg1_type;
	   enum bpf_arg_type arg2_type;
	   enum bpf_arg_type arg3_type;
	   enum bpf_arg_type arg4_type;
	   enum bpf_arg_type arg5_type;
	  };
	  enum bpf_arg_type arg_type [5];
 };

// 当参数类型为 ARG_PTR_TO_BTF_ID，需要指明参数的 BTF 编号
 union {
	  struct {
		   u32 *arg1_btf_id;
		   u32 *arg2_btf_id;
		   u32 *arg3_btf_id;
		   u32 *arg4_btf_id;
		   u32 *arg5_btf_id;
	  };
	  u32 *arg_btf_id [5];
	  struct {
		   size_t arg1_size;
		   size_t arg2_size;
		   size_t arg3_size;
		   size_t arg4_size;
		   size_t arg5_size;
	  };
	  size_t arg_size [5];
 };
 // 返回参数的 BTF 编号
 int *ret_btf_id;
 bool (*allowed)(const struct bpf_prog *prog);
};
```

其中，`func` 表示该 eBPF 辅助函数的具体实现，实现了特定的功能。`bpf_return_type` 描述该 eBPF 辅助函数的返回参数类型，而 `argx_type` 描述该函数的入参类型。

下面将对入参类型和返回值类型进行解析。

#### 入参类型

入参类型分为基本类型和扩展类型。扩展类型在基本类型的基础上，添加了空指针类型，即允许入参为空指针。另外，当参数类型为 ARG_PTR_TO_BTF_ID 时，则需要在 struct bpf_func_proto 的成员 argx_btf_id 指明具体的 btf 编号。

注：BTF 编号可以看成内核数据类型的编号，通过该编号可以确定数据类型。
##### 基本类型
基本类型大致包含三类：
（1）指针类型，指针类型又可以进行细分：
	1）具体类型的指针类型，如 ARG_PTR_TO_SOCKET 表示 struct socket 指针；
	2）由 BTF 编号确定数据类型的指针类型，如 ARG_PTR_TO_BTF_ID 表示某一内核数据类型指针，且该内核数据类型由 BTF 编号指定；
	3）指向某一类型内存的指针，如 ARG_PTR_TO_MAP_KEY 指向 eBPF 程序栈内存的指针。
（2）整数类型，如 ARG_CONST_SIZE 表示整数，且该整数的值不能为 0；
（3）任意类型，即 ARG_ANYTHING，其表示任意类型，但是需要初始化该值，否则 eBPF verifier 会报 未初始化 等相关错误。

完整的基本类型如下表所示：

![](attachments/Pasted%20image%2020240808201328.png)

![](attachments/Pasted%20image%2020240808201351.png)

##### 扩展类型
包含的扩展类型如下表所示：

![](attachments/Pasted%20image%2020240808201432.png)

#### 返回值类型

同参数类型类似，返回值类型也分为基本类型和扩展类型。扩展类型也是在基本类型的基础添加了空指针类型。

##### 基本类型

![](attachments/Pasted%20image%2020240808201515.png)

##### 扩展类型

扩展类型是在基本类型的基础上，添加了空指针类型，表示返回值可能是空指针，那么 eBPF verifier 需要考虑针对空指针进行安全验证。

![](attachments/Pasted%20image%2020240808201536.png)

### eBPF 辅助函数的实现

下面以 bpf_perf_event_output 为例介绍 eBPF 辅助函数的实现。eBPF 辅助函数 bpf_perf_event_output 是应用最广泛的一个，其主要功能是将数据通过 perf 缓冲区传送给用户态程序。实现 bpf_perf_event_output 需要完成以下三个步骤：

```text
1. 定义 struct bpf_func_proto 结构体，为 bpf_perf_event_output 辅助函数指定功能函数、参数类型、返回值类型等；
2. 为 bpf_perf_event_output 辅助函数分配唯一的编号；
3. 将 bpf_perf_event_output 与特定的 eBPF 程序类型绑定，以确保只有该类型的程序才能调用该辅助函数。
```
#### 定义 struct bpf_func_proto

```c
BPF_CALL_5 (bpf_perf_event_output, struct pt_regs *, regs, struct bpf_map *, map,
    u64, flags, void *, data, u64, size)
{
 ......
 return err;
}
 
static const struct bpf_func_proto bpf_perf_event_output_proto = {
 .func  = bpf_perf_event_output,
 .gpl_only = true,
 .ret_type = RET_INTEGER,
 .arg1_type = ARG_PTR_TO_CTX,
 .arg2_type = ARG_CONST_MAP_PTR,
 .arg3_type = ARG_ANYTHING,
 .arg4_type = ARG_PTR_TO_MEM | MEM_RDONLY,
 .arg5_type = ARG_CONST_SIZE_OR_ZERO,
};
```


bpf_perf_event_output 的入参类型分别是：
```c
ARG_PTR_TO_CTX: struct pt_regs 指针
ARG_CONST_MAP_PTR: struct bpf_map 指针
ARG_ANYTHING：任意类型，且数值已初始化
ARG_PTR_TO_MEM | MEM_RDONLY: 指向栈、报文或 eBPF map 元素值的指针
ARG_CONST_SIZE_OR_ZERO: 整数且该整数值可为 0
```

返回值类型是整数类型：RET_INTEGER.

#### 添加编号
在完成 struct bpf_func_proto 的定义之后，需要为其分配一个唯一的编号。下面的代码片段通过将其扩展为 BPF_FUNC_perf_event_output 宏定义，并将该辅助函数的编号设置为 25，即  `#define BPF_FUNC_perf_event_output 25`
**注：该代码片段位于内核源文件：`include/uapi/linux/bpf.h`**

```c
#define ___BPF_FUNC_MAPPER (FN, ctx...) 
 FN (unspec, 0, ##ctx)    \
 ......
 FN (perf_event_output, 25, ##ctx)  \
 ......
```


#### 绑定 eBPF 程序类型

最后一步是要指定允许调用该辅助函数的 eBPF 程序类型。
例如，下面的代码片段中，允许 `BPF_PROG_TYPE_KPROBE` 类型的 eBPF 程序调用 `bpf_perf_event_output` 辅助函数。如果未指定允许调用该辅助函数的程序类型的 eBPF 程序调用了该辅助函数，则在 eBPF 程序加载过程会出现类似于 `unknown func bpf_perf_event_output#25` 的 eBPF verifier 错误提示。

```c
static const struct bpf_func_proto *
kprobe_prog_func_proto (enum bpf_func_id func_id, const struct bpf_prog *prog)
{
 switch (func_id) {
 case BPF_FUNC_perf_event_output:
  return &bpf_perf_event_output_proto;
 ......
 default:
  return bpf_tracing_func_proto (func_id, prog);
 }
}
const struct bpf_verifier_ops kprobe_verifier_ops = {
 .get_func_proto  = kprobe_prog_func_proto,  // 验证改类型的 eBPF 程序是否可调用 func_id 所代表的辅助函数
 .is_valid_access = kprobe_prog_is_valid_access,
};
```

### 应用

**问题**：
您是否曾遇到过类似于 `R2 type=ctx expected=fp, pkt, pkt_meta, map_value` 的 eBPF verifier 报错？

**分析**
下面是引起该错误的代码示例。
```c
struct
{
    __uint (type, BPF_MAP_TYPE_HASH);
    __type (key, struct sock *);
    __type (value, struct sockmap_val);
    __uint (max_entries, 1024);
} sockmap SEC (".maps");
 
struct sockmap_val
{
    int nothing;
};
 
SEC ("tracepoint/tcp/tcp_rcv_space_adjust")
int tp__tcp_rcv_space_adjust (struct trace_event_raw_tcp_event_sk *ctx)
{
    struct sockmap_val *sv = bpf_map_lookup_elem (&sockmap, &ctx->skaddr);
    if (sv)
        bpf_printk ("% d\n", sv->nothing);
    return 0;
}
```

首先解释一下错误信息 `R2 type=ctx expected=fp, pkt, pkt_meta, map_value` 的含义。该错误表示 R2 寄存器的数据类型应该是指向栈内存的指针、报文指针、或者 eBPF map 的元素值指针，但实际数据类型是 ctx，即指向 struct pt_regs 的指针。因此，该问题实际上是因为数据类型不匹配引起的。

在调用 bpf_map_lookup_elem (&sockmap, &ctx->skaddr) 函数时，我们传递的参数 &ctx->skaddr 是 ctx 类型参数，而不是 fp 类型参数。那么为什么会有这个限制呢？

根据上文所述，eBPF 辅助函数的入参类型是通过 struct bpf_func_proto 进行定义的。我们可以参考 bpf_map_lookup_elem 辅助函数在内核代码中的实现来解释这个问题。在该函数的代码片段中，可以看到它的第二个入参类型为 ARG_PTR_TO_MAP_KEY，即指向 eBPF 程序栈内存的指针，也就是 fp。

```c
const struct bpf_func_proto bpf_map_lookup_elem_proto = {
 .func  = bpf_map_lookup_elem,
 .gpl_only = false,
 .pkt_access = true,
 .ret_type = RET_PTR_TO_MAP_VALUE_OR_NULL,
 .arg1_type = ARG_CONST_MAP_PTR,
 .arg2_type = ARG_PTR_TO_MAP_KEY,
};
```

**解决**
针对这个问题，一般的解决方法是先定义一个栈变量，将 `ctx->skaddr` 的值存储到栈上，例如 `u64 skaddr = ctx->skaddr`，然后在调用 `bpf_map_lookup_elem` 函数时，将该栈变量的地址 `&skaddr` 作为函数的参数传递进去。

### 其他
#### 辅助函数和系统调用
helper函数，不是系统调用，在内核中没有实现。
每个辅助函数的实现具有类似系统调用的共享函数签名。该签名定义如下：
```c
u64 fn(u64 r1, u64 r2, u64 r3, u64 r4, u64 r5)
```

内核将辅助函数抽象为宏`BPF_CALL_0()`到`BPF_CALL_5()`，类似于系统调用。

下面的示例是一个从辅助函数中提取的部分，该辅助函数通过调用相应的映射实现回调函数来更新映射元素：
```c
BPF_CALL_4(bpf_map_update_elem, struct bpf_map *, map, void *, key,
           void *, value, u64, flags)
{
    WARN_ON_ONCE(!rcu_read_lock_held());
    return map->ops->map_update_elem(map, key, value, flags);
}

const struct bpf_func_proto bpf_map_update_elem_proto = {
    .func           = bpf_map_update_elem,
    .gpl_only       = false,
    .ret_type       = RET_INTEGER,
    .arg1_type      = ARG_CONST_MAP_PTR,
    .arg2_type      = ARG_PTR_TO_MAP_KEY,
    .arg3_type      = ARG_PTR_TO_MAP_VALUE,
    .arg4_type      = ARG_ANYTHING,
};
```

## Syscall commands
**对象创建命令**
返回fd。
```bash
BPF_MAP_CREATE
BPF_PROG_LOAD
BPF_BTF_LOAD
BPF_LINK_CREATE
BPF_ITER_CREATE
BPF_RAW_TRACEPOINT_OPEN
```

**bpf map相关命令**
```c
BPF_MAP_CREATE
BPF_MAP_LOOKUP_ELEM
BPF_MAP_UPDATE_ELEM
BPF_MAP_DELETE_ELEM
BPF_MAP_GET_NEXT_KEY
BPF_MAP_LOOKUP_BATCH
BPF_MAP_LOOKUP_AND_DELETE_BATCH
BPF_MAP_UPDATE_BATCH
BPF_MAP_DELETE_BATCH
BPF_MAP_LOOKUP_AND_DELETE_ELEM
BPF_MAP_FREEZE
```


**pin相关命令**
```c
BPF_OBJ_PIN
BPF_OBJ_GET
```

**bpf program程序相关命令**
```c
BPF_PROG_LOAD
BPF_PROG_ATTACH
BPF_PROG_DETACH
BPF_PROG_TEST_RUN
BPF_PROG_RUN
BPF_PROG_BIND_MAP
```

**对象查询以及迭代遍历相关命令**
```c
BPF_PROG_GET_NEXT_ID
BPF_MAP_GET_NEXT_ID
BPF_PROG_GET_FD_BY_ID
BPF_MAP_GET_FD_BY_ID
BPF_OBJ_GET_INFO_BY_FD
BPF_PROG_QUERY
BPF_BTF_GET_FD_BY_ID
BPF_TASK_FD_QUERY
BPF_BTF_GET_NEXT_ID
BPF_LINK_GET_FD_BY_ID
BPF_LINK_GET_NEXT_ID

```

**link相关命令**
```c
BPF_LINK_CREATE
BPF_LINK_UPDATE
BPF_LINK_DETACH
```

**统计命令**
```c
BPF_ENABLE_STATS
```

## ebpf map
### ebpf map类型
#### BPF_MAP_TYPE_XSKMAP
#### BPF_MAP_TYPE_PERCPU_ARRAY

## object pin钉住对象
## 尾调用tail calls
## bpf to bpf calls
## 即时编译器JIT
## 加固hardening
## offload
## BTF


# 工具链
## 开发环境
## LLVM
## iproutes包

### 使用范例
#### xdp的使用

![](attachments/Pasted%20image%2020240719162129.png)

## bpftool
## bpf sysctls




# 参考
```bash

# 比较好的英文文档，介绍。
https://ebpf-docs.dylanreimerink.nl/linux/concepts/af_xdp/#zero-copy-mode

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# 极客时间，运行原理：eBPF 是一个新的虚拟机吗？
https://time.geekbang.org/column/article/481889

# [译] [论文] XDP (eXpress Data Path)：在操作系统内核中实现快速、可编程包处理（ACM，2018）
https://arthurchiao.art/blog/xdp-paper-acm-2018-zh/

# [译] Cilium：BPF 和 XDP 参考指南（2021）
https://arthurchiao.art/blog/cilium-bpf-xdp-reference-guide-zh/

# [译] 基于 BPF/XDP 实现 K8s Service 负载均衡 (LPC, 2020)
https://arthurchiao.art/blog/cilium-k8s-service-lb-zh/


# BPF 进阶笔记（一）：BPF 程序（BPF Prog）类型详解：使用场景、函数签名、执行位置及程序示例
https://arthurchiao.art/blog/bpf-advanced-notes-1-zh/

# BPF架构（二）
https://barryx.cn/cilium_bpf-xdp_2/

# [EBPF学习计划](https://davidlovezoe.club/wordpress/archives/862)
https://davidlovezoe.club/wordpress/archives/862

```