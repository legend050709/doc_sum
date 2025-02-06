```table-of-contents
```

# 介绍

irqbalance这个服务的作用是帮助平衡系统中各个CPU的中断负载。

## 中断介绍
中断是一种硬件设备通知CPU发生了某些事件的机制，比如网卡收到了数据包，或者键盘按下了某个键。 中断处理需要消耗CPU的时间和资源。

如果所有的中断都由一个CPU来处理，那么会导致该CPU过载，而其他CPU空闲，这样就浪费了多核的优势，也影响了系统的性能。


# Irqbalance 原理

Irqbalance 是用户空间用于优化中断的一个工具（后台程序），通过**==周期性==**的（默认10s）统计各个cpu上的中断情况（分析中断的来源和频率），**==动态==**对中断进行再分配，实现各个cpu上中断负载相对均衡。 

另外，irqbalance还会考虑CPU的缓存命中率和NUMA架构的内存访问延迟等因素，尽量将中断分配给最适合处理它的CPU。

事实上它就是 **动态的 smp irq affinity**， 它有很好的动态调节效果，但是对于大量小包的网络环境，irqbalance几乎是无效的。

## 详细实现

`irqbalance` 通过读取 `/proc/interrupts` 文件来获取当前中断的分配情况。
它会定期检查中断负载，并根据预设的策略（如 CPU 的空闲时间、负载等）调整中断的分配。

# Irqbalance的操作
## irqbalance 开机启动进程
在RHEL发行版里这个守护程序默认是开机启用的，那如何确认它的状态呢？

```bash
状态查看： service irqbalance status
干脆取消开机启动：chkconfig irqbalance off
```

```bash
# /usr/lib/systemd/system/irqbalance.service
[Unit]
Description=irqbalance daemon
Documentation=man:irqbalance(1)
Documentation=https://github.com/Irqbalance/irqbalance
ConditionVirtualization=!container
ConditionCPUs=>1

[Service]
EnvironmentFile=-/usr/lib/irqbalance/defaults.env
EnvironmentFile=-/etc/sysconfig/irqbalance
ExecStart=/usr/sbin/irqbalance --foreground $IRQBALANCE_ARGS
ReadOnlyPaths=/
ReadWritePaths=/proc/irq
RestrictAddressFamilies=AF_UNIX
RuntimeDirectory=irqbalance/

[Install]
WantedBy=multi-user.target
```

strace 查看，可以发现，周期行读取`/proc/interrupts` 和 `/proc/stat`。

![](attachments/Pasted%20image%2020241112163129.png)


## irqbalance 命令

![](attachments/Pasted%20image%2020241112162838.png)


irqbalance有一些参数可以用来调整它的行为，比如：

- –-powerthresh：设置多少个CPU可以空闲后进入省电模式。在省电模式下，一个CPU不会参与中断处理，以避免被不必要地唤醒。
- -–hintpolicy：设置如何处理内核对中断亲和性的提示。有效值有exact（总是应用提示），subset（将中断分配给提示的子集），或ignore（完全忽略提示）。
- –-policyscript：设置一个脚本的位置，该脚本会对每个中断执行，并返回一些键值对来指导irqbalance如何管理该中断。比如ban（是否禁止该中断被平衡），balance_level（该中断的平衡级别），或numa_node（该中断所属的NUMA节点）。
- –-banirq：设置一个或多个要禁止平衡的中断号。



## 配置文件`/etc/sysconfig/irqbalance`

```text
# cat /etc/sysconfig/irqbalance
# irqbalance is a daemon process that distributes interrupts across
# CPUs on SMP systems.  The default is to rebalance once every 10
# seconds.  This is the environment file that is specified to systemd via the
# EnvironmentFile key in the service unit file (or via whatever method the init
# system you're using has).

#
# IRQBALANCE_ONESHOT
#    After starting, wait for a minute, then look at the interrupt
#    load and balance it once; after balancing exit and do not change
#    it again.
#
#IRQBALANCE_ONESHOT=

#
# IRQBALANCE_BANNED_CPUS
#    64 bit bitmask which allows you to indicate which CPUs should
#    be skipped when reblancing IRQs.  CPU numbers which have their
#    corresponding bits set to one in this mask will not have any
#    IRQs assigned to them on rebalance.
#
#IRQBALANCE_BANNED_CPUS=

#
# IRQBALANCE_BANNED_CPULIST
#    The CPUs list which allows you to indicate which CPUs should
#    be skipped when reblancing IRQs. CPU numbers in CPUs list will
#    not have any IRQs assigned to them on rebalance.
#
#      The format of CPUs list is:
#        <cpu number>,...,<cpu number>
#      or a range:
#        <cpu number>-<cpu number>
#      or a mixture:
#        <cpu number>,...,<cpu number>-<cpu number>
#
#IRQBALANCE_BANNED_CPULIST=

#
# IRQBALANCE_ARGS
#    Append any args here to the irqbalance daemon as documented in the man
#    page.
#
#IRQBALANCE_ARGS=
```

**(1) IRQBALANCE_BANNED_CPUS**
指定要从中断负载平衡中排除的CPU。如果你有某些特定的CPU核心不希望irqbalance对其进行负载均衡，你可以在这里指定。





# 手动配置中断亲和性
手动配置中断亲和性，需要停掉 irqbalance, 否则会覆盖掉手动配置。

通过编辑：` /proc/irq/$i/smp_affinity_list` 文件，来手动设置某个中断的亲和性。
而 `cat /proc/interrupts` 可以具体查询某个网卡的的收发包的中断号。

# irqbalance 中的 IRQBALANCE_BANNED_CPUS 和 启动参数中的 isolcpus

一般DPDK的程序在系统方面的设置，会进行如下的配置：
```bash
cat /etc/default/grub
中 比如：

default_hugepagesz=1G hugepagesz=1G hugepages=200 isolcpus=1,2,3,4,5,6,7,8,9,18,19,20,21,22,23,24,25,26
```


```bash
大页内存：
cat /etc/fstab
nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0
```


```bash
中断平衡绑定：
# cat /etc/sysconfig/irqbalance
IRQBALANCE_BANNED_CPUS=7fc03fe

# systemctl status irqbalance

注：十六进制数 `7FC03FE` 转换为二进制为：
7FC03FE = 0111 1111 1100 0000 0011 1111 1110
```

```bash
程序启动：
	./dpvs -- -l 1,2,3,4,5,6,7,8,9,18,19,20,21,22,23,24,25,26 -w xxxx -w xxxx --syslog local5
```

## 对比
（1）`isolcpus` 是为了防止后续 Linux 内核将 某些任务（task）度到这些CPU上。

 （2）`irqbalance` 中的 `IRQBALANCE_BANNED_CPUS` 是为了禁止将某些中断（比如收到包之后的中断）绑定到某些cpu上。正常情况下，物理网口的 接收队列的中断都是绑定的 所有的`CPU`「实际选择的时候，好像还是会选择一个CPU」;
 `IRQBALANCE_BANNED_CPUS`可以防止将中断选择到了这些被禁止的CPU上。


因此，`isolcpus` 不仅仅是收包的中断事件，还有用户态程序的执行的调度CPU。
`irqbalance` 中的 `IRQBALANCE_BANNED_CPUS`只是禁止了中断绑定。

# 其他
## 没有 irqbalance 的情况
如果 `irqbalance` 没有运行，也没有配置手动配置中断的绑定，则 CPU 核 0 通常会处理大多数中断。



# 参考
```bash
## 深度剖析告诉你irqbalance有用吗？
https://blog.yufeng.info/archives/2422
```