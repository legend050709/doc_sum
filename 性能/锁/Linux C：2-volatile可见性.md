```table-of-contents
```

# 背景
## 可见性问题

# volatile的特性
## volatile可以保证可见性
### 可见性问题

### volatile保证可见性的效果

## volatile不能保证原子性
## volatile没有内存屏障功能
volatile加在变量前面，可以告诉编译器，此变量可能被意外的修改，所以要求编译器每次获取改值的时候，总是从内存拿最新的那份。
可以看到，volatile 本身就是在抑制编译器的正向优化，这就使得加上volatile关键字的程序性能就会偏低，这还是其次；
**更重要的事情是 volatile 并不能保证CPU不会重排它，它和内存栅栏是完全不同的概念。**



# 其他
## volatile修饰结构体时，结构体的成员也是volatile的吗
```c
struct A {
    int data;
};
volatile A a;
const A b;

```
答案是结构体内所有的都是volatile，引用c++标准里的一句话：
```bash
[Note: volatile is a hint to the implementation to avoid aggressive optimization involving the object because the value of the object might be changed by means undetectable by an implementation. See 1.9 for detailed semantics. In general, the semantics of volatile are intended to be the same in C + + as they are in C. ]
```
这里大体可以理解为一个对象是volatile，那对象里所有的成员也都是volatile。

# 小结
## 内核不建议使用 volatile 关键字
参考：[Why the "volatile" type class should not be used](https://www.kernel.org/doc/Documentation/process/volatile-considered-harmful.rst)

# 参考
```bash

# 高性能程序volatile的错误使用
https://mp.weixin.qq.com/s/tYZmMUxJnp_xEZpTJZ20xg

# # Volatile and memory barriers
https://jpbempel.github.io/2015/05/26/volatile-and-memory-barriers.html
  
### Volatile, Memory Barriers, and Load/Store Reordering
https://systemtbe.blogspot.com/2014/05/volatile-memory-barriers-and-loadstore.html


# Memory Model and Synchronization Primitive - Part 1: Memory Barrier
https://www.alibabacloud.com/blog/597460
```