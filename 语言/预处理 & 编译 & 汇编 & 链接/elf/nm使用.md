```table-of-contents
```

# 介绍
nm命令是linux下自带的GNU Binutils二进制工具集的一员，一般用来检查分析二进制文件、库文件（静态库、动态库）、可执行文件中的符号表，返回二进制文件中各段的信息。nm命令来源于`name`的简写。

该命令用来列出指定文件中的符号（如常用的函数名、变量等，以及这些符号存储的区域）。它显示指定文件中的符号信息，文件可以是对象文件、可执行文件或对象文件库。如果文件中没有包含符号信息，`nm`报告该情况，单不把他解释为出错。


## 主要功能

- 列出目标文件中的符号（函数、变量等）
- 显示符号的类型和属性
- 帮助分析程序链接问题
- 辅助调试和逆向工程


## 背景知识

### 目标文件(对象文件/二进制文件)、库文件、可执行文件
首先，提到这三种文件，我们不得不提的就是gcc的编译流程：预编译，编译，汇编，链接。
- **目标文件** :常说的目标文件是我们的程序文件(.c/.cpp,.h)经过预编译，编译，汇编过程生成的二进制文件,不经过链接过程，编译生成指令为：
    gcc(g++) -c file.c(file.cpp)  
    将生成对应的file.o文件,file.o即为二进制文件
    
- **库文件**： 分为静态库和动态库，这里不做过多介绍，静态库文件是由多个二进制文件打包而成，生成的.a文件，示例：
    ar -rsc liba.a test1.o test2.o test3.o  
    将test1.o test2.o test3.o三个文件打包成liba.a库文件
    
- **可执行文件**：可执行文件是由多个二进制文件或者库文件(由上所得，静态库文件其实是二进制文件的集合)经过链接过程生成的一个可执行文件。

## 为什么要用到nm

在上述提到的三种文件中，用编辑器是无法查看其内容的(乱码)，所以当我们有这个需求(例如debug，查看内存分布的时候)去查看一个二进制文件里包含了哪些内容时，这时候就将用到一些特殊工具，linux下的nm命令就可以完全胜任(同时还有objdump和readelf工具，这里暂不作讨论)。

# 使用

## 使用方法

![](attachments/Pasted%20image%2020251023112241.png)

## 参数说明

-A 或-o或 --print-file-name：打印出每个符号属于的文件  
-a或--debug-syms：打印出所有符号，包括debug符号  
-B：BSD码显示  
-C或--demangle[=style]：对低级符号名称进行解码，C++文件需要添加  
--no-demangle：不对低级符号名称进行解码，默认参数  
-D 或--dynamic：显示动态符号而不显示普通符号，一般用于动态库  
-f format或--format=format：显示的形式，默认为bsd，可选为sysv和posix  
-g或--extern-only：仅显示外部符号  
-h或--help：国际惯例，显示命令的帮助信息  
-n或-v或--numeric-sort：显示的符号以地址排序，而不是名称排序  
-p或--no-sort：不对显示内容进行排序  
-P或--portability：使用POSIX.2标准  
-V或--version：国际管理，查看版本  
--defined-only：仅显示定义的符号，这个从英文翻译过来可能会有偏差，故贴上原文：
```bash
Display only defined symbols for each object file
```



## 输出说明
### 范例
![](attachments/Pasted%20image%2020251023112605.png)

解析输出信息中各部分所代表的意思吧
- 第一列：符号所在的文件
- 第二列：那一串数字，指的就是地址。
- 第三列：每一个条目前面还有一个字母，类似'U','B','D等等，其实这些符号代表的就是当前条目所对应的内存所在部分
- 第四列：最右边的就是对应的符号内容了

注意：第二列的地址其实是相对于不同数据区的起始地址。在目标文件中指定的地址都是逻辑地址，符号真正的地址需要到链接阶段时进行相应的重定位以确定最终的地址。

```bash
0000000000000000 B g_uninit  
0000000000000000 D str  
0000000000000000 T func1()

他们的地址都是0，难道说0地址同时可以存三种数据？其实不是这样的，按照上面的符号表规则，g_uninit属于.bss段，str属于全局数据区，而func1()属于代码段，这个地址其实是相对于不同数据区的起始地址，即g_uninit在.bss段中的地址是0，以此类推，而.bss段具体被映射到哪一段地址，这属于平台相关，并不能完全确定。
```


**字符串常量**
比如：在全局数据区(D)中存放了`str`指针，那`str`指针指向的字符串放到哪里去了？其实这些字符串内容放在常量区，常量区属于代码区(t)(X86平台，不同平台可能有不同策略)，对应`nm`显示文件的这一部分：
```bash
 00000000000000a0    t _GLOBAL__sub_I_str  
```

### 符号说明

![](attachments/Pasted%20image%2020251023113822.png)

==对于每一个符号来说，其类型如果是小写的，则表明该符号是local的；大写则表明该符号是global(external)的==。

|类型|说明|
|---|---|
|A|绝对符号，链接时不会被改变|
|B/b|未初始化数据段（BSS段）中的符号|
|D/d|已初始化数据段中的符号|
|T/t|代码段中的符号（T表示全局，t表示局部）|
|U|未定义的符号（需要从其他文件链接）|
|W/w|弱符号（weak symbol）|
|R/r|只读数据段中的符号|
|C|公共符号（common symbol）|
|I|间接引用其他符号|


- A 该符号的值是绝对的，在以后的链接过程中，不允许进行改变。这样的符号值，常常出现在中断向量表中，例如用符号来表示各个中断向量函数在中断向量表中的位置。
- B/b 该符号的值出现在非初始化数据段(bss)中。例如，在一个文件中定义全局`static int test`。则该符号`test`的类型为b，位于`bss section`中。其值表示该符号在bss段中的偏移。一般而言，bss段分配于RAM中。
- C 该符号为`common`。`common symbol`是未初始话数据段。该符号没有包含于一个普通`section`中。只有在链接过程中才进行分配。符号的值表示该符号需要的字节数。例如在一个c文件中，定义`int test`，并且该符号在别的地方会被引用，则该符号类型即为C。否则其类型为B。
- D/d 该符号位于初始化数据段中。一般来说，分配到`data section`中。
    
    例如：定义全局`int baud_table[5] = {9600, 19200, 38400, 57600, 115200}`，会分配到初始化数据段中。
    
- G 该符号也位于初始化数据段中。主要用于`small object`提高访问`small data object`的一种方式。
- I 该符号是对另一个符号的间接引用。
- N 该符号是一个`debugging`符号。
- R 该符号位于只读数据区。
    
    - 例如定义全局`const int test[] = {123, 123}`；则`test`就是一个只读数据区的符号。
    - 值得注意的是，如果在一个函数中定义`const char *test = “abc”, const char test_int = 3`。使用nm都不会得到符号信息，但是字符串”abc”分配于只读存储器中，`test`在`rodata section`中，大小为4。
    
- S 符号位于非初始化数据区，用于`small object`。
- T/t 该符号位于代码区`text section`。
- U 该符号在当前文件中是未定义的，即该符号的定义在别的文件中。
    
    例如，当前文件调用另一个文件中定义的函数，在这个被调用的函数在当前就是未定义的；但是在定义它的文件中类型是T。但是对于全局变量来说，在定义它的文件中，其符号类型为C，在使用它的文件中，其类型为U。
    
- V 该符号是一个`weak object`。
- W `The symbol is a weak symbol that has not been specifically tagged as a weak object symbol`.
- ? 该符号类型没有定义


# `nm 可执行文件/库` 和  `nm -D 可执行文件/库`的区别
## 场景
有时候编译一个可执行二进制文件的时候，有可能编译的时候，**既使用了静态库，又使用了同名的动态库**。

但是`ldd 可执行二进制文件`的时候，看到了使用了动态库，无法确定程序中是否定义了同名的函数；如果引入了同名的静态库，那么二进制文件中就会有这些函数的定义，那么程序执行过程中就不会加载动态库。

## 区别
两者的区别在于它们查看的**符号表（Symbol Table）**不同。
- `nm 可执行文件/库`：查看**标准符号表 (`.symtab`)**。
- `nm -D 可执行文件/库`：查看**动态符号表 (`.dynsym`)**。

|**特性**|**nm (默认)**|**nm -D (或 nm --dynamic)**|
|---|---|---|
|**目标符号表**|`.symtab` (标准符号表)|`.dynsym` (动态符号表)|
|**包含的符号类型**|全局符号、**本地静态符号 (static)**、调试符号|仅限**导出的全局符号**和**导入的未定义符号**|
|**信息量**|最详细，非常庞大|较精简，只关注对外接口|
|**运行时依赖**|不需要，运行时不加载到内存|必须有，运行时加载器需要使用|
|**`strip` 命令的影响**|**会被 strip 删除。**删除后，`nm` 将显示 "no symbols"。|**不会被删除。**即使 strip 之后，`nm -D` 依然可以显示内容。|
|**主要使用场景**|调试、开发阶段分析、查看内部实现细节|分析共享库接口、查看依赖关系、分析已发布的(stripped)二进制|



### 标准符号表 (`.symtab`)

标准符号表 (`.symtab`)
- **包含内容最全**：它包含了编译和静态链接过程中产生的所有符号。这包括：
    - 全局函数和变量（Global）。
    - **本地静态函数和变量（Local static）**，即用 `static` 关键字修饰的，仅在当前文件内部可见的符号。
    - 调试信息符号。
        
- **用途**：主要用于静态链接器（`ld` 在构建程序时）、调试器（如 `gdb`）以及开发者进行详细分析。
- **运行时不需要**：当程序真正运行起来后，操作系统加载器（Loader）并不需要这个表。
- **可被删除 (Stripped)**：为了减小程序体积或增加逆向工程难度，可以使用 `strip` 命令将这个表安全地删除。删除后，程序仍然可以正常运行，但 `gdb` 调试会变得困难。



### 动态符号表 (`.dynsym`)
动态符号表 (`.dynsym`)
- **是标准表的子集**：它只包含那些在**程序运行时进行动态链接**所必需的符号。这包括：
    - 该文件**导出（Export）**给其他动态库使用的全局符号（例如，`libc.so` 导出的 `printf`）。
    - 该文件需要从其他动态库**导入（Import）**的未定义符号（U）。
        
- **不包含本地符号**：它通常不包含 `static` 修饰的本地函数或变量。
- **用途**：专供动态链接器（`ld-linux.so`）在程序启动或运行时使用，用于解析不同共享库之间的函数调用和变量引用。
- **运行时必需**：这个表必须被加载到内存中，否则程序无法运行。
- **不可完全删除**：即使使用 `strip` 命令，动态符号表也会被保留下来（除非你破坏了文件结构），因为它对程序的运行至关重要。

## 如何判断二进制中是否有静态库
（1）如果二进制没有使用strip去掉符号表，那么`nm 二进制文件`，看是否存在库中的函数的定义（`-T`、`-t`的标记）。
（2）如果二进制使用了strip去掉符号表，所有二进制自身的符号都看不到了，则无法判断静态库是否导入了二进制中。
（3）如果二进制没有使用strip去掉符号表，但是静态库被strip去掉符号表后导入到二进制，也是无法判断的。


## 使用

### 什么时候使用 `nm 可执行文件/库` (默认模式)

当你的目标是进行深入的开发调试或分析程序的**内部结构**时，使用默认模式。

- **场景一：开发调试，查看本地静态符号。** 你写了一个包含 `static void helper_func()` 的 C 文件。你想知道这个静态函数最终在二进制文件中的地址。`nm -D` 是看不到它的，必须用 `nm`。
    
    - 特征：你会看到很多类型为小写字母的符号（如 `t` 表示本地代码段符号，`d` 表示本地数据段符号）。
        
- **场景二：分析未被 strip 的静态库 (`.a` 文件)。** 静态库通常只包含 `.symtab`，使用 `nm` 查看其中包含哪些对象模块和符号。
    
- **场景三：使用 GDB 之前的地址确认。** 如果你需要配合 GDB 设置断点，并且拥有未 strip 的二进制文件，使用 `nm` 可以获得最精确的所有函数地址。
    

###  什么时候使用 `nm -D 可执行文件/库` (动态模式)

当你的关注点是程序与外界的**交互接口**（动态链接），或者处理的是已经发布的产品级二进制文件时，使用 `-D` 模式。

- **场景一（最常见）：分析一个共享库 (`.so`) 提供了哪些 API。** 你想知道 `libssl.so` 到底导出了哪些函数供别人调用。
    
    - 命令：`nm -D libssl.so | grep ' T '` (查找定义的全局代码符号)
        
- **场景二：分析一个可执行文件依赖了哪些外部函数。** 你想知道你的程序依赖了 `libc` 中的哪些具体函数。
    
    - 命令：`nm -D myprogram | grep ' U '` (查找未定义的符号)
        
- **场景三：分析已经被 `strip` 过的二进制文件。** 对于生产环境中发布的程序，通常都已经被 strip 过了。此时直接运行 `nm myprogram` 会提示 `nm: myprogram: no symbols`。 但是，你依然可以使用 `nm -D myprogram` 来查看它的动态符号表，了解它的基本接口和依赖。这是逆向工程或排查环境问题时的重要手段。
    
- **场景四：解决运行时 "symbol lookup error"。** 当程序运行时报错说找不到某个符号时，你需要用 `nm -D` 去检查相关的动态库是否真的导出了那个符号，以及主程序是否正确导入了它。


## 范例
```c
#include <stdio.h>

static void local_helper() { // 本地静态函数
    printf("Inside local helper\n");
}

void global_api() { // 全局导出函数
    local_helper();
    printf("Inside global API\n");
}
```

```bash
编译成共享库：`gcc -shared -fPIC -o libtest.so test.c`
```

### 使用 `nm libtest.so` (查看所有)
```bash
$ nm libtest.so
0000000000004030 b completed.7326
                 w __cxa_finalize@@GLIBC_2.2.5
0000000000001060 t deregister_tm_clones
00000000000010d0 t __do_global_dtors_aux
0000000000003e10 t __do_global_dtors_aux_fini_array_entry
0000000000003e18 d __dso_handle
0000000000003e20 d _DYNAMIC
0000000000001148 T _fini
0000000000001110 t frame_dummy
0000000000003e08 t __frame_dummy_init_array_entry
00000000000020d0 r __FRAME_END__
0000000000001128 T global_api
0000000000004000 d _GLOBAL_OFFSET_TABLE_
                 w __gmon_start__
0000000000002028 r __GNU_EH_FRAME_HDR
0000000000001000 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
0000000000001115 t local_helper
                 U puts@@GLIBC_2.2.5
0000000000001090 t register_tm_clones
0000000000004030 d __TMC_END__
```

输出会包含很多行，你可以找到：

- `0000000000001139 T global_api` (T 表示全局代码段符号)
    
- `0000000000001119 t local_helper` (t 表示本地代码段符号，小写)
    
- 以及许多以 `_` 开头的系统内部符号。

### 使用 `nm -D libtest.so` (查看动态)
```bash
$ nm -D libtest.so
                 w __cxa_finalize
0000000000001148 T _fini
0000000000001128 T global_api
                 w __gmon_start__
0000000000001000 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 U puts
```



# 应用


## `nm -u `查看未定义的符号
当遇到"undefined reference"错误时，可以使用 `nm` 检查哪些符号未定义：

```bash
nm -u my_program.o
```
输出要少得多，通常只包含：

- `0000000000001139 T global_api`
    
- `U puts` (因为 printf 可能会被优化为 puts，U 表示未定义，需要从 libc 导入)
    
- **注意：你在这里看不到 `local_helper`。**

### 执行 `strip libtest.so` 之后
- `nm libtest.so` -> 显示 "no symbols"。
- `nm -D libtest.so` -> 依然显示 `global_api` 和 `puts`。

```bash
$ nm libtest.so
nm: libtest.so: no symbols

$ nm -D libtest.so
                 w __cxa_finalize
0000000000001148 T _fini
0000000000001128 T global_api
                 w __gmon_start__
0000000000001000 T _init
                 w _ITM_deregisterTMCloneTable
                 w _ITM_registerTMCloneTable
                 U puts
```




## 查看符号大小并按大小排序

![](attachments/Pasted%20image%2020251023115423.png)

第一列：表示地址
第二列：表示大小。比如：`kucl_id`在 `utils.c` 中定义为：`static const char *kucl_id;`。因此是8个字节，然后是`b` 标识。
第三列：表示符号的类别
第四列：表示符号。

## `nm -A`查看静态库、动态库、ELF二进制可执行文件、.o文件的符号表

### 查看二进制文件/对象文件的符号
![](attachments/Pasted%20image%2020251023115035.png)


### 查看静态库

![](attachments/Pasted%20image%2020251023115801.png)


### 查看动态块的符号
```bash
nm -D libexample.so
```

![](attachments/Pasted%20image%2020251023115958.png)

### 查看可执行文件

```bash
$ nm -A kstats
kstats:                 U access@@GLIBC_2.2.5
kstats:                 U asprintf@@GLIBC_2.2.5
kstats:0000000000412868 B __bss_start
kstats:0000000000412a20 b buf.9764
kstats:00000000004129e0 b buf.9769
kstats:0000000000409570 T cJSON_AddArrayToObject
kstats:0000000000408ec0 T cJSON_AddBoolToObject
kstats:0000000000408d80 T cJSON_AddFalseToObject
kstats:0000000000408550 T cJSON_AddItemReferenceToArray
kstats:0000000000408640 T cJSON_AddItemReferenceToObject
kstats:0000000000408340 T cJSON_AddItemToArray
kstats:00000000004083a0 T cJSON_AddItemToObject
kstats:0000000000408480 T cJSON_AddItemToObjectCS
kstats:0000000000408b00 T cJSON_AddNullToObject
...
kstats:0000000000404b70 T kucl_tool_obj_get
kstats:00000000004128e0 b kucl_tool_objs_list
kstats:0000000000404b50 T kucl_tool_register_obj
kstats:0000000000402590 T kucl_trace_printf
kstats:0000000000412920 b kucl_ustack_log_root.10085
kstats:0000000000412980 b kucl_ustack_root.10081
kstats:0000000000405fc0 T kucl_vsnprintf
kstats:000000000040acb0 T __libc_csu_fini
kstats:000000000040ac40 T __libc_csu_init
kstats:                 U __libc_start_main@@GLIBC_2.2.5
kstats:                 U localtime@@GLIBC_2.2.5
kstats:000000000040a540 T log2_ceil
kstats:0000000000402470 T main
kstats:                 U malloc@@GLIBC_2.2.5
kstats:                 U memcpy@@GLIBC_2.14
kstats:0000000000404500 t mem_info_do_cmd
kstats:00000000004044e0 t mem_info_help
kstats:0000000000402460 t mem_info_init
kstats:                 U memset@@GLIBC_2.2.5
kstats:                 U mkdir@@GLIBC_2.2.5
kstats:000000000040a3f0 T mkdir_p
kstats:                 U opendir@@GLIBC_2.2.5
kstats:                 U open@@GLIBC_2.2.5
kstats:0000000000412880 B optind@@GLIBC_2.2.5
kstats:0000000000404bd0 T parse_args
kstats:0000000000406d40 t parse_string.isra.0
kstats:0000000000407420 t parse_value
kstats:0000000000405560 T print_and_del_cjson
kstats:                 U printf@@GLIBC_2.2.5
kstats:00000000004053d0 T print_json_array_obj_fill_new_obj
kstats:0000000000405290 T print_rdma_qp_flags_info
kstats:0000000000403aa0 t print_sockopt_conn_rep_info
kstats:0000000000403240 T print_sockopt_trace_conn_rep_info
kstats:0000000000406390 t print_string_ptr.part.0
kstats:00000000004065e0 t print_value.part.0
kstats:                 U putchar@@GLIBC_2.2.5
kstats:                 U puts@@GLIBC_2.2.5
kstats:0000000000412220 D rdma_qp_state_strs
kstats:0000000000412820 D rdma_qp_type_strs
kstats:0000000000412300 D rdma_stats_strs
kstats:0000000000412760 D rdma_wc_status_strs
kstats:                 U readdir@@GLIBC_2.2.5
kstats:                 U read@@GLIBC_2.2.5
kstats:                 U realloc@@GLIBC_2.2.5
kstats:0000000000402510 t register_tm_clones
kstats:0000000000402360 t register_trace_dump_kucl_conn_accept
kstats:00000000004023a0 t register_trace_dump_kucl_conn_event_triggered
kstats:00000000004023c0 t register_trace_dump_kucl_conn_recv_buff
kstats:00000000004023d0 t register_trace_dump_kucl_conn_recv_buff_done
kstats:00000000004023b0 t register_trace_dump_kucl_conn_send_buff
kstats:0000000000402380 t register_trace_dump_kucl_epoll_wait_event_from_kernel
kstats:0000000000402390 t register_trace_dump_kucl_epoll_wait_event_from_ready_list
kstats:0000000000402370 t register_trace_dump_kucl_epoll_wait_events
...
```

#### 比较两个版本的二进制文件
```bash
nm old_version > old.txt
nm new_version > new.txt
diff old.txt new.txt
```

## 库或者可执行文件编译是否开启debug(-g)


## 注意
1. 对于剥离（stripped）过的二进制文件，`nm` 可能无法显示有用的信息
2. 不同架构的二进制文件可能需要使用交叉编译工具链中的 `nm`
3. 动态符号（在共享库中）需要使用 `-D` 选项查看
4. 对于C++程序，建议总是使用 `-C` 选项解码符号名称

# 进阶技巧
## 结合其他工具使用
```bash
# 使用objdump查看更详细的符号信息
objdump -t my_program

# 使用readelf查看ELF文件头信息
readelf -s my_program
```


# 参考
```bash

```