```table-of-contents
```

# 内核模块概述
动态加载的模块包括树内模块和树外(OOT)模块。
in-tree 模块是 Linux 内核树的内部，即**它们已经是内核的一部分，与内核版本同步迭代，称为 intree **；

**采用单独的仓库管理，可以在大版本的基础上快速迭代小版本，称为 out-of-tree(OOT)** 。树外模块（out-of-tree）是 Linux 内核树的外部。它们通常是为开发和测试目的编写的，例如测试树级或处理不兼容的内核模块的新版本。

# out-of-tree
## loading out-of-tree module taints kernel 问题
### 问题描述
使用insmod命令加载编写的驱动模块时，出现提示信息：`loading out-of-tree module taints kernel`。不过，模块还是能够被加载。并且卸载后再次加载时，该提示信息没有再次出现。然而整个系统重启后再加载模块，仍然会出现该提示信息。也就是说，在linux的一次运行期间，加载自己编写的驱动模块时，出现了上述提示信息。

### 原因分析

提示信息中的`taint`是**污染**的意思，整个提示信息的意思是**加载树外模块污染内核**。先简单说一下内核污染，当内核受到污染意味着内核**处于社区不支持的状态**，并且内核提供的某些功能可能会被禁用。
此时，如果内核运行出现问题，内核开发者是不会理会的。

为什么要搞这样一个机制呢？
简单点说，有一个对linux感兴趣的同学下载了kernel的源码并移植到自己的开发板上，然后自己写驱动，并加载到内核。之后的一个时间点，假如内核运行出现了问题，此时该同学是不应该向内核开发者反应问题的。因为很有可能内核本身没问题，而是这个同学自己写的驱动存在问题，导致了内核的崩溃。
**内核开发者仅仅只审核了位于内核源码树中的代码，因而只对源码树中的代码负责**。换句话说，一个被污染的内核出现问题可能不是内核的bug；一个没有被污染的内核的错误报告更可能蕴含内核bug。有了这个机制，内核开发者就可以确定哪些错误报告是需要处理的，不然查半天发现不是自己的问题，这就耽误工夫了。

> 显然，本文所述的内核被污染的原因是加载了树外模块，也就是加载自己写的驱动，不在内核源码树中。


### 内核被污染的原因 总结
- 加载非GPL兼容的内核模块
- staging驱动程序的使用，它们是内核源代码的一部分，但尚未经过全面测试
- 使用内核源代码未包含的树外(out-of-tree)模块
- 强制加载不是为当前内核版本构建的模块
- 某些严重错误，例如machine check exceptions（MCE）和kernel oopses

![](attachments/Pasted%20image%2020240703144745.png)

> oops 可能是由内核本身引起的，也可能是某些进程试图让内核违反在系统上能做的事以及它们被允许做的事。oops 将生成一个崩溃签名crash signature，这可以帮助内核开发人员找出错误并提高代码质量。一般 oops 没什么大不了的。有些 oops 很严重，会导致系统恐慌(system panic)。从技术上讲，系统恐慌(system panic)是 oops 的一个子集（即更严重的 oops）。

### 内核是怎么知道这个模块是树外的？

![](attachments/Pasted%20image%2020240703145811.png)

我们在编译驱动模块的Makefile中使用M=$(PWD)来指定驱动源码所在的目录，内核的顶层Makefile在检查到M非空时，会设置KBUILD_EXTMOD变量，最终导致内核的编译体系不会在modulename.mod.c中添加MODULE_INFO(intree, "Y");，也就是说不会给我们自己的驱动打上属于树内的标记。

参考：[Marking loadable kernel module as in-tree](https://stackoverflow.com/questions/42954020/marking-loadable-kernel-module-as-in-tree)

### 解决方法
大多数情况下，我们可以忽略内核污染的情况。事实上，尽管加载内核时会有上述提示（`loading out-of-tree module taints kernel `），但终究成功加载了模块，驱动也能工作。

强迫症患者可能非要寻找一个解决方法，那么我们对症下药，可以使用这么几种解决方案：

(1) 向内核提交patch，让内核的开发者将你的驱动并入内核源码树（对于只是学习驱动的情况，这个方法并不合适）
(2) 自己把驱动程序拷贝到本地的源码树中，并自己添加相应的内核配置项，然后在树内编译驱动模块（保持M为空，这样做也挺麻烦的）
(3) 自己在驱动源码种添加一句MODULE_INFO(intree, "Y");，以欺骗内核本模块为树内模块（最好不要这么搞）




## upstream 和 out-of-tree

upstream 可以理解为 原始作者维护的版本，应该和 in-tree 是一个意思。
out-of-tree 可以为在原始版本中拉取的一个分支，自身进行魔改，然后维护的分支。

# 参考
```bash

# modulename: loading out-of-tree module taints kernel

https://blog.csdn.net/gzxb1995/article/details/105407014?utm_medium=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase&depth_1-utm_source=distribute.pc_relevant_t0.none-task-blog-BlogCommendFromMachineLearnPai2-1.nonecase
```