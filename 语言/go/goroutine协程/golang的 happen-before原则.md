```table-of-contents
```
# 概述
在并发编程中，理解操作之间的执行顺序和可见性是至关重要的。

Go 内存模型通过 Happen-Before 原则来定义并发操作之间的顺序关系。Happen-Before 原则确保了在不同 Goroutine 之间，某些操作的结果对其他操作是可见的。

#  happens-before 是什么
Happens Before 是一个专业术语，与 Go 语言没有直接关系，也就是并非是特有的。

用大白话来讲，其定义是：
> 在一个多线程程序中，假设存在 A 和 B 两个操作，如果 A 操作在 B 操作之前发生（A happens-before B），那么 A 操作对内存的影响将会对执行 B 的线程可见。

# 内存模型
 The Go Memory Model（Go 内存模型）定义的是什么，官方解释如下：
```text
The Go memory model specifies the conditions under which reads of a variable in one goroutine can be guaranteed to observe values produced by writes to the same variable in a different goroutine.
```
首先不要误解这里的内存模型的含义，它并不是指 对象的内存分配、内存回收和内存整理的规范，内存模型的作用是解决多线程环境下并发执行时的内存可见性和一致性问题。
具体点说，就是指在什么条件下，线程A 在读取一个变量的值的时候，能够看到其它线程对这个变量进行的写的结果。

随着不同的CPU架构（x86/amd64、ARM、Power 等）和多级缓存的发展，再加上CPU和编辑器的指令重排，同一个变量的读写变得无法确定，因此编程语言需要给出一系列的规则，来明确多线程同时访问同一个变量的可见性和顺序性，而这种规则称为内存模型。


# golang happen before 的保证
## 单线程中顺序规则
**程序顺序规则**：在一个线程内，程序的执行顺序和它们的代码指定的顺序是一样的，即使编译器或者 CPU 重排了读写顺序，从行为上来看，也和代码指定的顺序一样。

```go
func example() {
    a := 1
    b := 2 // a happens-before b
}
```

## 包的init函数

如果包P1中导入了包P2，则P2中的init函数Happens Before 所有P1中的任何初始化代码。
main函数Happens After 所有的init函数。

## goroutine
Goroutine的创建Happens Before所有此Goroutine中的操作。 
Goroutine的销毁Happens After所有此Goroutine中的操作。

## channel

![](attachments/Pasted%20image%2020241121150145.png)

channel有4个规则：
（1）往 Channel 中的发送操作，happens before 从该 Channel 接收相应数据的动作完成（rece finish）之前，即第 n 个 send 一定 happens before 第 n 个 receive 的完成。

（2）close 一个 Channel 的调用，肯定 happens before 从关闭的 Channel 中读取出一个零值。

（3）对于 unbuffered 的 Channel，也就是容量是 0 的 Channel，从此 Channel 中读取数据的调用一定 happens before 往此 Channel 发送数据的调用（send finish）完成。

(4) 如果 Channel 的容量是 m（m>0），那么，第 n 个 receive 一定 happens before 第 n+m 个 send 的完成(send finish).


## lock
锁有3个规则：

（1）第 n 次的 m.Unlock 一定 happens before 第 n+1 m.Lock 方法的返回；

（2）对于读写锁 RWMutex m，如果它的第 n 个 m.Lock 方法的调用已返回，那么它的第 n 个 m.Unlock 的方法调用一定 happens before 任何一个m.RLock 方法调用的返回（保证只有释放了持有的写锁，那些等待的读请求才能请求到读锁）

（3）对于读写锁 RWMutex m，如果它的第 n 个 m.RLock 方法的调用已返回，那么它的第 k （k<=n）个成功的 m.RUnlock 方法的返回一定 happens before 任意的 m.RUnlockLock 方法调用，只要这些 m.Lock 方法调用 happens after 第 n 次 m.RLock。



## sync.once 函数
**Once函数规则**：
对于 once.Do(f) 调用，f 函数的单次调用一定 happens before 任何 once.Do(f) 调用的返回。

## 传递性规则
如果操作A先行发生于操作B，操作B先行发生于操作C，那就可以得出操作A先行发生于操作C的结论。

# 参考
```bash
```