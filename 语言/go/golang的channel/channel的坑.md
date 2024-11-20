```table-of-contents
```
# channel的常见问题概述
**使用 Channel 最常见的错误是 程序panic 和 goroutine 泄漏，死锁**。

==一般而言，channel的死锁和 goroutine 泄漏 是相同的原因，下面直接就只讨论一个就行了==。

## 死锁

「死锁」就是两个线程互相等待，耗在那里，最后程序不得不终止。go 语言中的「死锁」也是类似的，两个 goroutine 互相等待，导致程序耗在那里，无法继续跑下去。

## goroutine泄露
内存泄漏一般都是通过 OOM(Out of Memory) 告警或者发布过程中对内存的观察发现的，服务内存往往都是缓慢上升，直到被系统 OOM 掉清空内存再周而复始。

在 go 语言中，错误地使用 channel 会导致 goroutine 泄漏，进而导致内存泄漏。

## 程序panic

# chan导致程序 hang 住的问题

## 读写chan不匹配的范例一
etcd issue 6857是一个程序 hang 住的问题：

在异常情况下，没有往 chan 实例中填充所需的元素，导致等待者永远等待。

具体来说，Status 方法的逻辑是生成一个 chan Status，然后把这个 chan 作为元素，传入到其他的chann中，交给其它的 goroutine 去处理和写入数据，最后，从  chan Status 中读取数据。即：创建一个chan，作为元素，通过其他的chan传递出去，其他协程读取后，向该chan写入数据。

不幸的是，如果正好节点（node）停止了，没有 goroutine 去填充这个 chan，会导致方法 hang 在返回的那一行上（下面的截图中的第 466 行）。

![](attachments/Pasted%20image%2020241115145737.png)

解决办法就是，在等待 status chan 返回元素的同时，也检查节点是不是已经停止了（done 这个 chan 是不是 close 了）。

分析：其实，我感觉这个修改还是有问题的。问题就在于，如果程序执行了 466 行，成功地把 c 写入到 Status 待处理队列后，执行到第 467 行时，如果停止了这个节点（node），那么，这个 Status 方法还是会阻塞在第 467 行。你可以自己研究研究，看看是不是这样。

### 小结
上面的问题是：对于一个 chan，异常情况下出现无写（即写退出），导致读卡住。
因此，需要在读卡住的地方，添加异常场景的监控识别，或者添加读超时机制。


## 读写chan不匹配的范例二
etcd issue 11256 是因为 unbuffered chan goroutine 泄漏的问题。

estNodeProposeAddLearnerNode 方法中一开始定义了一个 unbuffered 的 chan，也就是 applyConfChan，然后启动一个子 goroutine，这个子 goroutine 会在循环中执行业务逻辑，并且不断地往这个 chan 中添加一个元素。

TestNodeProposeAddLearnerNode 方法的末尾处会从这个 chan 中读取一个元素。

子 goroutine  在 for 循环中就往此 chan 中写入了一个元素，结果导致 TestNodeProposeAddLearnerNode 从这个 chan 中读取到元素就返回了。
悲剧的是，子 goroutine 的 for 循环还在执行，阻塞在下图中红色的第 851 行，并且一直 hang 在那里。

![](attachments/Pasted%20image%2020241115150708.png)

这个 Bug 的修复也很简单，只要改动一下 applyConfChan 的处理逻辑就可以了：只有子 goroutine 的 for 循环中的主要逻辑完成之后，才往 applyConfChan 发送一个元素，这样，TestNodeProposeAddLearnerNode 收到通知继续执行，子 goroutine 也不会被阻塞住了。

### 小结

上面的问题是：对于一个 无缓冲的chan，多写一读导致的问题。


## 超时导致chan读写不匹配的范例

```go
func process(timeout time.Duration) bool {
    ch := make(chan bool)
    go func() {  
        // 通过sleep, 模拟处理耗时的业务  
        time.Sleep((timeout + time.Second))  
        ch <- true // block  
        fmt.Println("exit goroutine")  
    }()  
    select {  
    case result := <-ch:  
        return result  
    case <-time.After(timeout):  
        return false  
    }
}

```

在这个例子中，process 函数会启动一个 goroutine，去处理需要长时间处理的业务，处理完之后，会发送 true 到 chan 中，目的是通知其它等待的 goroutine，可以继续处理了。

主 goroutine 接收到任务处理完成的通知，或者超时后就返回了。如果发生超时，process 函数就返回了，这就会导致 unbuffered 的 chan 从来就没有被读取。我们知道，unbuffered chan 必须等 reader 和 writer 都准备好了才能交流，否则就会阻塞。超时导致未读，结果就是子 goroutine 就阻塞，永远结束不了，进而导致 goroutine 泄漏。

解决这个 Bug 的办法很简单，就是将 unbuffered chan 改成容量为 1 的 chan，这样第 7 行就不会被阻塞了。

### 小结
上面的问题在于：无缓冲的 channel，select 超时退出，导致channel的读退出，进而导致另外一个协程的channel的写阻塞。

## 总结
chan导致程序 hang 住的问题，本质上是：
由于 生产者（消费者） 所在的 goroutine 已经退出 等原因，导致 **生产者和消费者的个数不匹配**，进而导致 消费者（生产者） 所在的 goroutine 会永远阻塞住。
上面的情况，尤其是==**对于无缓冲的 channel 更加容易出现**==。因此，==实际使用中，推荐尽量使用 buffered channel ，使用起来会更安全。==

### goroutine的退出的注意

（1）当 goroutine 退出「无论是正常退出，还是异常退出」时：
需要考虑该退出的 goroutine 使用的 channel 的生产者（消费者），有没有可能阻塞对应的消费者（生产者）的 goroutine。

（2）尽量使用buffered channel​：
l​使用buffered channel 能减少阻塞发生、即使疏忽了一些极端情况，也能降低 goroutine 泄漏的概率。

# chan导致panic的问题

首先，我们来总结下会 panic 的情况，总共有 3 种：

1. close 为 nil 的 chan；
2. send 已经 close 的 chan；
3. close 已经 close 的 chan。


# 总结



## 经验总结
所以，我给你提供一套选择的方法:

1. 共享资源的并发访问使用传统并发原语(即锁)；
2. 复杂的任务编排和消息传递使用 Channel；
3. 消息通知机制使用 Channel，除非只想 signal 一个 goroutine，才使用 Cond；
4. 简单等待所有任务的完成用 WaitGroup，也有 Channel 的推崇者用 Channel，都可以；
5. 需要和 Select 语句结合，使用 Channel；
6. 需要和超时配合时，使用 Channel 和 Context。



# 参考
```bash

```