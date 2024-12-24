```table-of-contents
```
# 概述

![](attachments/Pasted%20image%2020240910180512.png)

# bpf架构
## 概述
linux 内核的 bpf doc的官方文档，如下所示：

[ linux内核bpf 官方文档](https://docs.kernel.org/bpf/index.html)

![](attachments/Pasted%20image%2020240821165441.png)

如下图（图片来自 BPF Internals）所示，eBPF 在内核中的运行时主要由  5  个模块组成：

![](attachments/Pasted%20image%2020240910200217.png)

- （1）第一个模块是  eBPF 辅助函数。它提供了一系列用于 eBPF 程序与内核其他模块进行交互的函数。这些函数并不是任意一个 eBPF 程序都可以调用的，具体可用的函数集由 BPF 程序类型决定。关于 BPF 程序类型，我会在 06 讲 中进行讲解。
- （2）第二个模块是  eBPF 验证器。它用于确保 eBPF 程序的安全。验证器会将待执行的指令创建为一个有向无环图（DAG），确保程序中不包含不可达指令；接着再模拟指令的执行过程，确保不会执行无效指令。
- （3）第三个模块是由  11 个 64 位寄存器、一个程序计数器和一个 512 字节的栈组成的存储模块。这个模块用于控制 eBPF 程序的执行。其中，R0 寄存器用于存储函数调用和 eBPF 程序的返回值，这意味着函数调用最多只能有一个返回值；R1-R5 寄存器用于函数调用的参数，因此函数调用的参数最多不能超过 5 个；而 R10 则是一个只读寄存器，用于从栈中读取数据。
- （4）第四个模块是即时编译器，它将 eBPF 字节码编译成本地机器指令，以便更高效地在内核中执行。
- （5）第五个模块是  BPF 映射（map），它用于提供大块的存储。这些存储可被用户空间程序用来进行访问，进而控制 eBPF 程序的运行状态。


# eBPF 虚拟机
## eBPF 虚拟机 和系统虚拟机

eBPF 是一个运行在内核中的虚拟机，很多人在初次接触它时，会把它跟系统虚拟化（比如 kvm）中的虚拟机弄混。其实，虽然都被称为“虚拟机”，系统虚拟化和 eBPF 虚拟机还是有着本质不同的。

系统虚拟化基于 x86 或 arm64 等通用指令集，这些指令集足以完成完整计算机的所有功能。而为了确保在内核中安全地执行，eBPF 只提供了非常有限的指令集。这些指令集可用于完成一部分内核的功能，但却远不足以模拟完整的计算机。为了更高效地与内核进行交互，eBPF 指令还有意采用了 C 调用约定，其提供的辅助函数可以在 C 语言中直接调用，极大地方便了 eBPF 程序的开发。



## ebpf 指令集

# bpf的开发方式

## 手动开发每一步

### 流程
简单的总结为：

1. 使用C语言开发一个`eBPF`程序

2. 借助`LLVM`将`eBPF`程序编译为`BPF`字节码
```text
在最开始的开发中，必须通过手工编写`eBPF`汇编代码，并使用内核的`bpf_asm`汇编程序来生成BPF字节码；
现在只需要使用`LLVM Clang`编译器增加了对`eBPF`后端的支持，现在可以将C语言写的程序通过`LLVM Clang`编译器，编译成字节码。

通过使用Clang编译器，配合-march=bpf参数，您就可以用C语言编写自己的eBPF程序了。
```
3. 通过`bpf`系统调用将`BPF`字节码提交给内核
```text
可以使用`bpf()`系统调用函数和`BPF_PROG_LOAD`命令，直接加载包含这个字节码的对象文件。
```
4. 内核验证并运行`BPF`字节码，并把相关的状态保存到`BPF`映射`map`中
5. 用户程序通过`BPF`查询`BPF`字节码的运行状态

### 内核范例

在内核代码的 `samples/bpf/` 目录下有很多eBPF程序的示例，它们的文件名称大部分都具有`_kern.c`的后缀。Clang编译出来的目标文件(eBPF字节码)，需要由在本机运行的一个程序进行加载(这些示例的文件名称中通常具有`_user.c`).
> 注：`kern.c`和`user.c`分别对应内核态和用户态的`eBPF`使用。

### libbpf 库

为了更容易地编写eBPF程序，内核提供了libbpf库，其中包括用于加载程序、创建和操作eBPF对象的帮助函数。
举个例子，一个eBPF程序和使用libbpf库的用户程序的抽象的工作流程一般像如下这样的：

（1）读取eBPF字节码到用户应用程序中的缓冲区，并将其传递给bpf_load_program()函数；

（2）eBPF程序，当在内核运行时，它将调用bpf_map_lookup_elem()函数来查找map中的元素，并存储新值给这个元素；

（3）用户应用程序调用bpf_map_lookup_elem()函数来读取eBPF程序存储在内核中的值。

## bcc 开发
## bpftrace 开发

# ebpf 程序类型



![](attachments/Pasted%20image%2020240913114358.png)

程序类型是针对加载`eBPF`程序的系统调用命令`BPF_PROG_LOAD`的第二个参数中定义的；
范例如下所示：
![](attachments/Pasted%20image%2020240913114418.png)

`BPF_PROG_LOAD`加载的程序类型定义了以下四个方面：

1. 程序可以附加在哪里
> 即可以挂载的事件类型以及事件的参数
3. 验证器允许调用内核哪些辅助函数
4. 网络包数据是否可以直接访问
5. 作为第一个参数传递给程序的对象类型

## 内核支持的程序类型

![](attachments/Pasted%20image%2020241218173751.png)

实际上，程序类型本质上定义了一个`API`，**通过不同的程序类型区分允许调用的不同函数列表**；目前内核支持的`eBPF`程序类型列表如下所示：

（1）用于BPF追踪的程序类型:
```bash
BPF_PROG_TYPE_KPROBE: 用于内核动态插桩和用户态插桩(即kprobe和uprobe)
BPF_PROG_TYPE_TRACEPOINT: 用于内核静态跟踪点
BPF_PROG_TYPE_PERF_EVENT: 用于perl_event，包括PMC
BPF_PROG_TYPE_RAW_TRACEPOINT: 用于跟踪点，不处理参数
```

(2) 其他程序类型：
```bash
BPF_PROG_TYPE_SOCKET_FILTER: 用于挂载在网络套接字上用于网络数据包过滤，也是最早的BPF使用场景
BPF_PROG_TYPE_SCHED_CLS: 用于网络流量控制分类
BPF_PROG_TYPE_SCHED_ACT: 用于网络流量控制动作
BPF_PROG_TYPE_XDP: 用于从设备驱动程序接收路径运行的网络数据包过滤，XDP(eXpress Data Path)程序
BPF_PROG_TYPE_CGROUP_SKB: 用于控制组的网络数据包过滤
BPF_PROG_TYPE_CGROUP_SOCK: 由于控制组的网络包筛选器，它被允许修改套接字选项
BPF_PROG_TYPE_LWT_*: 用于轻量级隧道的网络数据包过滤
BPF_PROG_TYPE_SOCK_OPS: 用于设置套接字参数的程序
BPF_PROG_TYPE_SK_SKB: 用于套接字之间转发数据包的网络包过滤
BPF_PROG_CGROUP_DEVICE: 确定是否允许设备操作
```

# eBPF 验证器(verifier)

eBPF程序的目的之一是降低定制内核的门槛，然而这使得大量并不具备内核开发技能的开发者拥有了编写运行与内核的代码的机会，为了保障内核不被挂起或是崩溃或是其他异常，eBPF的程序有着严格的要求，eBPF验证器负责在加载eBPF程序时对eBPF程序进行一系列检查，通过检查的eBPF程序才会被放行至JIT Compiler进行编译。

![](attachments/Pasted%20image%2020240913145352.png)


## 检查项

- 程序不能有可能无法结束的循环（ 循环在5.3内核中被允许)，但要求必须能够结束）
    
- 程序不能超过最大指令数量（参考 [代码](https://elixir.bootlin.com/linux/latest/source/include/uapi/linux/bpf_common.h)中的BPFMAXINSNS），在linux内核中这个值是4096
    
- 代码中不允许存在unreachable code


# 即时编译器JIT

# helper辅助函数
## 定义
什么是 eBPF 辅助函数？eBPF 辅助函数是内核提供给开发者的接口。

为什么要有 eBPF 辅助函数呢？为什么不能像驱动一样直接调用内核函数呢？
这主要是为了保证系统安全。由于 eBPF 程序运行在内核态，为了防止不当调用内核函数导致系统崩溃或安全漏洞，eBPF 程序只能调用内核提供的 eBPF 辅助函数。

截止目前内核共提供了 210 多个 eBPF 辅助函数，具体详细列表可见内核源码文件：include/uapi/linux/bpf.h。

参见：[# 一文讲解eBPF helper 函数的设计与实现](https://blog.csdn.net/youzhangjing_/article/details/134533095)

## 作用

![](attachments/Pasted%20image%2020240808200434.png)

辅助函数是面向开发者的，提供操作BPF程序和BPF Map的工具类函数。由于内核本身会有不定期更新迭代，如果直接调用内核模块，那天可能就不能用了，因此通过定义和维护BPF辅助函数，由BPF辅助函数来去面对后端的内核函数的变化，对开发者透明，形成稳定API接口。

例如，BPF程序不知道如何生成一个随机数，有一个BPF辅助函数会可以帮你检索并询问内核，完成”给我一个随机数”的任务，或者”从BPF Map中读取某个值”等等。任何一种与操作系统内核的交互都是通过BPF辅助函数来完成的，由于这些都是稳定的API，所以BPF程序可以跨内核版本进行移植。

## 可使用的helper函数
可用的辅助函数可能因BPF程序类型而有所不同，例如，与附加到tc层的BPF程序相比，附加到套接字的BPF程序仅允许调用一部分辅助函数。

下图是部分BPF辅助函数的列表：bpf 的helper 函数都是以小写bpf_开头的。
![](attachments/Pasted%20image%2020240808200303.png)


##  helper 函数分类

![](attachments/Pasted%20image%2020241218173611.png)

### map相关的 helper 函数

![](attachments/Pasted%20image%2020241218173028.png)

### network 相关的 helper 函数

![](attachments/Pasted%20image%2020241218173239.png)


## eBPF 辅助函数的设计
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

### 入参类型

入参类型分为基本类型和扩展类型。扩展类型在基本类型的基础上，添加了空指针类型，即允许入参为空指针。另外，当参数类型为 ARG_PTR_TO_BTF_ID 时，则需要在 struct bpf_func_proto 的成员 argx_btf_id 指明具体的 btf 编号。

注：BTF 编号可以看成内核数据类型的编号，通过该编号可以确定数据类型。
#### 基本类型
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

#### 扩展类型
包含的扩展类型如下表所示：

![](attachments/Pasted%20image%2020240808201432.png)

### 返回值类型

同参数类型类似，返回值类型也分为基本类型和扩展类型。扩展类型也是在基本类型的基础添加了空指针类型。

#### 基本类型

![](attachments/Pasted%20image%2020240808201515.png)

#### 扩展类型

扩展类型是在基本类型的基础上，添加了空指针类型，表示返回值可能是空指针，那么 eBPF verifier 需要考虑针对空指针进行安全验证。

![](attachments/Pasted%20image%2020240808201536.png)

## eBPF 辅助函数的实现

下面以 bpf_perf_event_output 为例介绍 eBPF 辅助函数的实现。eBPF 辅助函数 bpf_perf_event_output 是应用最广泛的一个，其主要功能是将数据通过 perf 缓冲区传送给用户态程序。实现 bpf_perf_event_output 需要完成以下三个步骤：

```text
1. 定义 struct bpf_func_proto 结构体，为 bpf_perf_event_output 辅助函数指定功能函数、参数类型、返回值类型等；
2. 为 bpf_perf_event_output 辅助函数分配唯一的编号；
3. 将 bpf_perf_event_output 与特定的 eBPF 程序类型绑定，以确保只有该类型的程序才能调用该辅助函数。
```
### 定义 struct bpf_func_proto

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

### 添加编号
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

## 应用

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

## 其他
### 辅助函数和系统调用
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


# 用户态系统调用命令
## 命令分类
![](attachments/Pasted%20image%2020241218173458.png)

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



# object pin钉住对象
## ebpf 的 sysfs 接口

`eBPF`还通过`VFS`接口暴露`BPF`程序和`BPF`映射, 文件位置在于`/sys/fs/bpf/`
所以可以用`pinning`模式(类似于`daemon`程序)的用户态程序，持续运行交互`eBPF`程序（即使已经运行结束），当前的`Cillium`项目就是使用的这种方式。



# 尾调用tail calls
## 背景

BPF 提供了一种在内核事件和用户程序事件发生时安全注入代码的能力，这就让非内核开发人员也可以对内核进行控制，但是因为11 个 64 位寄存器和 32 位子寄存器、一个程序计数器和一个 512 字节的 BPF 堆栈空间以及100万条指令（5.1+），递归深度33的固有限制，使得可以实现的逻辑是有限的（非图灵完备）。

内核栈是很宝贵的，一般BPF到BPF的会使用额外的栈帧，尾调用最大的优势就是其复用了当前的栈帧并跳转至另外一个eBPF程序，可以在中看到如下描述：

>  The important detail that it’s not a normal call, but a tail call. The kernel stack is precious, so this helper reuses the current stack frame and jumps into another BPF program without adding extra call frame.

## 限制

eBPF程序都是独立验证的（调用者的堆栈和寄存器中的值被调用者不可访问），所以状态的传递一般可以使用per-CPU map传递，TC还可以使用skb_buff->cb这样的特殊数据项去传递数据；
其次类型相同的 BPF 程序才可以尾调用，而且还要与 JIT 编译器相匹配， 因此一个给定的 BPF 程序 要么是 JIT编译执行，要么是解释器执行（invoke interpreted programs）。



##  尾调用的使用
尾调用的步骤需要用户态和内核态配合，主要由两个部分组成：

**用户态**：
`BPF_MAP_TYPE_PROG_ARRAY`类型的特殊map，存储自定义`index`到`bpf_program_fd`的到映射。

**内核态**：
`bpf_tail_call`辅助函数，其负责跳转到另一个 eBPF 程序，其函数定义是这样的`static long (*bpf_tail_call)(void *ctx, void *prog_array_map, __u32 index)`，ctx是上下文，`prog_array_map`是前面说的`BPF_MAP_TYPE_PROG_ARRAY`类型的map，用于用户态设置跳转程序和用户自定义index的映射，index就是用户自定义索引了。
`bpf_tail_call` 如果运行成功，内核立即运行新eBPF程序的第一条指令（永远不会返回到之前的程序）。
如果跳转的目标程序不存在（即 index 在 `prog_array_map` 中不存在），或者此程序链已达到最大尾调用数(`MAX_TAIL_CALL_CNT`, 定义为32)，则调用可能会失败，如果调用失败，调用者继续执行后续指令。

## 尾调用的优缺点
### 优点
最大的优势如下：

1. 省内核栈空间
2. 用于增加可执行的eBPF程序指令的最大执行数
3. eBPF程序编排

#### 应用范例
**大的ebpf程序的逻辑拆分**
在BMC中有一个eBPF程序中有一个大循环，虽然eBPF程序只有142行，但是字节码已经到了七十多万行，如果不做逻辑拆分会在 verify 阶段被拒绝。


# ebpf 的 lib库

![](attachments/Pasted%20image%2020241218173954.png)

# bpf to bpf calls

# 加固hardening
# offload

# BTF

## 内核数据结构的定义的问题
### 问题
我们来看一个开发 eBPF 程序时最常碰到的问题：内核数据结构的定义。

在安装 BCC 工具的时候，你可能就注意到了，内核头文件 linux-headers-$(uname -r) 也是必须要安装的一个依赖项。这是因为 BCC 在编译 eBPF 程序时，需要从内核头文件中找到相应的内核数据结构定义。这样，你在调用 bpf_probe_read 时，才能从内存地址中提取到正确的数据类型。

编译时依赖内核头文件也会带来很多问题。主要有这三个方面：
- （1）首先，在开发 eBPF 程序时，为了获得内核数据结构的定义，就需要引入一大堆的内核头文件；

- （2）其次，内核头文件的路径和数据结构定义在不同内核版本中很可能不同。因此，你在升级内核版本时，就会遇到找不到头文件和数据结构定义错误的问题；

- （3）最后，在很多生产环境的机器中，出于安全考虑，并不允许安装内核头文件，这时就无法得到内核数据结构的定义。在程序中重定义数据结构虽然可以暂时解决这个问题，但也很容易把使用着错误数据结构的 eBPF 程序带入新版本内核中运行。


### 解决思路

那么，这么多的问题该怎么解决呢？
不用担心，BPF 类型格式（BPF Type Format, BTF）的诞生正是为了解决这些问题。
从**内核 5.2** 开始，只要开启了 **CONFIG_DEBUG_INFO_BTF**，在编译内核时，内核数据结构的定义就会自动内嵌在内核二进制文件 vmlinux 中。
```bash
# cat /boot/config-`uname -r` | grep -i btf
CONFIG_DEBUG_INFO_BTF=y
CONFIG_PAHOLE_HAS_SPLIT_BTF=y
CONFIG_DEBUG_INFO_BTF_MODULES=y
```
并且，你还可以借助下面的命令，把这些数据结构的定义导出到一个头文件中（通常命名为 vmlinux.h）:
```bash
bpftool btf dump file /sys/kernel/btf/vmlinux format c > vmlinux.h
```

如下所示，有了内核数据结构的定义，你在开发 eBPF 程序时只需要引入一个 vmlinux.h 即可，不用再引入一大堆的内核头文件了。

![](attachments/Pasted%20image%2020240910194616.png)

### 应用

借助 BTF、bpftool 等工具，我们也可以更好地了解 BPF 程序的内部信息，这也会让调试变得更加方便。比如，在查看 BPF map映射的内容时，你可以直接看到结构化的数据，而不只是十六进制数值：

```bash
# bpftool map dump id 386
[
  {
      "key": 0,
      "value": {
          "eth0": {
              "value": 0,
              "ifindex": 0,
              "mac": []
          }
      }
  }
]
```


例子：使用`bpftool`工具查看基于BTF编译生成的`BPF`程序
![](attachments/Pasted%20image%2020240913140454.png)

# ebpf的 CO-RE

## 内核升级后eBPF 程序的兼容问题

## 背景
如何让 eBPF 程序在内核升级之后，不需要重新编译就可以直接运行

### ebpf的 CO-RE

eBPF 的一次编译到处执行（Compile Once Run Everywhere，简称 CO-RE）项目借助了 BTF 提供的调试信息。

**核心在于**：
支持将`BPF`程序编译为字节码，保存后分发到其他机器执行, 这样可以避免要求运行环境安装BPF编译器(`LLVM`和`Clang`）

**核心挑战**：

- 不同操作系统内核数据结构的成员的偏移量不同，需要根据不同底层重写访问偏移量（也就意味着要重新编译）
- 不可见的数据结构成员，这要根据不同内核版本、内核配置选项信息以及用户提供的运行时信息来动态调整。
所以，目前集中要解决的就是`BPF`字节码的可重定位/替换（避免需要`llvm`重新编译）。

#### 解决方法 

**解决思路**：
再通过下面的两个步骤，使得 eBPF 程序可以适配不同版本的内核：
- 第一，通过对 BPF 代码中的访问偏移量进行重写，解决了不同内核版本中数据结构偏移量不同的问题；
- 第二，在 libbpf 中预定义不同内核版本中的数据结构的修改，解决了不同内核中数据结构不兼容的问题。

##### 前置条件

BTF 和一次编译到处执行（CO-RE）带来了很多的好处，但你也需要注意这一点：它们都要求比较新的内核版本（>=5.2），并且需要非常新的发行版（如 Ubuntu 20.10+、RHEL 8.2+ 等）才会默认打开内核配置 CONFIG_DEBUG_INFO_BTF。对于旧版本的内核，虽然它们不会再去内置 BTF 的支持，但开源社区正在尝试通过 BTFHub 等方法，为它们提供 BTF 调试信息。

##### 最小化基础依赖
目前，有许多 `BPF`（`eBPF`）初创公司正在构建网络，安全性和性能产品（并且更多未浮出水面的），但是要求客户安装 `LLVM`，`Clang` 和内核头文件依赖（可能消耗超过`100 MB`的存储空间）是一个额外的负担。

`BTF`和`CO-RE`的目标就是：使得`BPF` 工具现在可以是一个轻量的`ELF`二进制文件, 其中包含了预编译的`BPF`字节码，它可以在任何具有 `BTF` 的内核上运行，而不是要求客户安装各种重量级（且脆弱）的依赖项。

例如：（重写的`opensnoop`）

![](attachments/Pasted%20image%2020240913142515.png)

实现原理：
- `BTF` 提供类型信息，以便可以根据需要查询结构偏移量和其他详细信息(无需再重新遍历内核结构)
- `CO-RE`记录需要重写 BPF 程序的哪些部分以及如何重写

> 注：新的 BPF 二进制文件仅在设置了此内核配置选项后才可用 CONFIG_DEBUG_INFO_BTF = y；Ubuntu 20.10 已经将此配置选项设置为默认选项，所有其他发行版都应遵循。
> 现在，随着我们转向带有 BTF 和 CO-RE 的 libbpf C，已经不赞成使用 BCC Python 中的性能工具。



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
# 官方文档、很好
https://docs.ebpf.io/linux/helper-function/bpf_for_each_map_elem/

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

# eBPF-2-实战之编程接口、bcc与bpftrace
https://blog.csdn.net/weixin_43988498/article/details/125113777

# BPF 进阶笔记（一）：BPF 程序（BPF Prog）类型详解：使用场景、函数签名、执行位置及程序示例
https://arthurchiao.art/blog/bpf-advanced-notes-1-zh/

# BPF架构（二）
https://barryx.cn/cilium_bpf-xdp_2/

# [EBPF学习计划](https://davidlovezoe.club/wordpress/archives/862)
https://davidlovezoe.club/wordpress/archives/862

```