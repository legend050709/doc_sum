```table-of-contents
```

# drop_caches 手动释放缓存
## 介绍
![](attachments/Pasted%20image%2020231102114306.png)
![](attachments/Pasted%20image%2020231102114328.png)

## drop_caches对性能的影响
我们知道，对于 Page Cache 而言，是可以通过 drop_cache 来清掉的，很多人在看到系统中存在非常多的 Page Cache 时会习惯使用 drop_cache 来清理它们，但是这样做是会有一些负面影响的，比如说这些 Page Cache 被清理掉后可能会引起系统性能下降。

# 其他
## sync
Linux下的sync 命令将所有未写的系统缓冲区写到磁盘中，包含已修改的 i-node、已延迟的块 I/O 和读写映射文件。

也可以在编写程序的时候，`open` 系统调用的时候，加上 `O_SYNC` 标志，表示这个文件写的操作需要直接刷盘，也就是说每次调用 `write` 之后，文件数据和元数据都会写入磁盘。

> open +  `O_SYNC` 标志 的缺点：
所有写操作的延迟都会大大增加，不建议在频繁写的地方使用。（ `O_DIRECT` 不在考虑范围之内，如果你做的是数据库才可以考虑，不然造成的后果是你无法承受的。）



# 参考
```c
内核层面解析
https://blog.csdn.net/u010039418/article/details/108170523

极客时间： Page Cache回收引起的业务性能问题（++++++++++++++）
https://time.geekbang.org/column/article/278222
[Page Cache管理问题 (5讲) 这个系列都很好]

https://zhuanlan.zhihu.com/p/95813254
```