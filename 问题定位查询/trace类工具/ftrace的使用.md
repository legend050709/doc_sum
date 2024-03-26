```table-of-contents
```
# 背景
有没有这样的一种工具，它可以记录某个内核函数在过去一段时间里，每一次执行的单独耗时，以及它的完整调用链？
这样，可以查询某次**微突发**的具体原因。

# 简介
`perf tools`的`ftrace`具备“记录函数每一次的执行耗时”能力，**ftrace是基于内核的，只能跟踪分析linux内核中各种函数的执行耗时**。


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

```