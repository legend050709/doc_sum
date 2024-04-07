```table-of-contents
```

# buffer和cache的输出
## 范例
`free` 命令中的输出：
![](attachments/Pasted%20image%2020231102120412.png)
```c
# free -m
              total        used        free      shared  buff/cache   available
Mem:         257095      107827      144673        4079        4594      143819
Swap:             0           0           0
```

`vmstat` 中的输出：
![](attachments/Pasted%20image%2020240315150420.png)

# buffer && cache的理解
Buffer 和 Cache 可能不太好区分。
从字面上来说，Buffer 是**缓冲区**，而 Cache 是**缓存**，两者都是数据在内存中的临时存储。那么，你知道这两种“临时存储”有什么区别吗？

注：今天内容接下来的部分，Buffer 和 Cache 我会都用英文来表示，避免跟文中的“缓存”一词混淆。而文中的“缓存”，则通指内存中的临时存储。

# 总结
**Cache 和 Buffer的作用**：
为了协调 CPU 与磁盘间的性能差异，Linux 还会使用 Cache 和 Buffer ，分别把文件和磁盘读写的数据缓存到内存中。
Buffer 是对**磁盘数据**的缓存，而 Cache 是**文件数据**的缓存，它们既会用在读请求中，也会用在写请求中。


# 参考
```c
# 基础篇：怎么理解内存中的Buffer和Cache？
https://time.geekbang.org/column/article/74633
```