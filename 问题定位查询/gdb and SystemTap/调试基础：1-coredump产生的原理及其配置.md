```table-of-contents
```

# 产生coredump的信号

coredump是一种基于异步信号的内存信息捕获机制，可以触发coredump的信号有sigbus,sigsegv,sigill,sigabrt等。

# coredump 产生逻辑

在操作系统信号处理流程中会判断当前信号是否是`coredump`信号，如果是`coredump`信号则进入`dump`流程将内存保存到文件中提供给我用户`debug`分析。
`do_signal`是内核信号处理的入口，真正的信号处理逻辑在`get_signal_to_deliver`函数中，这里我们就能看到`coredump`的判断逻辑和入口函数：
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

`coredump`机制目前内核默认都已经集成，主要的配置接口有：

(1) `ulimit`，通过`ulimit -a`查看当前配置，执行`ulimit -c unlimited`将`core file size`改为`ulimited`

(2) `/proc/sys/kernel/core_pattern`，该配置项用于配置生成的core文件路径已经名字，相信信息可以通过`man 5 core`来查询.

```text
/tmp/core-%e-%s-%u-%g-%p-%t 会生成包含如下信息的 core 文件：
/tmp/core-<executable>-<signal>-<uid>-<gid>-<pid>-<timestamp>

各个参数的含义：
%e：导致 core dump 的程序的可执行文件名。
%s：导致 core dump 的信号编号。
%u：导致 core dump 的程序的实际用户 ID。
%g：导致 core dump 的程序的实际组 ID。
%p：导致 core dump 的程序的进程 ID。
%t：core dump 发生时的时间戳（自 epoch 时间以来的秒数）。
```

(3)  使 `core` 文件名称是否带有 `pid`，配置文件 `/proc/sys/kernel/core_uses_pid`。
文件的内容为 1，添加 pid；0为不添加 pid；

(4) `/proc/pid/coredump_filter`,用于配置`dump`文件具体内容，详细信息可以通过`man 5 core`来查询


```bash
默认情况下，/proc/pid/coredump_filter 可能是0x33；
如果想分析使用大页内存的程序，比如DPDK程序，那么分析大页内存，则一般需要设置0xff.
```

![](attachments/Pasted%20image%2020240521175001.png)


(5) gcore主动生产coredump文件：
```bash
gcore -o dpvs_24032_coredump.core 24032
```
![](attachments/Pasted%20image%2020250310141832.png)

```bash
# ll dpvs_24032_coredump.core.24032
-rw-r--r-- 1 root root 48113014944 Mar 10 14:16 dpvs_24032_coredump.core.24032

# cat /proc/cmdline
BOOT_IMAGE=/vmlinuz-4.18.0-2.4.3.3.kwai.x86_64 root=UUID=560d8fce-8372-450f-bbf3-7aff496492ba ro crashkernel=512M console=tty0 console=ttyS1,115200 intel_idle.max_cstate=0 default_hugepagesz=1G hugepagesz=1G hugepages=200 isolcpus=1,2,3,4,5,6,7,8,9,22,23,24,25,26,27,28,29,30

生成的coredump文件，大小是48G，而系统配置的大页内存是200G，实际使用了应该是44G左右。
因此，生成的coredump文件大小应该是和实际使用的大页内存相关。
```

## 注意


## 无法产生coredump的情况
（1）`ulimit -c` 设置的`core`文件大小比较小

（2） `/proc/pid/coredump_filter` 的设置不正确
一个情况下，`/proc/pid/coredump_filter` 都不需要设置。除非是大页内存等。

（3）`/proc/sys/kernel/core_pattern` 对应的文件的权限问题
可执行程序对于 `/proc/sys/kernel/core_pattern` 指定的文件没有写的权限。


# 参考
```c
# 调试 dpdk 应用程序的coredump整理
https://blog.csdn.net/legend050709/article/details/108822350
```