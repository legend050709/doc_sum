```table-of-contents
```

# free
## 介绍

## 输出说明
![](attachments/Pasted%20image%2020231102120412.png)
```c
# free -m
              total        used        free      shared  buff/cache   available
Mem:         257095      107827      144673        4079        4594      143819
Swap:             0           0           0
```
### 磁盘缓存
在 Linux 系统（实际上是任何操作系统）中，磁盘读写都是有缓存的，因为这种缓存往往有利于系统的读写加速，毕竟我们大部分场景下遇到的都是多读少写，因此，用暂时用不到的内存来当缓存，空间换时间是非常值得的。

- **缓存的作用原理**
1》读文件
当你读了一个文件，Linux 会先检查内存中的缓存有没有对应的内容，没有才会去读磁盘上的内容，然后会先将磁盘上的内存读到内存中，再返回给用户。这样，下一次读的时候，就不用再次从磁盘中读了，这样就会大大减少文件读的时间。

2》写文件
如果你这时候往这个文件中写了新的内容，Linux 会往缓存中写，而不是直接往磁盘里写，这样，你写文件的时间就会大大减少。只是，写过的缓存会被 Linux 标记为脏了，也就是所谓的内存脏页，Linux 会周期性地收集所有内存脏页，排序整理，然后往磁盘中真正写入，这就是所谓的回写（writeback），也可以叫做刷盘。

> 注：缓存带来的问题：
> 比如突然掉电时，造成了内存中的脏页来不及刷入磁盘，导致数据丢失。

### buffer && cache
在Linux 2.4的内存管理中，buffer指Linux内存的：Buffer cache。
cache指Linux内存中的：Page cache。一般呢，是这么解释两者的。 
```text
A buffer is someting that has yet to be ‘written’ to disk. 
A cache is someting that has been ‘read’ from the disk and stored for later use.
```

在Linux 2.6之后Linux将他们统一合并到了Page cache作为文件层的缓存。
而buffer则被用作block层的缓存。
block层的缓存是什么意思呢，你可以认为一个buffer是一个physical disk block在内存的代表，用来将内存中的pages映射为disk blocks，这部分被使用的内存被叫做buffer。

buffer里面的pages，指的是Page cache中的pages，所以，buffer也可以被认为Page cache的一部分。
cache 只会缓存文件的读写, 磁盘的读写用 buffer (文件系统和磁盘块设备)。

- 理解：
**buffer/cache 是应对大量写或者碎片写而使用的缓冲区。
buffer主要用来标记, 实际上写的区域还是 cache 里的 pages , 只不过 buffer 记录了哪些 pages 被修改了, 这样在系统回写的时候能提高效率**
**buffer/cache 是为了快速命中, 快速读取数据而使用的加速区, 热点数据会被存放于此**

## 使用


# 参考
```c

```