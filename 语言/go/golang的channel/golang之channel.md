```table-of-contents
```
# channel简介
## 通道的作用
为什么要有通道呢？ 通道的作用是解决各个gorountine之间通信的问题;

在开始之前可以先记住一个原则，通道必须和gorountine一起使用，一个直观的体现就是必须要和`go`关键字搭配使用。 其实也非常好理解，因为如果只有一个gorountine，那么也用不着通信
# channel的特点
特点:
- 线程安全
- `chan` 是一种类型（引用类型）,需要make初始化
>`chan int`代表这是一个int型的通道；
>既然它是一种类型,那就涉及是引用类型还是值类型，它是引用类型，所以需要make初始化。


# channel 基础操作
## 创建 channel
前面说了**通道是引用类型**,因此channel 使用之前需要通过 make 创建。

```go
unBufferChan := make(chan int) // 1  
bufferChan := make(chan int, N) // 2
```
上面的方式 1 创建的是无缓冲 channel，方式 2 创建的是缓冲 channel。

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
- 向 channel 中添加数据（channel<-data）;
- 从 channel 中读取数据（data<-channel）;
    - data<-channel， 从 channel 中接收数据并赋值给 data
    - <-channel，从 channel 中接收数据并丢弃

## channel 种类
channel 分为无缓冲 channel 和有缓冲 channel。两者的区别如下：

- 无缓冲：
发送和接收动作是同时发生的。如果没有 goroutine 读取 channel （<- channel），则发送者 (channel <-) 会一直阻塞。
![](attachments/Pasted%20image%2020231215155120.png)

- 有缓冲：
缓冲 channel 类似一个有容量的队列。当队列满的时候发送者会阻塞；当队列空的时候接收者会阻塞。
![](attachments/Pasted%20image%2020231215155224.png)

## 关闭 channel

通道和文件不同，创建后通道可以不用写close，它可以被垃圾回收；
只有在必须告诉接收者不再有需要发送的值时才有必要关闭。
> 即：对于 channel 而言，唯一需要 close 的就是我们想通过 close 触发 channel 读事件。

发送者可通过 `close` 关闭一个信道来表示没有需要发送的值了。
接收者可以通过为接收表达式分配第二个参数来测试信道是否被关闭：
> 若**没有值可以接收且信道已被关闭**，那么在执行完之后 `ok` 会被设置为 `false`。
```go
v, ok := <-ch
```
**ok的结果和含义**：  
- `true`：读到通道数据，不确定是否关闭，可能channel还有保存的数据，但channel已关闭。  
- `false`：通道关闭，无数据读到。

**close channel 的注意事项**：
- 重复关闭 channel 会导致 panic。
- 向关闭的 channel 发送数据会 panic。
- close通道后，即使通道内还有数据，也不影响接收。它会将所有数据读取完毕，在读取完已发送的数据后会返回元素类型的零值(zero value)。
> 即：从关闭的 channel 读数据不会 panic，读出 channel 中已有的数据之后再读就是 channel 类似的默认值，比如 chan int 类型的 channel 关闭之后读取到的值为 0。
> 读取关闭后的无缓存通道，不管通道中是否有数据，返回值都为默认值 和 false。
> 读取关闭后的有缓存通道，将缓存数据读取完后，再读取返回值为默认值 和 false。

- 发送方来触发close channel。
>因为 channel 的关闭在接收端能感知到，但是发送端感知不到，因此一般只能在发送端主动关闭。而且大部分时候可以不执行 close，只需要读写即可。

### 区分默认值和channel关闭
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
### 优雅判断是否 close 的封装
```go
package main

import "fmt"

type T int

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

## channel的状态
channel存在`3种状态`：

1. nil，未初始化的状态，只进行了声明，或者手动赋值为`nil`
2. active，正常的channel，可读或者可写
3. closed，已关闭
>注：**千万不要误认为关闭channel后，channel的值是nil**

channel可进行`3种操作`：
1. 读
2. 写
3. 关闭

把这3种操作和3种channel状态可以组合出`9种情况`：

|操作|nil的channel|正常channel|已关闭channel|
|---|---|---|---|
|<- ch|阻塞|成功或阻塞|读到零值|
|ch <-|阻塞|成功或阻塞|panic|
|close(ch)|panic|成功|panic|

对于nil通道的情况，也并非完全遵循上表，**有1个特殊场景**：当`nil`的通道在`select`的某个`case`中时，这个case会阻塞，但不会造成死锁。

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

## channel 的构造
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


# 无缓冲channel
## 定义
如果缓冲大小设置为 0 或者不设置，channel 为无缓冲类型。
```go
    make(chan Type)   //等价于make(chan Type, 0)
```

## 操作
- 创建
```
var ch = make(chan int)  // 创建一个int类型的channel
cap(ch)                  // ch的容量是0
```

- 发送/存入 
```
ch <- 1  // 存入一个int类型的值
```

- 接收/取出
 ```
x := <-ch  // 取出ch中的值，并赋值给x
```

- 关闭
 ```
close(ch) // 关闭发送方channel，对接收发channel关闭操作会panic
val, ok := <-ch // ok 可用于判断通道是否关闭。
```

## 特征
**无缓冲 channel**：
发送者会阻塞直到接收者接收了发送的值。无缓冲的channel的读写者必须**同时完成发送和接收**。
即：无缓冲 channel 在消息发送时需要接收者就绪。也就是说receiver的goroutine 和sender的goroutine 必须成对出现, 不可以是一个goroutine中出现读和写。

```go
package main

import (
	"sync"
	"time"
)

func main() {
	c := make(chan string)

	var wg sync.WaitGroup
	wg.Add(2)

	go func() {
		defer wg.Done()
		c <- `foo`
	}()

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
- 分析：
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
有缓冲的通道并不强制channel的读写者必须**同时完成发送和接收**，读者只会在没有数据时阻塞，写者只会在没有可用容量时阻塞，这就有点像阻塞队列了。
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
- 向 nil 通道发送数据会被阻塞
- 向无缓冲 channel 写数据，如果读协程没有准备好，会阻塞
    - 无缓冲 channel ，必须要有读有写，写了数据之后，必须要读出来，否则导致 channel 阻塞，从而使得协程阻塞而使得协程泄漏
    - 一个无缓冲 channel，如果每次来一个请求就开一个 go 协程往里面写数据，但是一直没有被读取，那么就会导致这个 chan 一直阻塞，使得写这个 chan 的 go 协程一直无法释放从而协程泄漏。
- 向有缓冲 channel 写数据，如果缓冲已满，会阻塞
    - 有缓冲的 channel，在缓冲 buffer 之内，不读取也不会导致阻塞，当然也就不会使得协程泄漏，但是如果写数据超过了 buffer 还没有读取，那么继续写的时候就会阻塞了。如果往有缓冲的 channel 写了数据但是一直没有读取就直接退出协程的话，一样会导致 channel 阻塞，从而使得协程阻塞并泄漏。

## 读操作阻塞
读操作，什么时候会被阻塞？

- 从 nil 通道接收数据会被阻塞
- 从无缓冲 channel 读数据，如果写协程没有准备好，会阻塞
- 从有缓冲 channel 读数据，如果缓冲为空，会阻塞
- 
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
# Channel 的缺点
Channel 的缺点：
1. Channel 可能会导致循环阻塞或者协程泄漏，这个是最最要重点关注的。
2. Channel 中传递指针会导致数据竞态问题（data race/ race conditions）
3. Channel 中传递的都是数据的拷贝，可能会影响性能，但是就目前我们的机器性能来看，这点数据拷贝所带来的 CPU 消耗，大多数的情况下可以忽略。


# channel 的其他用法
## channel 实现并发同步
channel 作为 Go 并发模型的核心思想：不要通过共享内存来通信，而应该通过通信来共享内存。

那么在 Go 里面，当然也可以很方便通过 channel 来实现协程的并发和同步了，并且 channel 本身还可以支持有缓冲和无缓冲的，通过 channel + timeout 实现并发协程之间的同步也是常见的一种使用姿势。

## 通道取值
### for-range 读取channel数据
不管是有缓冲还是无缓冲，都可以使用 for-range 从 channel 中读取数据，并且这个是一直循环读取的。
for-range 中的 range 产生的迭代值为向 Channel 中发送的值，如果已经这个 channel 已经 close 了，那么首先还会继续执行，直到所有值被读取完，然后才会跳出 for 循环。
因此，通过 for-range 读取 chann 数据会比较方便，因为我们只需要读取数据就行了，不需管他的退出，在 close 之后如果数据读取完了会自动帮我们退出。
如果既没有 close 也没有数据可读，那么就会阻塞到 range 这里，除非有数据产生或者 chan 被关闭了。
如果 channel 是 nil，读取会被阻塞，也就是会一直阻塞在 range 位置。

```go
   ch := make(chan int)

   // 一直循环读取 range 中的迭代值
   for v := range ch {
        // 得到了 v 这个 chann 中的值
        fmt.Println("读取数据：",v)
   }

```

## 单向 channel
单向 channel，顾名思义只能写或读的 channel。但是仔细一想，只能写的 channel，如果不读其中的值有什么用呢？其实单向 channel 主要用在函数声明中。比如。
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

// ch只能存
func counter(ch chan<- int) {
	for i := 0; i < 6; i++ {
		ch <- i
	}

	close(ch) // 存完后关闭通道（放心不影响取）
}

// ch1只能存
// ch2只能取
func squarer(ch1 chan<- int, ch2 <-chan int) {
	for i := range ch2 {
		ch1 <- i * i
	}

	close(ch1) // ch1存完关闭
}

// ch 只能取
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
**场景**
使用channel传递信号，而不是传递数据时。

**原理**
没数据需要传递时，传递空struct。

## 使用channel传递结构体的指针而非结构体
**原理**
channel本质上传递的是数据的拷贝，拷贝的数据越小传输效率越高，传递结构体指针，比传递结构体更高效。

**用法**
```go
reqCh chan *Request  
  
// 好过  
reqCh chan Request
```

## 使用channel传递channel
**场景**
使用场景有点多，通常是用来获取结果。

**原理**
channel可以用来传递变量，channel自身也是变量，可以传递自己。

**用法**
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
# Golang channel 源码深度剖析
https://www.cyhone.com/articles/analysis-of-golang-channel/

# 深入理解 Go Channel
http://legendtkl.com/2017/07/30/understanding-golang-channel/

# 实现无限缓存的channel
https://colobu.com/2021/05/11/unbounded-channel-in-go/
```