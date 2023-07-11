# df
# du
# iostat
# iotop
通过 iotop 来定位 IO高的进程。

# pt-ioprofile
## 介绍
通过 pt-ioprofile定位负载来源文件。pt-ioprofile的原理是对某个pid附加一个strace进程进行IO分析。


## 安装

## 使用
![](attachments/Pasted%20image%2020230809153408.png)
从上图可以看出IO负载的主要来源是sbtest （sysbench的IO bound OLTP测试）。
并且压力主要集中在读取上。

## 风险
注：pt-ioprofile是一种侵入性工具，不应在生产服务器上使用pt-ioprofile。
```c
However, it works by attaching strace to the process using ptrace(), which will make it run very slowly until strace detaches. In addition to freezing the server, there is also some risk of the process crashing or performing badly after strace detaches from it, or indeed of strace not detaching cleanly and leaving the process in a sleeping state. As a result, this should be considered an intrusive tool, and should not be used on production servers unless you are comfortable with that.
```


# 常见操作
## 查看哪个进程对于哪个文件的读写IO较高
## 磁盘中占用最大的文件的查找
# 参考
```c
https://cloud.tencent.com/developer/article/1718267
```
