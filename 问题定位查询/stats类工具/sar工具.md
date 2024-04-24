```table-of-contents
```

# 常用方法
## sar -n DEV

## sar -P ALL 

### 作用
**查看 每个 core的 cpu 使用情况**。

> 注： sar 无法展示每个 CPU的 中断的情况。 而是将  sys 和 si、hi 这些都放入到了 ` %system` 这一项中。

 

![](attachments/Pasted%20image%2020240513173028.png)




### 背景

一般情况下，CPU存在多个Core，想要**查看每个Core的 CPU的使用情况(Si, hi, iowait, sys, idle ，usr)**。

就是通过 **top 命令**，然后 `1` 可以看到每个core的CPU使用情况。
但是如果 core的个数很多(比如：开启了超线程，64 物理core就存在了 128个逻辑core)，通过 top 可以无法展示完全所有的 core的CPU使用情况。

![](attachments/Pasted%20image%2020240527150358.png)

其他方式 ：**sar -P ALL  **  或者 **mpstat -P ALL  ** 

![](attachments/Pasted%20image%2020240527150423.png)

![](attachments/Pasted%20image%2020240527150445.png)

### sar 命令输出各core的使用情况

```bash
比如：
sar -P ALL 1 10
```




### mpstat 命令输出各core的使用情况
```bash
mpstat -P ALL 5 2

```

输出信息简单解释：

- %usr：除了nice为负的进程，系统上其他进程在用户空间的CPU运行时间占比
- %nice：nice为负的进程在用户空间的CPU运行时间占比
- %sys：系统上所有进程在内核空间的CPU运行时间占比，但不包括硬中断和软中断所耗的CPU时间
- %iowait：CPU等待磁盘操作的时间占比
- %irq：CPU处理硬中断的时间占比
- %soft：CPU处理软中断的时间占比
- %steal：虚拟CPU等待时间占比
- %guest：虚拟CPU运行时间占比
- %idle：系统空间时间占比


# 参考
```bash

```