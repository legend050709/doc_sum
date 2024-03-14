```table-of-contents
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

```