```table-of-contents
```
# 前言

# 组件介绍
## Vmware 和 ubuntu
在 mac上安装一个 ubuntu 操作系统。那么就需要通过`Vmware`安装`ubuntu`虚拟机。
`ubuntu`虚拟机又是作为`qemu`的宿主机，`qemu`模拟器中运行自己编译的`Linux`内核。

## QEMU
QEMU 是一款开源的虚拟机软件，支持多种不同架构的模拟（Emulation）以及配合 kvm 完成当前架构的虚拟化（Virtualization）的特性，是当前最火热的开源虚拟机软件。

## Linux内核镜像
## BusyBox
## gdb

# 整体流程
![](attachments/Pasted%20image%2020250601223013.png)

整体流程如下：
(1) 编译`linux`内核
(2) 编译`busybox`（作用是启动内核后可以运行一些简单的命令操作系统）
(3) 下载`qemu`
(4) 制作文件系统并启动内核；

# 流程详解



## `qemu`运行`Linux`内核
```bash
qemu-system-x86_64 \
      -machine q35 \
      -cpu host \
      -accel kvm \
      -smp 4 \
      -m 4G \
      -kernel ~/linux/arch/x86/boot/bzImage \
      -append "console=ttyS0" \
      -nographic
```

上面命令中各参数的意义：
```bash
	-machine q35 # 使用更新的 q35 机器类型，而非默认的 i440fx 机器类型
	-cpu host # 指定要模拟的 cpu 类型以及该 cpu 支持的特性，host 表示和本机 cpu 一样
	-accel kvm # 使用 kvm 加速
	-smp 4 # 指定 cpu 个数
	-m 4G # 指定内存大小
	-kernel ~/linux/arch/x86/boot/bzImage # 指定要运行的 linux 内核，这个是我们上面构建好的
	-append "console=ttyS0" # 指定内核参数
	-nographic # 使用命令行界面，而非图形化界面
```

在执行完该命令后，内核就开始启动，它会在终端输出各种日志。
但在启动后期，内核会发生 `panic`：
![](attachments/Pasted%20image%2020250601224008.png)
该 `panic` 产生的原因，是内核尝试挂载根文件系统，但是却没有找到，因为我们根本就没有为其指定根文件系统。
修复这个 `panic` 的方式也很简单，就是创建一个根文件系统，然后用 `root` 参数，告诉内核这个根文件系统在哪里。

### 创建根文件系统
内核挂载根文件系统的目的，是为了执行它里面的 `init` 程序，`init` 程序执行成功，就表示内核的启动流程结束，用户态启动流程开始。
所以，我们创建根文件系统的最终目标，就是在它内部的正确位置，安装一个 `init` 程序。
这个 `init` 程序可以是 `linux` 下的任何可执行文件，比如它可以是一个 `shell` 脚本，可以是一个我们自己写的程序，也可以是当前各`linux` 发行版正在使用的 `systemd`。不过这里有一点需要注意，就是程序一般都是有**运行时依赖**的，比如 `linux` 下的绝大部分程序，在运行时都依赖 **glibc** 这个库。

所以，如果我们想要安装在根文件系统的 init 程序正常执行，我们除了在正确位置安装 `init` 程序外，还要安装它的运行时依赖。
这个安装过程，如果用手工来完成的话，是非常麻烦的。
那怎么办呢？
我们可以用 `linux` 发行版的包管理工具来安装 `init` 程序。
包管理工具可以让我们非常方便的把 init 程序，以及它的运行时依赖，以及这些依赖的依赖，都安装到我们指定的根文件系统里，或者说是安装到指定根文件系统所在的硬盘里。


# 参考
```bash
# 搭建基于 qemu 与 gdb 的 Linux 内核开发环境
https://www.less-bug.com/posts/build-a-linux-kernel-development-environment-based-on-qemu-and-gdb/

# 搭建 Linux 内核网络调试环境（vscode + gdb + qemu）
https://zhuanlan.zhihu.com/p/445453676
```