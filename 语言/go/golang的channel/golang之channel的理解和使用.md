```table-of-contents
```
# channel简介
## 通道的作用
为什么要有通道呢？ 通道的作用是解决各个gorountine之间通信的问题;

在开始之前可以先记住一个原则，通道必须和gorountine一起使用，一个直观的体现就是必须要和`go`关键字搭配使用。 其实也非常好理解，因为如果只有一个gorountine，那么也用不着通信

## 内置类型
**Channel 类型是 Go 语言内置的类型，你无需引入某个包，就能使用它**。虽然 Go 也提供了传统的并发原语，但是它们都是通过库的方式提供的，你必须要引入 sync 包或者 atomic 包才能使用它们，而 Channel 就不一样了，它是内置类型，使用起来非常方便。

Channel 和 Go 的另一个独特的特性 goroutine 一起为并发编程提供了优雅的、便利的、与传统并发控制不同的方案，并演化出很多并发模式。

# channel的理念

## CSP
CSP 是 Communicating Sequential Process 的简称，中文直译为通信顺序进程，或者叫做交换信息的循序进程，是用来描述并发系统中进行交互的一种模式。

CSP 是 Tony Hoare 在 1978 年发表在 ACM 的一篇论文。论文里指出一门编程语言应该重视 **input 和 output 的原语，尤其是并发编程的代码**。


## 通过共享内存来通信 和 通过通信来共享内存

**执行业务处理的 goroutine 不要通过共享内存的方式通信，而是要通过 Channel 通信的方式分享数据。**

```text
Don’t communicate by sharing memory, share memory by communicating.
```

“communicate by sharing memory”和“share memory by communicating”是两种不同的并发处理模式。

### communicate by sharing memory
“communicate by sharing memory”是传统的并发编程处理方式，就是指，共享的数据需要用锁进行保护，goroutine 需要获取到锁，才能并发访问数据。

通过共享内存来通信是指多个线程或进程直接访问相同的内存区域，它们通过读写这个共享内存区域来进行数据传递和通信。
在这种模式下，各个线程或进程之间可以直接修改共享内存中的数据，实现数据的共享和传递。然而，这种方式需要开发者自行处理数据同步和互斥访问的问题，以避免数据竞争和一致性问题。

### share memory by communicating
“share memory by communicating”则是类似于 CSP 模型的方式，通过通信的方式，一个 goroutine 可以把数据的“所有权”交给另外一个 goroutine（虽然 Go 中没有“所有权”的概念，但是从逻辑上说，你可以把它理解为是所有权的转移）。

通过通信来共享内存是指使用消息传递等通信方式，在不同的线程或进程之间进行数据交换和共享。
在这种模式下，各个线程或进程之间并不直接访问共享内存，而是通过发送消息、使用通道等方式来进行数据交换。
这种方式可以**避免直接操作共享内存带来的数据竞争（data race）和一致性问题**，
通过通信进行数据共享更加安全可靠，也更容易实现并发编程中的数据同步和通信需求。

## 小结
channel的 设计目的 就是为了 避免并发编程中，加锁导致 data race，影响性能。

channel 类似于一个队列，数据可以从一个 goroutine 中发送到 channel，然后从另一个 goroutine 中接收。
channel 可以是有缓冲的，这意味着可以在 channel 中存储一定数量的值，而不仅仅是一个。
如果 channel 是无缓冲的，则发送和接收操作将会同步阻塞，直到有 goroutine 准备好接收或发送数据。


# channel的特点

## 线程安全
 Golang的Channel,发送一个数据到Channel 和 从Channel接收一个数据 都是 原子性的。
 
### 内置同步机制

在channel的底层实现中，所有对channel的操作（包括发送、接收、关闭等）都会被加锁，以防止多个goroutine同时操作channel时出现数据竞争。
Go runtime为每个channel分配了一个`mutex`锁来保护channel的状态，从而保证了在多goroutine并发操作时的线程安全性。

 在向通道发送或接收数据时，会自动进行加锁和解锁操作，确保==每次操作的原子性和线程安全性==。

因此，==存在多个生产者，多个消费者 读写 channel，不需要额外加锁==。

#### 关闭channel的安全性

（1）关闭一个无缓冲区的channel时：
所有在等待接收该channel的goroutine都会被立即唤醒，并且它们会收到`零值`，从而安全退出。

（2）关闭一个有缓冲的channel时：
如果channel中存在缓冲的数据，则消费者协程逐个从中读取到数据，知道数据读取完，后续读取的就是零值。

注：正常情况下，存在一个生产者，多个消费者的时候。向channel中传递一个数据，只有一个消费者可以接收到数据。

### 阻塞特性
- 当通道满了（发送者发送数据时），发送操作会阻塞直到有其他 goroutine 从通道中接收数据。
- 当通道为空（接收者尝试接收数据时），接收操作会阻塞直到有其他 goroutine 向通道中发送数据。
- 这种阻塞特性可以有效避免并发读写冲突，保证了数据操作的线程安全性。



## channel是引用类型
`chan` 是一种类型（**引用类型**）,需要make初始化
`chan int`代表这是一个int型的通道；
既然它是一种类型,那就涉及是引用类型还是值类型，它是引用类型，所以需要make初始化。

```go
ch := make(chan int)  
// 创建一个传输int类型数据的无缓冲的channel
```


## channel中元素类型是任意的

chan 中的元素是任意的类型，所以也可能是 chan 类型，我来举个例子，比如下面的 chan 类型也是合法的：

怎么判定箭头符号属于哪个 chan 呢？
其实，“<-”有个规则，总是尽量和左边的 chan 结合（The `<-` operator associates with the leftmost `chan` possible:）

```go
chan<- chan int  
// chan <-: 表示单向channel，是只写chan；
// 元素类型是： chan int ， 即每个元素还是chan

chan<- <-chan int  
// 元素类型是：<-chan int  

<-chan <-chan int  
// 元素类型是：<-chan int  

chan (<-chan int)
// 元素类型是：<-chan int
```

等价于
```go
chan<- （chan int） // <- 和第一个 chan 结合  
chan<- （<-chan int） // 第一个<-和最左边的 chan 结合，第二个<-和左边第二个 chan 结合  
<-chan（<-chan int） // 第一个<-和最左边的 chan 结合，第二个<-和左边第二个 chan 结合  
chan (<-chan int) // 因为括号的原因，<-和括号内第一个 chan 结合
```
# channel的应用场景
## 数据交流
channel 当作并发的 buffer 或者 queue，解决生产者 - 消费者问题。多个 goroutine 可以并发当作生产者（Producer）和消费者（Consumer）。

## 数据传递
一个 goroutine 将数据交给另一个 goroutine，相当于把数据的拥有权 (引用) 托付出去。

## 信号通知
一个 goroutine 可以将信号 (closing、closed、data ready 等) 传递给另一个或者另一组 goroutine。

## 任务编排
可以让一组 goroutine 按照一定的顺序并发或者串行的执行，这就是编排的功能。

## 锁
利用 Channel 也可以实现互斥锁的机制。


# channel 基础操作

## 创建 channel
前面说了**通道是引用类型**,因此channel 使用之前需要通过 make 创建。

==通过 make，我们可以初始化一个 chan，未初始化的 chan 的零值是 nil==。


```go
unBufferChan := make(chan int) // 1  
bufferChan := make(chan int, N) // 2
```
上面的方式 1 创建的是无缓冲 channel，方式 2 创建的是有缓冲 channel。

注：**chan close之后，其值并不是nil**。

### 注意
**如果使用 channel 之前没有 make，会出现 dead lock 错误。**
至于为什么是 dead lock，下文我们从源码里面看看。
```go
func main() {  
    var x chan int  
    go func() {  
        x <- 1  
    }()  
    <-x  
}
```
结果如下所示：
```go
$ go run channel1.go  
fatal error: all goroutines are asleep - deadlock!  
  
goroutine 1 [chan receive (nil chan)]:  
main.main()  
    /Users/kltao/code/go/examples/channl/channel1.go:11 +0x60  
  
goroutine 4 [chan send (nil chan)]:  
main.main.func1(0x0)
```

## channel 读写操作
```go
ch := make(chan int, 10)  
  
// 读操作  
x <- ch  
  
// 写操作  
ch <- x
```

通信操作符 `<-` 的箭头指示数据流向，箭头指向哪里，数据就流向哪里，它是一个二元操作符，可以支持任意类型。
==`<-`指向channel，表示向channel中写；`<-`来自于channel，表示从channel中读；==

- **（1）向 channel 中添加数据（channel<-data）**;
- **（2）从 channel 中读取数据（data<-channel）**;
data<-channel， 从 channel 中接收数据并赋值给 data。
<-channel，从 channel 中接收数据并丢弃。

### 从channel读
存在三种方式：
```go
(1) <-c
	读取数据并丢弃
(2) v := <-c 
	读取数据并保存到v中
(3) v, ok := <-c
	读取数据并保存到v中，ok可以判断c是否close
```

#### chan中读取到零值
```go
v, ok := <-c
```
接收数据时，还可以返回两个值。
（1）第一个值是返回的 chan 中的元素；
（2）第二个值是 bool 类型，代表是否成功地从 chan 中读取到一个值。
（2.1）如果第二个参数是 false
表示chan 已经被 close 而且 chan 中没有缓存的数据；
这个时候，读取的第一个值是零值。


==因此，如果从 chan （无论有缓冲chan，还是无缓冲chan）读取到一个零值，可能是 sender 真正发送的零值，也可能是 closed 的并且没有缓存元素产生的零值==。

### 向channel写
```go
形如：chan <- x 
	表示将 x 变量写入到 channel 中
```
## 值为nil的chan
没有通过make初始化的chan，其值默认为nil。
nil 是 chan 的零值，是一种特殊的 chan，对值是 nil 的 chan 的发送接收调用者总是会阻塞。close 这样的chan，会导致panic。


## chan 的容量和长度

cap 返回 chan 的容量，len 返回 chan 中缓存的还未被取走的元素数量。


# 关闭 channel

## close channle的原则

有一条广泛流传的关闭 channel 的原则：

```text
don’t close a channel from the receiver side and don’t close a channel if the channel has multiple concurrent senders.
```

不要从一个 receiver 侧关闭 channel；也不要在有多个 sender 时，关闭 channel。

比较好理解，向 channel 发送元素的就是 sender，因此 sender 可以决定何时不发送数据，并且关闭 channel。但是如果有多个 sender，某个 sender 同样没法确定其他 sender 的情况，这时也不能贸然关闭 channel。

但是上面所说的并不是最本质的，最本质的原则就只有一条：
```text
 don’t close (or send values to) closed channels.
```


## close channel和close 文件的区别
通道channel和文件不同，创建后通道可以不用写close，它可以被垃圾回收；

## close channel的使用场景

当一个 chan 没有 sender 和 receiver 时，即不再被使用时，GC 会在一段时间后标记、清理掉这个 chan。

因此，只有在必须告诉接收者不再有需要发送的值时才有必要关闭。

 即：对于 channel 而言，唯一需要 close 的就是我们想通过 close 触发 channel 读事件。
也就是==通过 close 来发送信号==。

发送者可通过 `close` 关闭一个信道来表示没有需要发送的值了。接收者可以通过为接收表达式分配第二个参数来测试信道是否被关闭。

### 消费者如何判断channel是否关闭
若**没有值可以接收且信道已被关闭**，那么在执行完之后 `ok` 会被设置为 `false`。
```go
v, ok := <-ch
```

**ok的结果和含义**：  
- `true`：读到通道数据，不确定是否关闭，可能channel还有保存的数据，但channel已关闭。  
- `false`：通道关闭，无数据读到。


### 小结

关闭 chan 最优雅的方式，就是不要关闭 chan。除非是 通过 close 来发送信号。


## close channel 的注意事项

- **（1）close之后继续close**：
重复关闭 channel 会导致 panic。

- **（2）close之后继续写**：
向关闭的 channel 发送数据会 panic。

- **（3）发送时关闭**：
发送时关闭。
比如：关闭和发送在2个协程中，发送和关闭同时执行。

- **（4）close之后，继续读**
close通道后，即使通道内还有数据，也不影响接收。它会将所有数据读取完毕，在读取完已发送的数据后会返回元素类型的零值(zero value)。
即：从关闭的 channel 读数据不会 panic，读出 channel 中已有的数据之后，再读就是 channel 类似的默认值，比如 chan int 类型的 channel 关闭之后读取到的值为 0。

==（3.1）读取close后的无缓存通道，不管通道中是否有数据，返回值都为默认值 和 false==。

==（3.2）读取close后的有缓存通道，将缓存数据读取完后，再读取返回值为默认值 和 false==。

因此，==应该通过 `v, ok := <-ch` 的方式来判断channel是否关闭，而不是通过是否可以读取到数据的方式==。


- **（5）一般是通过发送方来触发close channel**
因为 channel 的关闭在接收端能感知到，但是发送端感知不到，因此一般只能在发送端主动关闭。
==而且大部分时候可以不执行 close，只需要读写即可==。

- **（6）未初始化的channel的 close**：
未初始化时关闭channel ，会导致 panic。

### close channel 会造成 panic的范例

```go
 // 1.未初始化时关闭
 func TestCloseNilChan(t *testing.T) {
    var errCh chan error
    close(errCh)
    
    // Output:
    // panic: close of nil channel
 }
 ​
 // 2.重复关闭
 func TestRepeatClosingChan(t *testing.T) {
    errCh := make(chan error)
    var wg sync.WaitGroup
    wg.Add(1)
 ​
    go func() {
       defer wg.Done()
       close(errCh)
       close(errCh)
    }()
 ​
    wg.Wait()
    
    // Output:
    // panic: close of closed channel
 }
 ​
 // 3.关闭后发送
 func TestSendOnClosingChan(t *testing.T) {
    errCh := make(chan error)
    var wg sync.WaitGroup
    wg.Add(1)
 ​
    go func() {
       defer wg.Done()
       close(errCh)
       errCh <- errors.New("chan error")
    }()
 ​
    wg.Wait()
    
    // Output:
    // panic: send on closed channel
 }
 ​
 // 4.关闭后发送 或者 发送时关闭
 func TestCloseOnSendingToChan(t *testing.T) {
    errCh := make(chan error)
    var wg sync.WaitGroup
    wg.Add(1)
 ​
    go func() {
       defer wg.Done()
       defer close(errCh)
 ​
       go func() {
          errCh <- errors.New("chan error") // 由于 chan 没有缓冲队列，代码会一直在此处阻塞
       }()
 ​
       time.Sleep(time.Second) // 等待向 errCh 发送数据
    }()
 ​
    wg.Wait()
 ​
    // Output:
    // panic: send on closed channel
 }

```

## 区分从chan中读取默认值和chan关闭

对于上面的第三点，我们需要区分一下：channel 中的值是默认值还是 channel 关闭了。可以使用 ok-idiom 方式，这种方式在 map 中比较常用。
```go
ch := make(chan int, 10)  
...  
close(ch)  
  
// ok-idiom   
val, ok := <-ch  
if ok == false {  
    // channel closed  
}
```
如果 channel 关闭，则 ok 值为 false ，此时val 接收到的是 channel 类型的零值。

## close channel的方法

参考：[如何优雅的关闭channel](https://golang.design/go-questions/channel/graceful-close/)

### 不太优雅的方法
如下，不那么优雅地关闭 channel 的方法：

####  defer-recover 机制
使用 defer-recover 机制，放心大胆地关闭 channel 或者向 channel 发送数据。即使发生了 panic，有 defer-recover 在兜底，这样避免了程序宕机。

```go
func SafeClose(ch chan T) (justClosed bool) {
    defer func() {
        if recover() != nil {
            // The return result can be altered
            // in a defer function call.
            justClosed = false
        }
    }()

    // assume ch != nil here.
    close(ch)   // panic if ch is closed
    return true // <=> justClosed = true; return
}
```


同样的问题是向已关闭的channel发送值。
```go
func SafeSend(ch chan T, value T) (closed bool) {
    defer func() {
        if recover() != nil {
            closed = true
        }
    }()

    ch <- value  // panic if ch is closed
    return false // <=> closed = false; return
}
```

注：上面的关闭 channel 的方法，很是粗暴。通过 recover 来保证多次close channel时，程序不挂掉。

####  sync.Once来保证
使用 sync.Once 来保证只关闭一次。

```go
如下所示：
	封装 channel 和 sync.Once 得到一个新的 自定义的 channel 。

type MyChannel struct {
    C    chan T
    once sync.Once
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.once.Do(func() {
        close(mc.C)
    })
}
```

注：

#### sync.Mutex 加锁来保证

当然，你也可以用 sync.Mutex 避免关闭channel多次。

```go
如下所示：
	封装 channel 和 标记位，锁 为一个新的自定义的channel。

type MyChannel struct {
    C      chan T
    closed bool
    mutex  sync.Mutex
}

func NewMyChannel() *MyChannel {
    return &MyChannel{C: make(chan T)}
}

func (mc *MyChannel) SafeClose() {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    if !mc.closed {
        close(mc.C)
        mc.closed = true
    }
}

func (mc *MyChannel) IsClosed() bool {
    mc.mutex.Lock()
    defer mc.mutex.Unlock()
    return mc.closed
}
```

这种方式可能比较礼貌，但是==并不能避免数据竞争（data race）==。
目前，Go运行机制并不能保证关闭channel和向channel发送值同时执行不会发生数据竞争。如果对同一channel执行通道发送操作的同时调用SafeClose函数，则可能会发生数据竞争（尽管这种数据竞争一般无害的）。

### 优雅关闭的方法
到底应该如何优雅地关闭 channel？

根据 sender 和 receiver 的个数，分下面几种情况：
```text
1. 一个 sender，一个 receiver
2. 一个 sender， M 个 receiver
3. N 个 sender，一个 reciver
4. N 个 sender， M 个 receiver
```
#### 一个sender时(SPSC/SPMC)
对于 1，2，只有一个 sender 的情况就不用说了，直接从 sender 端关闭就好了，没有问题。重点关注第 3，4 种情况。

#### 多个sender，只有一个receiver时（MPSC）
我们**不能为阻止数据传输让接收者关闭数据channel，这样违反了channel关闭的原则**，但是我们可以通过关闭额外的==信号channel==去通知发送者停止发送值。

第 3 种情形下，优雅关闭 channel 的方法是：
```text
the only receiver says “please stop sending more” by closing an additional signal channel。
```

解决方案就是增加一个==传递关闭信号的 channel（信号channel）==，receiver 通过信号 channel 下达关闭数据 channel 指令。senders 监听到关闭信号后，停止发送数据。

**整体流程总结**：
```text
(1) 添加一个 信号channel，用来传递关闭信号
	注：信号channel必须是无缓冲区的channel。
	如果是有缓冲区的channel，那么向信号channel中发送一个消息，存在多个接收者，
	那么只会唤醒一个接收者。

（2）消费者只有一个
	在特定情况下，消费者可以通过close函数，发送关闭信号。多个生产者收到信号，则处理。
	由于是无缓冲区的信号channel，多个生产者都可以同时收到关闭信号。
	注：如果是有缓冲区的信号channel，则关闭信号，可能先被一个协程收到；
```

代码实现，如下所示：
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 100000
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(1)

    // ...
    dataCh := make(chan int)
    // 信号channel: stopCh 必须是无缓冲区channel。
    stopCh := make(chan struct{}) 
        // stopCh is an additional signal channel.
        // Its sender is the receiver of channel
        // dataCh, and its receivers are the
        // senders of channel dataCh.

    // senders
    for i := 0; i < NumSenders; i++ {
        go func() {
            for {
                // The try-receive operation is to try
                // to exit the goroutine as early as
                // possible. For this specified example,
                // it is not essential.
                // try-receive 操作是为了尽早退出 goroutine。
                // 在这个特定的例子中，这并不是必需的。
                select {
                case <- stopCh:
                    return
                default:
                }

                // Even if stopCh is closed, the first
                // branch in the second select may be
                // still not selected for some loops if
                // the send to dataCh is also unblocked.
                // But this is acceptable for this
                // example, so the first select block
                // above can be omitted.
                /*
                即使 `stopCh` 被关闭，在循环中，第二个 `select` 语句中的第一个分支仍然可能不会被选择；比如，随机执行到来 发送到 `dataCh` 到case。
                但对于这个例子来说，这是可以接受的。
                因此上面的第一个 `select` 语句块可以省略。
                */
                select {
                case <- stopCh:
                    return
                case dataCh <- rand.Intn(Max):
                }
            }
        }()
    }

    // the receiver
    go func() {
        defer wgReceivers.Done()

        for value := range dataCh {
            if value == Max-1 {
                // The receiver of channel dataCh is
                // also the sender of stopCh. It is
                // safe to close the stop channel here.
                close(stopCh)
                return
            }

            log.Println(value)
        }
    }()

    // ...
    wgReceivers.Wait()
}
```
正如注释中说的，信号channel的发送者是数据接收者channel。信号channel遵从channel关闭原则，只能被他的发送者关闭。

这里的 stopCh 就是信号 channel，它本身只有一个 sender，因此可以直接关闭它。senders 收到了关闭信号后，select 分支 “case <- stopCh” 被选中，退出函数，不再发送数据。

需要说明的是，上面的代码并没有明确关闭 dataCh。
在 Go 语言中，对于一个 channel，如果最终没有任何 goroutine 引用它，不管 channel 有没有被关闭，最终都会被 gc 回收。所以，在这种情形下，所谓的==优雅地关闭 channel 就是不关闭 channel，让 gc 代劳==。

#### 多个sender，多个receiver时（MPMC）

对于对个sender，多个receiver的场景。这是最复杂的情况。我们不能让任何一个发送者和接收者关闭数据channel。我们也不能让任何一个接收者关闭信号channel通知所有的接收者和发送者结束游戏。其中任何一种方式都打破了关闭原则。然而，我们可以引入一个中间角色去关闭==信号channel==。

优雅关闭 channel 的方法是：
```text
any one of them says “let’s end the game” by notifying a moderator to close an additional signal channel。
```

和第 3 种情况不同，这里有 M 个 receiver，如果直接还是采取第 3 种解决方案，由 receiver 直接关闭 stopCh 的话，就会重复关闭一个 channel，导致 panic。

因此需要增加一个==中间人协程==，M 个 receiver 都向它发送关闭 dataCh 的“请求”，中间人收到第一个请求后，就会直接下达关闭 dataCh 的指令（通过关闭 stopCh，这时就不会发生重复关闭的情况，因为 stopCh 的发送方只有中间人一个）。另外，这里的 N 个 sender 也可以向中间人发送关闭 dataCh 的请求。


**整体流程总结**：
```text
(1) 添加一个中间人协程、信号channel、中间人channel。
	信号channel：
		传递关闭信号；
		注：信号channel必须是无缓冲区的channel。
		如果是有缓冲区的channel，那么向信号channel中发送一个消息，存在多个接收者，
		那么只会唤醒一个接收者。	
	中间人协程：
		接收中间人的消息，
		通过close函数，传递关闭信号；
	中间人channel：
		receiver、worker 给中间人协程发送信号
(2) 多个 receiver 或 多个 worker 通过 中间人channel 给中间人协程发送信号；
(3) 中间人协程从中间人channel中收到信号后，调用close函数关闭信号channel。
(4) 多个 receiver 或 多个 worker 收到 信号channel的关闭信号后，退出处理。
```

代码实现如下：
```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
    "strconv"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 100000
    const NumReceivers = 10
    const NumSenders = 1000

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int)
    
	// 信号channel: stopCh 必须是无缓冲区channel。
    stopCh := make(chan struct{})
        // stopCh is an additional signal channel.
        // Its sender is the moderator goroutine shown
        // below, and its receivers are all senders
        // and receivers of dataCh.
    toStop := make(chan string, 1)
        // The channel toStop is used to notify the
        // moderator to close the additional signal
        // channel (stopCh). Its senders are any senders
        // and receivers of dataCh, and its receiver is
        // the moderator goroutine shown below.
        // It must be a buffered channel.

    var stoppedBy string

    // moderator
    go func() {
        stoppedBy = <-toStop
        close(stopCh)
    }()

    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    // Here, the try-send operation is
                    // to notify the moderator to close
                    // the additional signal channel.
                    select {
                    case toStop <- "sender#" + id:
                    default:
                    }
                    return
                }

                // The try-receive operation here is to
                // try to exit the sender goroutine as
                // early as possible. Try-receive and
                // try-send select blocks are specially
                // optimized by the standard Go
                // compiler, so they are very efficient.
                // 此中单独的 stopCh 放在这里，是为了提前退出；
                // 而不是多个channel的多路判断，多个case发生，不一定选择哪个执行。
                select {
                case <- stopCh:
                    return
                default:
                }

                // Even if stopCh is closed, the first
                // branch in this select block might be
                // still not selected for some loops
                // (and for ever in theory) if the send
                // to dataCh is also non-blocking. If
                // this is unacceptable, then the above
                // try-receive operation is essential.
                /*
                即使 stopCh 被关闭，这个 select 块中的第一个分支在循环中仍然可能没有被选中（理论上可能永远不会被选中），比如 发送到 dataCh 的操作一直被随机选择到。
                如果这种情况不可接受，那么上述的尝试接收（try-receive）操作就是必不可少的。
                */
                select {
                case <- stopCh:
                    return
                case dataCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }

    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func(id string) {
            defer wgReceivers.Done()

            for {
                // Same as the sender goroutine, the
                // try-receive operation here is to
                // try to exit the receiver goroutine
                // as early as possible.
                select {
                case <- stopCh:
                    return
                default:
                }

                // Even if stopCh is closed, the first
                // branch in this select block might be
                // still not selected for some loops
                // (and forever in theory) if the receive
                // from dataCh is also non-blocking. If
                // this is not acceptable, then the above
                // try-receive operation is essential.
                select {
                case <- stopCh:
                    return
                case value := <-dataCh:
                    if value == Max-1 {
                        // Here, the same trick is
                        // used to notify the moderator
                        // to close the additional
                        // signal channel.
                        select {
                        case toStop <- "receiver#" + id:
                        default:
                        }
                        return
                    }

                    log.Println(value)
                }
            }
        }(strconv.Itoa(i))
    }

    // ...
    wgReceivers.Wait()
    log.Println("stopped by", stoppedBy)
}
```

```text
分析：

try-receive 操作：
	如果可读，就读取；否则就利用default 退出；不会造成阻塞。即 try-receive。
select {
case <- stopCh:
	...
default:
}


try-send 操作：
	如果可写，就写入；否则就利用default 退出；不会造成阻塞。即 try-send。
select {
case toStop <- "sender#" + id:
	....
default:
}
```

代码里 toStop 就是中间人的角色，使用它来接收 senders 和 receivers 发送过来的关闭 dataCh 请求。

这里将 toStop 声明成了一个 缓冲型的 channel。这是为了避免当中间人准备从toStop接收信号之前信号丢失。
假设 toStop 声明的是一个非缓冲型的 channel，那么第一个发送的关闭 dataCh 请求可能会丢失。因为无论是 sender 还是 receiver 都是通过 select 语句来发送请求，如果中间人所在的 goroutine 没有准备好，那 select 语句就不会选中，直接走 default 选项，什么也不做。这样，第一个关闭 dataCh 的请求就会丢失。


如果，我们把 toStop 的容量声明成 Num(senders) + Num(receivers)，那样我们就不需要try-send的select块去通知中间人。发送 dataCh 请求的部分可以改成更简洁的形式：

```go
...
toStop := make(chan string, NumReceivers + NumSenders)
...
            value := rand.Intn(Max)
            if value == 0 {
                toStop <- "sender#" + id
                return
            }
...
                if value == Max-1 {
                    toStop <- "receiver#" + id
                    return
                }
...
```

直接向 toStop 发送请求，因为 toStop 容量足够大，所以不用担心阻塞，自然也就不用 select 语句再加一个 default case 来避免阻塞。

可以看到，这里同样没有真正关闭 dataCh，原样同第 3 种情况。

#### 多个receiver，单个sender(SPMC)场景关闭的变形
只有一个sender的时候，我们正常情况下，可以在生产者协程中进行close。
有时候，关闭信号是可以由第三方发出。在这种情况下，我们可以用额外的信号去通知发送者关闭channel。

```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 100000
    const NumReceivers = 100
    const NumThirdParties = 15

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int)
    closing := make(chan struct{}) // signal channel
    closed := make(chan struct{})
    
    // The stop function can be called
    // multiple times safely.
    /* sender 发送了 closed，则第三方协程也不再阻塞，需要退出 */
    stop := func() {
        select {
        case closing<-struct{}{}:
            <-closed
        case <-closed:
        }
    }
    
    // some third-party goroutines
    for i := 0; i < NumThirdParties; i++ {
        go func() {
            r := 1 + rand.Intn(3)
            time.Sleep(time.Duration(r) * time.Second)
            stop()
        }()
    }

    // the sender
    go func() {
        defer func() {
            close(closed)
            close(dataCh)
        }()

        for {
            select{
            case <-closing: return
            default:
            }

            select{
            case <-closing: return
            case dataCh <- rand.Intn(Max):
            }
        }
    }()

    // receivers
    for i := 0; i < NumReceivers; i++ {
        go func() {
            defer wgReceivers.Done()

            for value := range dataCh {
                log.Println(value)
            }
        }()
    }

    wgReceivers.Wait()
}
```


#### 多个sender情况的变体

在上面N发送者的情况，为了坚守channel关闭原则，我们避免关闭channel。然而，有时候，我们必须关闭channel来告诉所有接收者不再发送数据。在这种情况下，我们可以通过引入中间channel，将N-sender情形转化为One-sender情形。中间channel只有一个发送者，所以我们可以通过关闭这个channel来代替关闭原始数据channel。


```go
package main

import (
    "time"
    "math/rand"
    "sync"
    "log"
    "strconv"
)

func main() {
    rand.Seed(time.Now().UnixNano())
    log.SetFlags(0)

    // ...
    const Max = 1000000
    const NumReceivers = 10
    const NumSenders = 1000
    const NumThirdParties = 15

    wgReceivers := sync.WaitGroup{}
    wgReceivers.Add(NumReceivers)

    // ...
    dataCh := make(chan int)     // will be closed
    middleCh := make(chan int)   // will never be closed
    closing := make(chan string) // signal channel
    closed := make(chan struct{})

    var stoppedBy string

    // The stop function can be called
    // multiple times safely.
    stop := func(by string) {
        select {
        case closing <- by:
            <-closed
        case <-closed:
        }
    }
    
    // the middle layer
    go func() {
        exit := func(v int, needSend bool) {
            close(closed)
            if needSend {
                dataCh <- v
            }
            close(dataCh)
        }

        for {
            select {
            case stoppedBy = <-closing:
                exit(0, false)
                return
            case v := <- middleCh:
                select {
                case stoppedBy = <-closing:
                    exit(v, true)
                    return
                case dataCh <- v:
                }
            }
        }
    }()
    
    // some third-party goroutines
    for i := 0; i < NumThirdParties; i++ {
        go func(id string) {
            r := 1 + rand.Intn(3)
            time.Sleep(time.Duration(r) * time.Second)
            stop("3rd-party#" + id)
        }(strconv.Itoa(i))
    }

    // senders
    for i := 0; i < NumSenders; i++ {
        go func(id string) {
            for {
                value := rand.Intn(Max)
                if value == 0 {
                    stop("sender#" + id)
                    return
                }

                select {
                case <- closed:
                    return
                default:
                }

                select {
                case <- closed:
                    return
                case middleCh <- value:
                }
            }
        }(strconv.Itoa(i))
    }

    // receivers
    for range [NumReceivers]struct{}{} {
        go func() {
            defer wgReceivers.Done()

            for value := range dataCh {
                log.Println(value)
            }
        }()
    }

    // ...
    wgReceivers.Wait()
    log.Println("stopped by", stoppedBy)
}
```


### 小结

```text
don’t close a channel from the receiver side and don’t close a channel if the channel has multiple concurrent senders.
```
更本质的原则：
```text
don’t close (or send values to) closed channels.
```

（1）只有一个发送者时：
只能在发送端关闭channel，并且是唯一的发送者。
不要在接受者端关闭channel，也不要在有多个并发发送者的情况下关闭channel。

（2）存在多个发送者时：
发送者不要来关闭 channel（因为多个发送者都在发送，且不可能同时关闭多个发送者，否则会造成重复关闭），而是使用专门的 stop channel（即信号channel）。

（2.1）发送者和接收者多对一时，接收者关闭 stop channel；
（2.2）多对多时，由任意一方关闭 stop channel，双方监听 stop channel 终止后及时停止发送和接收。

## 判断channel是否 close

### 有问题的channel是否close的判断
```go
func IsClosed(ch <-chan T) bool {
    select {
    case <-ch:
        return true
    default:
    }

    return false
}

func main() {
    c := make(chan T)
    fmt.Println(IsClosed(c)) // false
    close(c)
    fmt.Println(IsClosed(c)) // true
}
```

分析：
（1）首先，IsClosed 函数是一个有副作用的函数。
每调用一次，都会读出 channel 里的一个元素，改变了 channel 的状态。这不是一个好的函数，干活就干活，还顺手牵羊！

（2）其次，IsClosed 函数返回的结果仅代表调用那个瞬间，并不能保证调用之后会不会有其他 goroutine 对它进行了一些操作，改变了它的这种状态。

例如，IsClosed 函数返回 false，但这时有另一个 goroutine 关闭了 channel，而你还拿着这个过时的 “channel 未关闭”的信息，向其发送数据，就会导致 panic 的发生。


### 使用for-range退出判断是否关闭

for range 语法会自动判断 channel 是否结束，当channel被发送数据的协程关闭时，range就会结束，接着退出for循环。


#### 使用场景

它在并发中的使用场景是：
当协程==只从1个channel读取数据==，然后进行处理，处理后协程退出。

下面这个示例程序，当in通道被关闭时，协程可自动退出。

```go
go func(in <-chan int) {
    // Using for-range to exit goroutine
    // range has the ability to detect the close/end of a channel
    for x := range in {
        fmt.Printf("Process %d\n", x)
    }
}(inCh)
```

忽略读取的值，只是清空 chan：
```go
for range ch {  
}
```

### 使用 for-select + channel 的多重返回值
**思路**：
for-range 只能从一个channel中读取数据，for-select 可以从多个channel 中读取数据。在 for-select 中，读取 channel事，利用 `,ok`多重返回值 来判断某个channel 是否close。

**错误示例1**
```go
// 错误示例1
func isChanClose(ch chan int) bool {
    _, ok := <- c
}

分析：
	上面是个**错误示例**，因为 ch 中无数据可读时，会阻塞，卡死协程。
```

怎么解决这个问题？用 `select` 和 `<-chan` 来结合可以解决这个问题。

**错误示例2**
```go
// 错误示例2
func isChanClose(ch chan int) bool {
    select {
    case  <- ch:
        return true
    default:
    }
    return false
}

分析：
	ch 关闭的情况下，从ch中读取数据，依然可以读取到，如果缓冲区读取完成之后，继续读取，则读取的是零值。
	ch 未关闭的情况下，如果不存在数据，则走default，返回了false。
```

**正确示例**
```go
// 正确示例：
func isChanClose(ch chan int) bool {
    select {
    case _, received := <- ch:
        return !received
    default:
    }
    return false
}
```

#### 范例

**（1）如果某个通道关闭后，需要退出协程，直接return即可**。
示例代码中，该协程需要从in通道读数据，还需要定时打印已经处理的数量，有2件事要做，所有不能使用for-range，需要使用for-select，当in关闭时，`ok=false`，我们直接返回。

```go
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in:
            if !ok {
                return
            }
            fmt.Printf("Process %d\n", x)
            processedCnt++
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }
    }
}()
```

**(2) 如果某个通道关闭了，不再处理该通道，而是继续处理其他case**，退出是等待所有的可读通道关闭。

我们需要**使用select的一个特征：select不会在nil的通道上进行等待**。这种情况，把只读通道设置为nil即可解决。
> TODO:  这个是不是和从ni的channel中读取是阻塞的，是矛盾的???

```go
go func() {
    // in for-select using ok to exit goroutine
    for {
        select {
        case x, ok := <-in1:
            if !ok {
                in1 = nil
            }
            // Process
        case y, ok := <-in2:
            if !ok {
                in2 = nil
            }
            // Process
        case <-t.C:
            fmt.Printf("Working, processedCnt = %d\n", processedCnt)
        }
 
        // If both in channel are closed, goroutine exit
        if in1 == nil && in2 == nil {
            return
        }
    }
}()

注：此中 for-select 不存在default ???

```

#### 存在的问题
读取channle时，使用`,ok`多重返回值来退出使用for-select协程，解决是当写入数据的通道关闭时，没数据可读时读协程的正常结束。

想想下面这2种场景，`,ok`多重返回值来判断channel是否close，还能适用吗？

（1）接收的协程要退出了，如果它直接退出，不告知发送协程，发送协程将阻塞。

（2）如果又启动了其他的工作协程处理数据，如何通知它退出？


## 其实并不需要 `isChanClose` 函数
==其实并不需要 `isChanClose` 函数==
上面实现的 `isChanClose` 是可以判断出 channel 是否 close，但是适用场景有限，因为可能等你 `isChanClose` 判断的时候返回值 false，你以为 channel 还是正常的，但是下一刻 channel 被关闭了，这个时候往里面“写”数据就又会 panic。因为判断之后还是有时间窗口，所以 `isChanClose` 的适用还是有限。
```go
if isChanClose( c ) {
    // 关闭的场景，exit  
    return
}

// 未关闭的场景，继续执行（可能还是会 panic）
c <- x
```

### 本质
我们换一个思路，你其实并不是一定要判断 channel 是否 close，真正的目的是：**安全的使用 channel，避免使用到已经关闭的 closed channel，从而导致 panic** 。

这个问题的本质上是保证一个事件的时序。

### 处理

官方推荐通过 `context` 来配合使用，我们可以通过一个 ctx 变量来指明 close 事件，而不是直接去判断 channel 的一个状态。

#### 读处理

```go
select {
case <-ctx.Done():
    // ... exit
    return
case v, ok := <-c:
    // do something....
default:
    // do default ....
}

注：
	ctx.Done() 返回一个可读的channel。
```

`ctx.Done()` 事件发生之后，我们就明确不去读 channel 的数据。


#### 写处理

```go
select {
case <-ctx.Done():
    // ... exit
    return
default:
    // push 
    c <- x
}
```
`ctx.Done()` 事件发生之后，我们就明确不写数据到 channel ，或者不从 channel 里读数据，那么保证这个时序即可。就一定不会有问题。


### 小结
我们只需要确保一点：

**（1）触发时序保证**：
一定要先执行触发 ctx.Done() 的事件`「比如，带有cancle的ctx，通过执行cancle函数」`，再去做 close channel 的操作，保证这个时序的才能保证 select 判断的时候没有问题；
只有这个时序，才能保证在获悉到 Done 事件的时候，一切还是安全的；
    
 **（2）条件判断顺序**：
select 的 case 先判断 ctx.Done() 事件，这个很重要哦，否则很有可能先执行了 chan 的操作从而导致 panic 问题；


# channel的状态
## channel存在`3种状态`

1. nil，未初始化的状态，只进行了声明，或者手动赋值为`nil`
2. active，正常的channel，可读或者可写
3. closed，已关闭
>注：==千万不要误认为关闭channel后，channel的值是nil==

## channel的操作和状态的关系

### channel的三种操作
channel可进行`3种操作`：
1. 读
2. 写
3. 关闭

### 不同状态下的操作

![](attachments/Pasted%20image%2020241114142035.png)


把这3种操作和3种channel状态可以组合出`9种情况`：

|操作|nil的channel|正常channel|已关闭channel|
|---|---|---|---|
|<- ch|阻塞|成功或阻塞|缓冲区读完之后，读到零值|
|ch <-|阻塞|成功或阻塞|panic|
|close(ch)|panic|成功|panic|

对于nil通道的情况，也并非完全遵循上表，**有1个特殊场景**：当`nil`的通道在`select`的某个`case`中时，这个case会阻塞，但不会造成死锁。

### 范例
#### 读取close后的channle的零值
```go
 func TestReadFromClosedChan(t *testing.T) {
    var errCh = make(chan error)
 ​
    go func() {
       defer close(errCh)
       errCh <- errors.New("chan error")
    }()
 ​
    go func() {
       for i := 0; i < 3; i++ {
          fmt.Println(i, <-errCh)
       }
    }()
 ​
    time.Sleep(time.Second)
    
    // Output:
    // 0 chan error
    // 1 <nil>
    // 2 <nil>
 }
```

如上所示，创建一个零缓冲的channel，关闭了channel之后，后续读取的是channel中数据类型的零值。
```go
type error interface { 
	Error() string 
}
```


# channel 的底层实现
在探究 channel 源码之前，我们肯定首先需要先找到 channel 在 Golang 的具体实现在哪。因为我们在使用 channel 时，用的是 `<-` 符号，并不能直接在 go 源码中找到其实现。但是 Golang 编译器必然会将 `<-` 符号翻译成底层对应的实现。

我们可以使用 Go 自带的命令: `go tool compile -N -l -S hello.go`, 将代码翻译成对应的汇编指令。
或者，直接可以使用 `Compiler Explorer` 这个在线工具。对于上述示例代码可以直接在这个链接看其汇编结果: [go.godbolt.org/z/3xw5Cj](https://go.godbolt.org/z/3xw5Cj)。如下图：
![](attachments/Pasted%20image%2020231217151356.png)
通过仔细查看以上示例代码对应的汇编指令，可以发现以下的对应关系：

1. channel 的构造语句 `make(chan int)`, 对应的是 `runtime.makechan` 函数
2. 发送语句 `c <- 1`, 对应的是 `runtime.chansend1` 函数
3. 接收语句 `x := <- c`, 对应的是 `runtime.chanrecv1` 函数

以上几个函数的实现都位于 go 源码中的 `runtime/chan.go` 代码文件中。

## Channel需要两个队列实现
一个Channel可以被看作是一个通信通道，用于在不同的进程之间传递数据。在具体的实现中，一个Channel通常需要使用两个队列来实现。这两个队列是发送队列和接收队列。

## 创建channel的实现
channel 的构造语句 `make(chan int)`，将会被 golang 编译器翻译为 `runtime.makechan` 函数, 其函数签名如下：
```go
func makechan(t *chantype, size int) *hchan
```
其中，`t *chantype` 即构造 channel 时传入的元素类型。`size int` 即用户指定的 channel 缓冲区大小，不指定则为 0。该函数的返回值是 `*hchan`。hchan 则是 channel 在 golang 中的内部实现。其定义如下：
```go
type hchan struct {
	qcount   uint           // buffer 中已放入的元素个数
	dataqsiz uint           // 用户构造 channel 时指定的 buf 大小
	buf      unsafe.Pointer // buffer
	elemsize uint16         // buffer 中每个元素的大小
	closed   uint32         // channel 是否关闭，== 0 代表未 closed
	elemtype *_type         // channel 元素的类型信息
	sendx    uint           // buffer 中已发送的索引位置 send index
	recvx    uint           // buffer 中已接收的索引位置 receive index
	recvq    waitq          // 等待接收的 goroutine  list of recv waiters
	sendq    waitq          // 等待发送的 goroutine list of send waiters

	lock mutex
}
```
hchan 中的所有属性大致可以分为三类：

1. buffer 相关的属性。例如 buf、dataqsiz、qcount 等。 当 channel 的缓冲区大小不为 0 时，buffer 中存放了待接收的数据。使用 ring buffer 实现。
2. waitq 相关的属性，可以理解为是一个 FIFO 的标准队列。其中 recvq 中是正在等待接收数据的 goroutine，sendq 中是等待发送数据的 goroutine。waitq 使用双向链表实现。
3. 其他属性，例如 lock、elemtype、closed 等。

`makechan` 的整个过程基本都是一些合法性检测和对 `buffer`、`hchan` 等属性的内存分配。通过简单分析 hchan 的属性，我们可以知道其中有两个重要的组件，`buffer` 和 `waitq`。`hchan` 所有行为和实现都是围绕这两个组件进行的。


# channel 种类

## 缓冲区分类
channel 分为无缓冲 channel 和有缓冲 channel。两者的区别如下：

### 无缓冲chan (unbuffered chan)
发送和接收动作是同时发生的。如果没有 goroutine 读取 channel （<- channel），则发送者 (channel <-) 会一直阻塞。
即： ==生产者和消费者的goroutine，无论谁先执行，谁都会阻塞以等待另一个goroutine执行。最终，发送和接收动作是同时发生的==。

![](attachments/Pasted%20image%2020231215155120.png)

### 有缓冲chan（buffered chan）
缓冲 channel 类似一个有容量的队列。当队列满的时候发送者会阻塞；当队列空的时候接收者会阻塞。
![](attachments/Pasted%20image%2020231215155224.png)


## 读写分类

Channel 类型（为了说起来方便，我们下面都把 Channel 叫做 chan）分为**只能接收**、**只能发送**、**既可以接收又可以发送**三种类型。下面是它的语法定义：

ChannelType = ( “chan” | “chan” “<-” | “<-” “chan” ) ElementType .

把既能接收又能发送的 chan 叫做双向的 chan，把只能发送和只能接收的 chan 叫做单向的 chan。

注：带有“<-”的，表示的是单向的 chan。否则，就是双向的channel。
**这个箭头总是射向左边的，元素类型总在最右边。如果箭头指向 chan，就表示可以往 chan 中塞数据；如果箭头远离 chan，就表示 chan 会往外吐数据**。

### 单向channel
#### 只读channel
```go
<-chan int // 只能从 chan 接收 int
```
#### 只写channel
```go
chan<- struct{} // 只能向channel中发送 struct{}
注：struct{}是数据类型
```

### 双向channel：可读可写channel


# 无缓冲channel
## 定义
**如果缓冲大小设置为 0 或者不设置，channel 为无缓冲类型**。

```go
    make(chan Type)   //等价于make(chan Type, 0)
```

## 操作
### 创建
```
var ch = make(chan int)  // 创建一个int类型的channel
cap(ch)                  // ch的容量是0
```

### 发送/存入 
```
ch <- 1  // 存入一个int类型的值
```

### 接收/取出
 ```
x := <-ch  // 取出ch中的值，并赋值给x
```

### 关闭
 ```
close(ch) // 关闭发送方channel，对接收发channel关闭操作会panic
val, ok := <-ch // ok 可用于判断通道是否关闭。
```

## 特征
**无缓冲 channel**：
发送者会阻塞直到接收者接收了发送的值。无缓冲的channel的读写者必须**同时完成发送和接收**。

即：==无缓冲 channel 在消息发送时需要接收者就绪==。
也就是说**receiver的goroutine 和sender的goroutine 必须成对出现**, 不可以是一个goroutine中出现读和写。


```go
package main

import (
	"sync"
	"time"
)

func main() {
	c := make(chan string)

	var wg sync.WaitGroup

	wg.Add(1)
	go func() {
		defer wg.Done()
		c <- `foo`
	}()

	wg.Add(1)
	go func() {
		defer wg.Done()

		time.Sleep(time.Second * 1)
		println(`Message: `+ <-c)
	}()

	wg.Wait()
}
```
第一个协程会在发送消息`foo`时阻塞，原因是接收者还没有就绪。

```go
package main

import (
    "fmt"
)

func f1(in chan int) {
    fmt.Println(<-in)
}

func main() {
    out := make(chan int)
    out <- 2
    go f1(out)
}
```
上诉代码会出现死锁。main 入口 goroutine，通道 out 产生了发送阻塞。



## 无缓冲channel和容量为1的channel对比
```go
c1 := make(chan int) // 无缓冲 
c2 := make(chan int, 1) // 有缓冲
```


### 无缓冲的channel
下面讨论一个简单的场景：A goroutine 向channel (c1)写入一个int，B goroutine 从channel(c1)读走一个int。
```go
c1 <- 1 // A 
<- c1 // B
```

**分析**：
这里的A或B，无论谁先执行，谁都会阻塞以等待另一个goroutine执行。
也就是说往里写得等来读的，从里读得等来写的。
A和B对c1的读写是同步的，直观的理解是A和B对c1的读写是同时发生的，当A对c1写完了，则B从c1中就读完了。
无缓冲的channel的读写者必须**同时完成发送和接收**，而不能串行，显然单协程无法满足。单协程会造成循环等待，会死锁。如下所示：
```go
func main() {
    ch := make(chan int)
    ch <- 1
        <- ch
}
```
### 缓冲为1的channel
继续上面提到的简单场景：A向channel写入一个int，B从channel读走一个int。还是一样的代码：
```go
c2 <- 1   // A
<- c2     // B
```

这里和无缓冲通道不同的地方在于：

==有缓冲的通道并不强制channel的读写者必须**同时完成发送和接收**；
读者只会在没有数据时阻塞，写者只会在没有可用容量时阻塞，这就有点像阻塞队列了==。

# 有缓冲的channel

## 定义
缓冲大小不为0，channel 为有缓冲类型。

## 特征
**有缓存通道**的特点是，有缓存时可以向通道中写入数据后直接返回，缓存中有数据时可以从通道中读到数据直接返回，这时有缓存通道是不会阻塞的。
它阻塞的场景是：
1. 通道的缓存无数据，但执行读通道。
2. 通道的缓存已经占满，向通道写数据。

# channel 阻塞场景
## 写操作阻塞
写操作，什么时候会被阻塞？
**（1）向 nil 通道发送数据会被阻塞**

**（2）向无缓冲 channel 写数据，如果读协程没有准备好，会阻塞**
无缓冲 channel ，必须要有读有写，写了数据之后，必须要读出来，否则导致 channel 阻塞，从而使得协程阻塞而使得协程泄漏

注：一个无缓冲 channel；如果每次来一个请求，就开一个 go 协程往里面写数据，但是一直没有被读取，那么就会导致这个 chan 一直阻塞，使得写这个 chan 的 go 协程一直无法释放从而占用大量的协程。
    
**（3）向有缓冲 channel 写数据，如果缓冲已满，会阻塞**

 有缓冲的 channel，在缓冲 buffer 之内，不读取也不会导致阻塞，当然也就不会使得协程泄漏；
但是如果写数据超过了 buffer容量 还没有读取，那么继续写的时候就会阻塞了。
如果往有缓冲的 channel 写了数据但是一直没有读取就直接退出协程的话，一样会导致 channel 阻塞，从而使得协程阻塞无法退出。

## 读操作阻塞
读操作，什么时候会被阻塞？

**（1）从 nil 通道接收数据会被阻塞**

**（2）从无缓冲 channel 读数据，如果写协程没有准备好，会阻塞**

**（3）从有缓冲 channel 读数据，如果缓冲为空，会阻塞**

## close操作阻塞
close 操作，什么时候会被阻塞？
- close channel 对 channel 阻塞是没有任何效果的，写了数据但是不读，直接 close，还是会阻塞的。


# Select实现channel无阻塞读写
## select 的特性
- select 的 case 分支里面，可以读数据，也可以写数据。**最多只允许有一个 default case（也可以没有default）**，它可以放在 case 列表的任何位置，并且没有任何影响。
- select 可以同时处理多个 channel，如果有同时多个 case 分支可以去处理，比如同时有多个 channel 可以接收数据，那么 Go 会伪随机(pseudo-random)的选择一个 case 处理。如果没有 case 需要处理，则会选择 default 分支去处理。如果没有 default case，则 select 语句会阻塞，直到某个 case 分支可以处理了。
- 每次 select 语句的执行，是会扫描完所有的 case 后才确定如何执行，而不是说遇到合适的 case 就直接执行了。
- 对于 nil channel 上的操作会一直被阻塞，如果没有 default case，只有 nil channel 的 select 会一直被阻塞。
- select 语句和 switch 语句一样，它不是循环，它只会选择一个 case 来处理，如果想一直处理channel，你可以在外面加一个无限的 for 循环，变为 for-select。

```go
for {
    select {
    case c <- x:
        x, y = y, x+y
    case <-quit:
        fmt.Println("quit")
        return
    }
}
```
## 使用Select的default实现无阻塞读
下面示例代码是使用select修改后的无缓冲通道和有缓冲通道的读写，以下函数可以直接通过main函数调用，其中的Ouput的注释是运行结果，从结果能看出，在通道不可读或者不可写的时候，不再阻塞等待，而是直接返回。
```go
// 无缓冲通道读  
func ReadNoDataFromNoBufChWithSelect() {  
	bufCh := make(chan int)  
  
	if v, err := ReadWithSelect(bufCh); err != nil {  
		fmt.Println(err)  
	} else {  
		fmt.Printf("read: %d\n", v)  
	}  
  
	// Output:  
	// channel has no data  
}  
  
// 有缓冲通道读  
func ReadNoDataFromBufChWithSelect() {  
	bufCh := make(chan int, 1)  
  
	if v, err := ReadWithSelect(bufCh); err != nil {  
		fmt.Println(err)  
	} else {  
		fmt.Printf("read: %d\n", v)  
	}  
  
	// Output:  
	// channel has no data  
}  
  
// select结构实现通道读  
func ReadWithSelect(ch chan int) (x int, err error) {  
	select {  
	case x = <-ch:  
		return x, nil  
	default:  
		return 0, errors.New("channel has no data")  
	}  
}  
  
// 无缓冲通道写  
func WriteNoBufChWithSelect() {  
	ch := make(chan int)  
	if err := WriteChWithSelect(ch); err != nil {  
		fmt.Println(err)  
	} else {  
		fmt.Println("write success")  
	}  
  
	// Output:  
	// channel blocked, can not write  
}  
  
// 有缓冲通道写  
func WriteBufChButFullWithSelect() {  
	ch := make(chan int, 1)  
	// make ch full  
	ch <- 100  
	if err := WriteChWithSelect(ch); err != nil {  
		fmt.Println(err)  
	} else {  
		fmt.Println("write success")  
	}  
  
	// Output:  
	// channel blocked, can not write  
}  
  
// select结构实现通道写  
func WriteChWithSelect(ch chan int) error {  
	select {  
	case ch <- 1:  
		return nil  
	default:  
		return errors.New("channel blocked, can not write")  
	}  
}
```
## select的default实现无阻塞的写
有些场景，我们期望往缓冲队列中写入数据的时候，如果队列已满，那么不要进行写阻塞，而是告知队列已满。
```go
var jobChan = make(chan int, 3)

func TryEnqueue(job int) bool {
	select {
	case jobChan <- job:
		fmt.Printf("true\n")  // 队列未满
		return true
	default:
		fmt.Printf("false\n") // 队列已满
		return false
	}
}
```
## 使用Select+超时改善无阻塞读写
使用default实现的无阻塞通道阻塞有一个**缺陷**：
>当通道不可读或写的时候，**存在default，会即可返回**。实际场景，更多的需求是，我们希望，尝试读一会数据，或者尝试写一会数据，如果实在没法读写，再返回，程序继续做其它的事情。

**使用定时器替代default**可以解决这个问题。比如，我给通道读写数据的容忍时间是500ms，如果依然无法读写，就即刻返回，修改一下会是这样：
```go
func ReadWithSelect(ch chan int) (x int, err error) {
    timeout := time.NewTimer(time.Microsecond * 500)

    select {
    case x = <-ch:
        return x, nil
    case <-timeout.C:
        return 0, errors.New("read time out")
    }
}

func WriteChWithSelect(ch chan int) error {
    timeout := time.NewTimer(time.Microsecond * 500)

    select {
    case ch <- 1:
        return nil
    case <-timeout.C:
        return errors.New("write time out")
    }
}

```

# channel的生产者和消费者


# channel的应用范例
## 定时器

与 timer 结合，一般有两种玩法：
（1）实现超时控制
（2）实现定期执行某个任务。

### 超时控制
有时候，需要执行某项操作，但又不想它耗费太长时间，上一个定时器就可以搞定：

```go
select {
	case <-time.After(100 * time.Millisecond):
	case <-s.stopc:
		return false
}
```
等待 100 ms 后，如果 s.stopc 还没有读出数据或者被关闭，就直接结束。这是来自 etcd 源码里的一个例子，这样的写法随处可见。

### 定时任务
```go
func worker() {
	ticker := time.Tick(1 * time.Second) // 周期性定时器
	for {
		select {
		case <- ticker:
			// 执行定时任务
			fmt.Println("执行 1s 定时任务")
		}
	}
}
```
每隔 1 秒种，执行一次定时任务。

## 控制并发数

有时需要定时执行几百个任务，例如每天定时按城市来执行一些离线计算的任务。但是并发数又不能太高，因为任务执行过程依赖第三方的一些资源，对请求的速率有限制。这时就可以通过 channel 来控制并发数。

```go
var limit = make(chan int, 3)

func main() {
    // …………
    for _, w := range work {
        go func() {
	        /*
		        w() 中访问速率受限的资源；
		        通过有缓冲的channel。
		        w()的操作在 channel的入队和出队之间，来控制了 资源访问的并发数。
				
				在执行 w() 之前，先要从 channel 中拿“许可证”，拿到许可证之后，才能执行 w()，并且在执行完任务，要将“许可证”归还。这样就可以控制同时运行的 goroutine 数。
	        */
            limit <- 1
            w()
            <-limit
        }()
    }
    // …………
}
```

分析：
这里，`limit <- 1` 放在 func 内部而不是外部，原因是：
> 如果在外层，就是控制系统 goroutine 的数量，可能会阻塞 for 循环(影响主协程)，影响业务逻辑。

另外，需要注意的是，如果 w() 发生 panic，那“许可证”可能就还不回去了，因此需要使用 defer 来保证。

## 停止信号

channel 用于停止信号的场景还是挺多的，经常是关闭某个 channel 或者向 channel 发送一个元素，使得接收 channel 的那一方获知道此信息，进而做一些其他的操作。

详细，参见上面的讨论。

## 解耦生产方和消费方
### 范例一
服务启动时，启动 n 个 worker（消费方），作为工作协程池，这些协程工作在一个 `for {}` 无限循环里，从某个 channel 消费任务并执行任务：

```go
func main() {
	taskCh := make(chan int, 100)
	go worker(taskCh)

    // 塞任务
	for i := 0; i < 10; i++ {
		taskCh <- i
	}

    // 等待 1 小时 
	select {
	case <-time.After(time.Hour):
	}
}

func worker(taskCh <-chan int) {
	const N = 5
	// 启动 5 个工作协程
	for i := 0; i < N; i++ {
		go func(id int) {
			for {
				task := <- taskCh
				fmt.Printf("finish task: %d by worker %d\n", task, id)
				time.Sleep(time.Second)
			}
		}(i)
	}
}
```

5 个工作协程在不断地从工作队列里取任务，生产方只管往 channel 发送任务即可，解耦生产方和消费方。

程序输出，如下所示：
```shell
finish task: 1 by worker 4
finish task: 2 by worker 2
finish task: 4 by worker 3
finish task: 3 by worker 1
finish task: 0 by worker 0
finish task: 6 by worker 0
finish task: 8 by worker 3
finish task: 9 by worker 1
finish task: 7 by worker 4
finish task: 5 by worker 2
```

### 范例二
多生产者、多消费者，利用通道channel 传递数据；利用WaitGroup控制生产者、消费者执行结束。
```go
package main

import (
	"fmt"
	"sync"
)

func main() {
	wgP := &sync.WaitGroup{}
	wgC := &sync.WaitGroup{}
	var num = 5
	ch := make(chan int, num)

	// 消费者
	for i := 0; i < num; i++ {
		wgC.Add(1)
		go Consumer(wgC, ch, i)
	}

	// 生产者
	for i := 0; i < num; i++ {
		wgP.Add(1)
		go Producter(wgP, ch)
	}

	wgP.Wait()

	// 生产者发送结束后, 要关闭通道，否则消费者会阻塞造成协程泄漏
	close(ch)

	wgC.Wait()

}

func Producter(wg *sync.WaitGroup, ch chan int) {
	defer wg.Done()
	for i := 0; i < 10; i++ {
		ch <- i
	}
}

func Consumer(wg *sync.WaitGroup, ch chan int, i int) {
	defer wg.Done()
	for item := range ch {
		fmt.Printf("%d号协程: %d\n", i+1, item)
	}
}


```

执行结果，如下所示：
```go
2号协程: 0
2号协程: 5
2号协程: 6
2号协程: 0
2号协程: 0
2号协程: 0
3号协程: 2
5号协程: 4
5号协程: 7
1号协程: 1
2号协程: 1
2号协程: 0
2号协程: 1
2号协程: 2
2号协程: 2
2号协程: 2
2号协程: 3
2号协程: 3
2号协程: 2
2号协程: 3
2号协程: 9
1号协程: 1
1号协程: 4
1号协程: 5
1号协程: 6
1号协程: 7
1号协程: 8
1号协程: 9
1号协程: 5
5号协程: 8
5号协程: 5
5号协程: 6
5号协程: 7
5号协程: 8
5号协程: 9
3号协程: 1
3号协程: 4
3号协程: 5
3号协程: 6
3号协程: 7
3号协程: 8
3号协程: 9
2号协程: 4
4号协程: 3
1号协程: 4
1号协程: 9
5号协程: 3
3号协程: 6
2号协程: 7
4号协程: 8
```


# channel 的其他用法
## channel 实现并发同步
channel 作为 Go 并发模型的核心思想：==不要通过共享内存来通信，而应该通过通信来共享内存==。

那么在 Go 里面，当然也可以很方便通过 channel 来实现协程的并发和同步了，并且 channel 本身还可以支持有缓冲和无缓冲的，通过 channel + timeout 实现并发协程之间的同步也是常见的一种使用姿势。

## 通道取值
### for-range 读取channel数据
**不管是有缓冲还是无缓冲，都可以使用 for-range 从 channel 中读取数据，并且这个是一直循环读取的**。

for-range 中的 range 产生的迭代值为向 Channel 中发送的值。
（1）如果这个 channel 已经 close 了，那么首先还会继续执行，直到所有值被读取完，然后才会跳出 for 循环。

因此，通过 for-range 读取 chann 数据会比较方便，因为我们只需要读取数据就行了，不需管他的退出，在 close 之后如果数据读取完了会自动帮我们退出。

（2）如果既没有 close ，也没有数据可读，那么就会阻塞到 range 这里，除非有数据产生或者 chan 被关闭了。

（3）如果 channel 是 nil，读取会被阻塞，也就是会一直阻塞在 range 位置。

```go
   ch := make(chan int)

   // 一直循环读取 range 中的迭代值
   for v := range ch {
        // 得到了 v 这个 chann 中的值
        fmt.Println("读取数据：",v)
   }

```

#### 小结
因此，使用 `for-range` 读取channel，要考虑写channel的协程将channel 关闭；
否则，使用 `for-range` 读取channel 有可能会阻塞。

## 单向 channel
单向 channel，顾名思义只能写或读的 channel。
但是仔细一想，只能写的 channel，如果不读其中的值有什么用呢？

其实==单向 channel 主要用在函数声明中==。

比如。
```go
func foo(ch chan<- int) <-chan int {...}
```

foo 的形参是一个只能写的 channel，那么就表示函数 foo 只会对 ch 进行写，当然你传入的参数可以是个普通 channel。
foo 的返回值是一个只能读的 channel，那么表示 foo 的返回值规范用法就是只能读取。

```go
// Done returns a channel which is closed if and when this pipe is closed  
// with CloseWithError.  
func (p *http2pipe) Done() <-chan struct{} {  
    p.mu.Lock()  
    defer p.mu.Unlock()  
    if p.donec == nil {  
        p.donec = make(chan struct{})  
        if p.err != nil || p.breakErr != nil {  
  
            p.closeDoneLocked()  
        }  
    }  
    return p.donec  
}
```
也许你会说这么写在功能上和使用普通的 channel 并不会有什么差别。确实是这样的。但是使用单向 channel 编程体现了一种非常优秀的编程范式：**convention over configuration**，中文一般叫做 **约定优于配置**。这种编程范式在 Ruby 中体现的尤为明显。

### 作用
目的：
1. 使代码更易读、更易维护，
2. 防止只读协程对通道进行写数据，但通道已关闭，造成panic。

### 范例
```go
package main

import (
	"fmt"
)

// 在函数中 ch只能写
func counter(ch chan<- int) {
	for i := 0; i < 6; i++ {
		ch <- i
	}

	close(ch) // 存完后关闭通道（放心不影响取）
}

// 在函数中 ch1只能存
// 在函数中 ch2只能取
func squarer(ch1 chan<- int, ch2 <-chan int) {
	for i := range ch2 {
		ch1 <- i * i
	}

	close(ch1) // ch1存完关闭
}

// 在函数中 ch 只能取
func printer(ch <-chan int) {
	for i := range ch {
		fmt.Println(i)
	}
}

func main() {
	ch1 := make(chan int)
	ch2 := make(chan int)
	go counter(ch1)
	go squarer(ch2, ch1)

	printer(ch2)
}

```

## 使用`chan struct{}`作为信号channel
### 场景
使用channel传递信号，而不是传递数据时。

### 原理
没数据需要传递时，传递空struct。

### 注意
`struct{}` 是一个数据类型，`struct{}{}`才是`struct{}`类型的一个实例。

## 使用channel传递结构体的指针而非结构体
### 原理
channel本质上传递的是数据的拷贝，拷贝的数据越小传输效率越高，传递结构体指针，比传递结构体更高效。

### 用法
```go
reqCh chan *Request  
  
// 好过  
reqCh chan Request
```

## 使用channel传递channel
### 场景
使用场景有点多，通常是用来获取结果。

### 原理
channel可以用来传递变量，channel自身也是变量，可以传递自己。

### 用法
```go
package main  
  
import (  
	"fmt"  
	"math/rand"  
	"sync"  
	"time"  
)  
  
func main() {  
	reqs := []int{1, 2, 3, 4, 5, 6, 7, 8, 9}  
  
	// 存放结果的channel的channel  
	outs := make(chan chan int, len(reqs))  
	var wg sync.WaitGroup  
	wg.Add(len(reqs))  
	for _, x := range reqs {  
		o := handle(&wg, x)  
		outs <- o  
	}  
  
	go func() {  
		wg.Wait()  
		close(outs)  
	}()  
  
	// 读取结果，结果有序  
	for o := range outs {  
		fmt.Println(<-o)  
	}  
}  
  
// handle 处理请求，耗时随机模拟  
func handle(wg *sync.WaitGroup, a int) chan int {  
	out := make(chan int)  
	go func() {  
		time.Sleep(time.Duration(rand.Intn(3)) * time.Second)  
		out <- a  
		wg.Done()  
	}()  
	return out  
}
```




# Channel 的缺点
Channel 的缺点：
**(1) Channel 可能会导致循环阻塞或者协程泄漏**

**(2) Channel 中传递的都是数据的拷贝，可能会影响性能**
但是就目前我们的机器性能来看，这点数据拷贝所带来的 CPU 消耗，大多数的情况下可以忽略。

**(3) Channel 中==传递指针==会导致数据竞态问题（data race/ race conditions）**

data race 指的是多线程并发读写一个变量，对应到Golang中就是多个goroutine同时读写一个变量，这种行为是未定义的，也就是说读变量出来的值很有可能不是写入的值，这个值是任意值都有可能。


## data race 范例

```go
package main

import (
    "fmt"
    "runtime"
    "time"
)

var i int64 = 0

func main() {
    runtime.GOMAXPROCS(2)
    go func() {
        for {
            fmt.Println("i is", i)
            time.Sleep(time.Second)
        }
    }()

    for {
        i += 1
    }
}
```

在我mac本地环境会不断的输出0。
全局变量i被两个goroutine同时读写，也就是我们所说的data race，导致了i的值是未定义的。
如果读写的是一块动态伸缩的内存，很有可能会导致panic。例如多goroutine读写map。
幸运的是，Golang针对data race有专门的内置工具，例如把上面的代码保存为main.go，执行 `go run -race main.go` 会把相关的data race输出：

```text
==================

WARNING: DATA RACE

Read at 0x00000121e848 by goroutine 6:

  main.main.func1()

      /Users/saas/src/awesomeProject/datarace/main.go:15 +0x3e

 

Previous write at 0x00000121e848 by main goroutine:

  main.main()

      /Users/saas/src/awesomeProject/datarace/main.go:21 +0x7b

 

Goroutine 6 (running) created at:

  main.main()

      /Users/saas/src/awesomeProject/datarace/main.go:13 +0x4f

==================
```

# 其他
## channel 和 全局变量
chan 类似管道，管道顾名思义一端进一端出，很形象表明了一个连接器。go 中的 chan 连接 goroutine，游离于众多 goroutine 之间，功用性与全局变量有得一拼。
但 chan 绝对不是全局变量，一个全局变量，可以在同一函数体内重复读写，但对无缓冲 chan 而言是不可以，原因在同一 goroutine 内对同一 chan 读写时，存在读或写阻塞面临切换上下文，另一个对应的永远没执行机会。

- 无缓冲通道
如下的程序会出现死锁

```go
 package main
 import "fmt"

 func main() {
     ch := make(chan int)
     ch <- 5
     fmt.Println(<-ch)
 }
```

- 有缓冲通道
如下的程序正常
```go
 package main

 import "fmt"

 func main() {
     ch := make(chan int, 1)
     ch <- 5
     fmt.Println(<-ch)
 }
```
> 注：有缓冲通道，意味着在未超过当前通道限制数之前，当前的 goroutine 是非阻塞，不会发生上下文切换，即当前 goroutine 的控制权不发生转移，runtime 也就不会去寻求其它相关 goroutine 执行。

# 参考
```c
# Go 程序员面试笔试宝典：优雅关闭channel
https://golang.design/go-questions/channel/graceful-close/


# Golang channel 源码深度剖析
https://www.cyhone.com/articles/analysis-of-golang-channel/

# 深入理解 Go Channel
http://legendtkl.com/2017/07/30/understanding-golang-channel/

# 实现无限缓存的channel
https://colobu.com/2021/05/11/unbounded-channel-in-go/


# Go并发编程实战课
https://politcloud.org/categories/go%E5%B9%B6%E5%8F%91%E7%BC%96%E7%A8%8B%E5%AE%9E%E6%88%98%E8%AF%BE/
```