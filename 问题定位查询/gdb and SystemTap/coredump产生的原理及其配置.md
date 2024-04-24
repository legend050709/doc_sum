```table-of-contents
```

# 产生coredump的信号

coredump是一种基于异步信号的内存信息捕获机制，可以触发coredump的信号有sigbus,sigsegv,sigill,sigabrt等。

# coredump 产生逻辑

在操作系统信号处理流程中会判断当前信号是否是coredump信号，如果是coredump信号则进入dump流程将内存保存到文件中提供给我用户debug分析。
do_signal是内核信号处理的入口，真正的信号处理逻辑在get_signal_to_deliver函数中，这里我们就能看到coredump的判断逻辑和入口函数：
```c
	if (sig_kernel_coredump(signr)) { //判断接收到的信号是否是coredump信号
	        if (print_fatal_signals)
	            print_fatal_signal(info->si_signo);
	        proc_coredump_connector(current);
	        /*
	         * If it was able to dump core, this kills all
	         * other threads in the group and synchronizes with
	         * their demise.  If we lost the race with another
	         * thread getting here, it set group_exit_code
	         * first and our do_group_exit call below will use
	         * that value and ignore the one we pass it.
	         */
	        do_coredump(info);   //coredump的实现入口
	}
```
```c
#define SIG_KERNEL_COREDUMP_MASK (\
        rt_sigmask(SIGQUIT)   |  rt_sigmask(SIGILL)    | \
    rt_sigmask(SIGTRAP)   |  rt_sigmask(SIGABRT)   | \
        rt_sigmask(SIGFPE)    |  rt_sigmask(SIGSEGV)   | \
    rt_sigmask(SIGBUS)    |  rt_sigmask(SIGSYS)    | \
        rt_sigmask(SIGXCPU)   |  rt_sigmask(SIGXFSZ)   | \
    SIGEMT_MASK                    )


#define sig_kernel_coredump(sig)    siginmask(sig, SIG_KERNEL_COREDUMP_MASK)
```
如下，通过`man 7 signal `展示了会产生coredump的信号。
![](attachments/Pasted%20image%2020231121114208.png)

```text
在do_coredump中会判断与coredump相关的一些配置信息（是否配置了管道，ulimit是否正确，文件大小以及生成路径和文件名），并且生成对应的core文件。
```
# coredump相关设置

coredump机制目前内核默认都已经集成，主要的配置接口有：

1. ulimit，通过ulimit -a查看当前配置，执行ulimit -c unlimited将core file size改为ulimited
2. /proc/sys/kernel/core_pattern，该配置项用于配置生成的core文件路径已经名字，相信信息可以通过man 5 core来查询
3. /proc/pid/core_filter,用于配置dump文件具体内容，详细信息可以通过man 5 core来查询

![](attachments/Pasted%20image%2020240521175001.png)



## 无法产生coredump的情况
（1）ulimit -c 设置的core文件大小比较小

（2） /proc/pid/core_filter 的设置不正确
一个情况下，/proc/pid/core_filter 都不需要设置。除非是大页内存等。

（3）/proc/sys/kernel/core_pattern 对应的文件的权限问题
可执行程序对于 /proc/sys/kernel/core_pattern 指定的文件没有写的权限。


# 参考
```c

```