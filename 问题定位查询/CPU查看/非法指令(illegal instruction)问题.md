# 现象
在一台服务器上无法运行程序 `demo`
```bash
./demo --server --config ../config/config.xml
```
提示直接core掉了:
```bash
Illegal instruction (core dumped)
```
通过 `dmesg -T` 检查系统日志可以看到:
```bash
[Fri Mar  3 17:52:34 2023] traps: demo[107300] trap invalid opcode ip:89813b3 sp:7fff24b82d98 error:0
[Fri Mar  3 17:52:34 2023]  in demo[400000+e49a000]
```


# `Illegal instruction`介绍
所谓 `Illegal instruction` (错误指令)，表示处理器(CPU)收到了一条它不支持的指令。

大多数情况下，是因为程序采用了特定的优化编译，需要依赖一定(新型)的`CPU`指令集。例如，一些近期的`tensorflow`构建都是假设你的CPU支持 `AVX` 指令，而对于早于2011年的处理器或者低端`x86 CPU(Pentium, Celeron, Atom)`都不支持AVX指令集。 
那么，要如何找到这个程序触发的CPU指令是什么呢？

# 触发SIGILL的CPU指令查询


# SIGILL原因小结
## 错误修改代码段
进程代码段中数据是作为指令运行的，如果不小心代码段被错误覆盖，那么CPU可能无法识别对应的代码，进而造成Illegal Instruction。

同样，如果栈被不小心覆盖了，造成返回地址错误、CPU跳转到错误地址，执行没有意义的内存数据，进而造成Illegal Instruction。

进一步可以认为，任何导致代码段错误的问题都可能带来Illegal Instruction。

## CPU指令集演进
CPU的指令集在不停演进，如果将较新指令集版本的程序（即：编译的程序使用较新的指令集）在老版本CPU（不支持较新指令集）的机器上运行，则老版本CPU运行时会有Illegal Instruction问题。

## 工具链Bug
编译器(汇编器或者连接器)自身的bug，有可能生成CPU无法识别的指令。

## 内存访问对齐或浮点格式问题
出现错误的指令可能和访存地址指令有关。 
另外，浮点数的格式是否符合IEEE的标准也可能会有影响。


# 错误排查指南
![](attachments/Pasted%20image%2020240321120341.png)


# 其他
## 查看指定CPU的架构
![](attachments/Pasted%20image%2020240321111102.png)

CPU的架构的分类：
`Complex Instruction Set Computing；CISC）`
`Reduced Instruction Set Computing，RISC）`
![](attachments/Pasted%20image%2020240321111342.png)
## 查看指定CPU的微架构
英特尔官网地址：[http://ark.intel.com/](https://link.zhihu.com/?target=http%3A//ark.intel.com/)，然后输入我们的CPU型号查询
![](attachments/Pasted%20image%2020240320171952.png)
![](attachments/Pasted%20image%2020240320172152.png)
对应的平台全称则为:`broadwell`

下图为`intel`各平台的微架构(march: micro architecture)全称：
![](attachments/Pasted%20image%2020240320172231.png)
![](attachments/Pasted%20image%2020240321112010.png)

## 查看gcc基于当前硬件平台选择的微架构
**查看GCC编译器针对当前硬件平台的`-march`默认值**
```bash
gcc -v -Q --help=target | grep march

这个命令会显示GCC在编译时默认使用的-march选项。
-v选项用于增加编译器的输出信息，
-Q用于显示编译器的内置选项，
--help=target则专门用来获取与目标架构相关的选项。

请注意，你需要确保已经安装了GCC，并且它是在PATH环境变量中可用的。如果系统中有多个版本的GCC，你可能需要指定完整路径来运行特定版本的GCC。

在某些情况下，如果没有明确的-march设置，输出可能会显示为-march=nocona或类似的值，这表示使用的是一个保守的、广泛兼容的架构。如果输出是native，那么这意味着GCC将自动检测并使用当前机器的最佳指令集。
```

**查看GCC编译器针对当前硬件平台的`-march=native`会具体选择哪个CPU微架构(march)**：
```bash
gcc -march=native -Q --help=target 
gcc -march=native -Q --help=target | grep march

注：同一个机器上，一个在宿主机，一个在容器中，不同的GCC版本，输出的结果可能不同。

在这个命令中:
`-march=native`选项，这会让`GCC`针对当前运行的处理器选择最佳的架构。
`-Q`用于显示编译器的内置选项，
`--help=target`则专门用来获取与目标架构相关的选项。
```

执行这个命令后，你应该能看到类似这样的输出，显示了`GCC`认为的“原生”架构名称。
例如，如果你的处理器是`Intel`的`Core i7`，输出可能会是`-march=haswell`或者更具体的型号，**取决于你的`CPU`型号和`GCC`版本**。
请注意，由于**这个命令依赖于实际运行它的硬件，所以在不同的机器上运行可能会得到不同的结果。**




# 参考
```c
# 排查 “Illegal instruction (core dumped)”
https://cloud-atlas.readthedocs.io/zh-cn/latest/kernel/tracing/illegal_instruction_core_dumped.html


# DPDK 21.11.1版本的交叉编译
https://cloud.tencent.com/developer/article/2336661
```