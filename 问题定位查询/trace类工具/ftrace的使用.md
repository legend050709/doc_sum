```table-of-contents
```
# 背景
有没有这样的一种工具，它可以记录某个内核函数在过去一段时间里，每一次执行的单独耗时，以及它的完整调用链？
这样，可以查询某次**微突发**的具体原因。


# ftrace 的 介绍
`Ftrace`是`Function Trace`的简写，由 `Steven Rostedt` 开发的，从 2008 年发布的内核 2.6.27 中开始就内置了。

`Ftrace`是一个系统内部提供的追踪工具，旨在帮助内核设计和开发人员去追踪系统内部的函数调用流程。

随着`Ftrace`的不断完善，除了追踪函数调用流程的作用外，还可以用来调试和分析系统的延迟和性能问题，并发展成为一个追踪类调试工具的框架。

除了`Ftrace`外，追踪类调试工具还包括：
![](attachments/Pasted%20image%2020241225194238.png)

# ftrace 的框架图

`Ftrace`的框架图如下

![](attachments/Pasted%20image%2020241225194325.png)


# ftrace 的使用

## debugfs
`ftrace` 通过 `debugfs` 向用户态提供访问接口。
配置内核时激活 `debugfs` 后会创建目录 `/sys/kernel/debug/tracing` ，`debugfs` 文件系统就是挂载到该目录。

**开机启动创建debugfs**
要挂载该目录，需要将如下内容添加到 `/etc/fstab` 文件：
```bash
debugfs  /sys/kernel/debug  debugfs  defaults  0  0
```

**临时创建debugfs**
```bash
mount  -t  debugfs  debugfs  /sys/kernel/debug
```



# trace-cmd工具
`trace-cmd`工具（`ftrace`的一个命令行工具，大大简化`ftrace`的使用）。

比如：`trace-cmd`工具来记录“do_page_fault”缺页中断函数在一段时间范围的执行耗时。
(1) 查看函数的每一次执行耗时，如下所示：
![](attachments/Pasted%20image%2020240314120748.png)
由上可知，`do_page_fault`在内核的每一次执行，耗时都集中在`1us`左右，短时间内未见异常。

(2) 查看`do_page_fault`每次耗时，以及函数内部各子函数的执行耗时
![](attachments/Pasted%20image%2020240314121004.png)
![](attachments/Pasted%20image%2020240314121012.png)

上图直接展示了`do_page_fault`函数内部各子函数的执行耗时。假设当某一次`do_page_fault`耗时异常，那就可以通过日志准确定位到某个异常的子函数，这样对我们定位问题是非常有用的。



# 参考
```bash
# 业务时延检测利器-uftrace
https://www.cnblogs.com/t-bar/p/16898892.html

# 使用 ftrace 跟踪内核
https://linux.cn/article-9838-1.html

# 【一文秒懂】Ftrace系统调试工具使用终极指南
https://www.cnblogs.com/-Donge/p/17981595
```