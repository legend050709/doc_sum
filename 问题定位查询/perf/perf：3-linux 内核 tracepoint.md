```table-of-contents
```

# linux内核的tracepoint
 linux内核的 tracepoint 是一个基于静态检测的 Linux 内核事件源。
静态检测描述的是添加到源代码中的**硬编码**的软件检测点：
（1）Linux 的内核静态检测技术称为tracepoint。
（2）用户空间软件的静态检测技术称为用户静态定义跟踪（USDT）。
USDT被软件库（例如libc）用来检测库调用

## 介绍
跟踪点是放置在内核代码中较重要位置的硬编码检测点。内核开发者在内核函数中的特定逻辑位置处，有意放置了这些插装点，然后使这些跟踪点会被编译到内核的二进制文件中。例如，在系统调用、调度程序事件、文件系统操作和磁盘I/O的开始和结束处都有跟踪点。

## 原理
tracepoint是预先在函数的插入点中插桩，当执行到函数的插入点，则执行插桩函数，进而触发与插入点预先绑定的probe函数，probe函数可以是一个或者多个，probe函数可以定义为任意的行为，从而可以起到对函数内部观测的作用。

当系统执行到 tracepoint 点时，会执行 tracepoint 上的我们注册的 probe 函数（可以注册多个 probe 函数），类似于printk，输出当前 tracepoint 上下文的环境信息，只是**不是输出到终端，而是 ring buffer，这样我们在用户层通过debugfs(tracefs)接口就可以读取 ring buffer 中的信息，了解到内核中当前运行的状态**。



# 参考
```bash
# Linux tracepoint 简介
https://blog.csdn.net/weixin_45030965/article/details/128402502
```