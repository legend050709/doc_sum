```table-of-contents
```
# 背景
很多时候，我们会遇到这样的情况，上层与下层的goroutine需要同时取消，这样就涉及到了goroutine间的通信。在Go中，**推荐我们以通信的方式共享内存，而不是以共享内存的方式通信**。所以，就需要用到channl，但是，在上述场景中，如果需要自己去处理channl的业务逻辑，就会有很多费时费力的重复工作，因此，context出现了。


# context 介绍
Context通常被译作上下文，它是一个比较抽象的概念。
般理解为程序单元(Goroutine)的一个运行状态、现场、快照。context翻译为上下文，其中上下是指存在上下层的传递，上层会把内容传递给下层。

每个 Goroutine 在执行之前，都要先知道程序当前的执行状态，通常父Goroutine将这些执行状态封装在一个“上下文”变量中，传递给要执行的 Go 协程中。

context包不仅实现了在程序单元之间共享状态变量的方法，同时能通过简单的方法，使我们在被调用程序单元的外部，通过设置ctx变量值，将过期或撤销这些信号传递给被调用的程序单元。
> 比如：在网络编程中，若存在A调用B的API, B再调用C的API，若A调用B取消，那也要取消B调用C，通过在A,B,C的API调用之间传递Context，以及判断其状态，就能解决此问题，这是为什么gRPC的接口中带上ctx context.Context参数的原因之一。

# context作用

Context 只有两个简单的功能：
 1) 跨 API 或在goroutine 间携带键值对.
 2) 传递取消信号(主动取消、时限/超时自动取消) .

在 Goroutine 构成的树形结构中对信号进行同步以减少计算资源的浪费是 `context.Context` 的最大作用。
Go 服务的每一个请求都是通过单独的 Goroutine 处理的，HTTP/RPC 请求的处理器会启动新的 Goroutine 访问数据库和其他服务。
如下图所示，我们可能会创建多个 Goroutine 来处理一次请求，而 `context.Context`的作用是在不同 Goroutine 之间同步请求特定数据、取消信号以及处理请求的截止日期。
![](attachments/Pasted%20image%2020231122145406.png)


每一个 `context.Context`都会从最顶层的 Goroutine 一层一层传递到最下层。`context.Context`可以在上层 Goroutine 执行出现错误时，将信号及时同步给下层。
1》不使用 Context 同步信号时：
![](attachments/Pasted%20image%2020231122145647.png)
如上图所示，当最上层的 Goroutine 因为某些原因执行失败时，下层的 Goroutine 由于没有接收到这个信号所以会继续工作；

2》使用 Context 同步信号时：
![](attachments/Pasted%20image%2020231122145726.png)
当我们正确地使用`context.Context`可以在上层 Goroutine 执行出现错误时，就可以在下层及时停掉无用的工作以减少额外资源的消耗：

# context 包
context 包是专门用来简化处理针对单个请求的多个 Go 协程与请求截止时间、取消信号以及请求域数据等相关操作的。
 
官方文档对于 context 包的解释是：
> Package context defines the Context type, which carries deadlines, cancelation signals, and other request-scoped values across API boundaries and between processes.

一个常遇到的例子是在 Go 语言实现的服务器程序中，每个网络请求一般需要创建单独的Go 协程进行处理，这些 Go 协程有可能涉及到多个 API 的调用，进而可能会开启其他的 Go 协程；由于这些 Go 协程都是在处理同一个网络请求，所以它们往往需要访问一些共享的资源，比如用户认证令牌环、请求截止时间等；而且如果请求超时或者被取消后，所有的 Go 协程都应该马上退出并且释放相关的资源。使用上下文可以让 Go 语言开发者方便地实现这些 Go 协程之间的交互操作，跟踪并控制这些 Go 协程，并传递请求相关的数据、取消 Go 协程的信号或截止日期等。
![](attachments/Pasted%20image%2020231122120406.png)

#  context 原理
Goroutine 的创建和调用关系往往是有着层级关系的，顶部的 Goroutine 应有办法主动关闭其下属的 Goroutine 的执行。为了实现这种关系，Context 结构也应该像一棵树，叶子节点须总是由根节点衍生出来的。
第一个创建 Context 的 goroutine 被称为 root 节点：root 节点负责创建一个实现 Context 接口的具体对象，并将该对象作为参数传递至新拉起的 goroutine 作为其上下文。下游 goroutine 继续封装该对象并以此类推向下传递。
Context 可以安全的被多个 goroutine 使用。开发者可以把一个 Context 传递给任意多个 goroutine 然后 cancel 这个 context 的时候，就能够通知到所有的 goroutine。

# context 的底层设计
context是Go中用来进程通信的一种方式，其底层是借助`channl`与`snyc.Mutex`实现的。
`context`的底层设计，我们可以概括为1个接口，4种实现与6个方法。
- **1 个接口/数据结构**
    - Context 规定了`context`的四个基本方法。

- **4 种实现**
	- emptyCtx 实现了一个空的`context`，可以用作根节点
	- cancelCtx 实现一个带`cancel`功能的`context`，可以主动取消
	- timerCtx 实现一个通过定时器`timer`和截止时间`deadline`定时取消的`context`
	- valueCtx 实现一个可以通过 `key、val` 两个字段来存数据的`context`

- **6 种方法**
	- Background 返回一个`emptyCtx`作为根节点
	- TODO 返回一个`emptyCtx`作为未知节点
	- WithCancel 返回一个`cancelCtx`
	- WithDeadline 返回一个`timerCtx`
	- WithTimeout 返回一个`timerCtx`
	- WithValue 返回一个`valueCtx`

## context的接口
**Context 接口源码**, 如下所示：
```golang
// A Context carries a deadline, cancelation signal, and request-scoped values
// across API boundaries. Its methods are safe for simultaneous use by multiple
// goroutines.
type Context interface {
    // Done returns a channel that is closed when this Context is canceled
    // or times out.
    Done() <-chan struct{}

    // Err indicates why this context was canceled, after the Done channel
    // is closed.
    Err() error

    // Deadline returns the time when this Context will be canceled, if any.
    Deadline() (deadline time.Time, ok bool)

    // Value returns the value associated with key or nil if none.
    Value(key interface{}) interface{}
}

```
- **Done** 方法在 Context 被取消或超时时返回一个 close 的 只读的任何类型的管道channel。使用该方法开发者可以在接收到上下文的取消请求，或者截止时间到了之后做一些清理操作，然后退出 Go 协程，释放相关资源，也可以调用 `Err()` 方法得到上下文被取消的原因，或者调用 `Value()` 方法得到上下文中的相关值。
- **Err** 返回上下文被取消的原因，此方法一般是在 `Done()` 方法返回的管道有数据的时候（表明相应的上下文被取消）调用；
- **Deadline** 方法即设置上下文的截止时间，到了这个时间上下文会自动发起取消请求；当第二个返回值 `ok` 为 `false` 时表示没有设置截止时间，如果需要取消的话，需要调用取消函数进行取消；
- **Value** 方法通过“键”获取上下文上绑定的“值”，这个方法是线程安全的，与 `Err()` 方法一样，一般是在 `Done()` 方法返回的管道有数据的时候调用；

## context的实现
### emptyCtx
```go
type emptyCtx int

func (*emptyCtx) Deadline() (deadline time.Time, ok bool) {
	return
}

func (*emptyCtx) Done() <-chan struct{} {
	return nil
}

func (*emptyCtx) Err() error {
	return nil
}

func (*emptyCtx) Value(key interface{}) interface{} {
	return nil
}

```

`context.emptyCtx` 通过空方法实现了 `context.Context` 接口中的所有方法，它没有任何功能。因此 `emptyCtx` 是一个不能被取消，没有设置截止时间也没有没有携带任何值的 Context 接口的实现。
`emptyCtx`的 主要作用是：通过 context 包中的两个工厂方法来基于 `emptyCtx` 创建两种不同的 Context 对象，需要开始上下文的时候都是以这两个 Context 对象作为最顶层的“根上下文”，然后再衍生出其他的“子上下文”，最终这些上下文被组织成一棵树状结构；这样，当一个上下文被取消时，所有继承自它的上下文都会被自动取消。
```go
var (
    background = new(emptyCtx)
    todo       = new(emptyCtx) 
)

func Background() Context {
    return background
}
func TODO() Context {
	return todo
}
```
`Background`和`TODO`也只是互为别名，在实现上没有区别，只是在使用语义上有所差异：

- `Background`得到上下文的默认值，所有其他的上下文都应该从它衍生出来；主要用于 `main()` 函数、初始化以及测试代码中；是上下文的根节点，它不能被取消；
- `TODO`应该仅在不确定应该使用哪种上下文时使用；

在多数情况下，如果当前函数没有上下文作为入参，我们都会使用 `context.Background`作为起始的上下文向下传递。

### cancelCtx
`cancelCtx`实现了`canceler`接口与`Context`接口：
```go
type canceler interface {
	cancel(removeFromParent bool, err error)
	Done() <-chan struct{}
}

type cancelCtx struct {
    // 直接嵌入了一个 Context，那么可以把 cancelCtx 看做是一个 Context
	Context

	mu       sync.Mutex            // protects following fields
	done     atomic.Value          // of chan struct{}, created lazily, closed by first cancel call
	children map[canceler]struct{} // set to nil by the first cancel call
	err      error                 // set to non-nil by the first cancel call
}
```
我们可以使用`WithCancel`的方法来创建一个`cancelCtx`:
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
	if parent == nil {
		panic("cannot create context from nil parent")
	}
	c := newCancelCtx(parent)
	propagateCancel(parent, &c)
	return &c, func() { c.cancel(true, Canceled) }
}
func newCancelCtx(parent Context) cancelCtx {
	return cancelCtx{Context: parent}
}
```
##  context的方法
`Context` 接口中有四个核心方法：`Deadline()`、`Done()`、`Err()`、`Value()`。

### Deadline()
`Deadline() (deadline time.Time, ok bool)` 方法返回 `Context` 的截止时间，表示在这个时间点之后，`Context` 会被自动取消。如果 `Context` 没有设置截止时间，该方法返回一个零值 `time.Time` 和一个布尔值 `false`。
```go
deadline, ok := ctx.Deadline()
if ok {
    // Context 有截止时间
} else {
    // Context 没有截止时间
}

```

### Done()
`Done()` 方法返回一个只读通道，当 `Context` 被取消时，该通道会被关闭。你可以通过监听这个通道来检测 `Context` 是否被取消。如果 `Context` 永不取消，则返回 `nil`。
```go
select {
case <-ctx.Done():
    // Context 已取消
default:
    // Context 尚未取消
}

```
### Err()
`Err()` 方法返回一个 `error` 值，表示 `Context` 被取消时产生的错误。如果 `Context` 尚未取消，该方法返回 `nil`。
```go
if err := ctx.Err(); err != nil {
    // Context 已取消，处理错误
}
```
### Value()
`Value(key any) any` 方法返回与 `Context` 关联的键值对，一般用于在 `Goroutine` 之间传递请求范围内的信息。如果没有关联的值，则返回 `nil`。
```go
value := ctx.Value(key)
if value != nil {
    // 存在关联的值
}
```

# context 的创建
顶部的Goroutine应有办法主动关闭其下属的Goroutine的执行（不然程序可能就失控了）。为了实现这种关系，Context结构也应该像一棵树，叶子节点须总是由根节点衍生出来的。
![](attachments/Pasted%20image%2020231122154200.png)
## 根context 的创建
有两种方式创建根 Context：
1. `context.Background()`
2. `context.TODO()`
### `context.Background()` 
在顶层 goroutine 中通过调用 `context.Background()` 可以返回一个空 的Context，该Context一般由接收请求的第一个Goroutine创建。这个 Context 是 所有 Context 的 root，不能够被cancel，没有值、也没有过期时间。它常常作为处理Request的顶层context存在。


## 子context的派生
有四种方法派生 context ：
1. `func WithCancel(parent Context) (ctx Context, cancel CancelFunc)`
2. `func WithValue(parent Context, key, val interface{}) Context`
3. `func WithDeadline(parent Context, d time.Time) (Context, CancelFunc)`
4. `func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)`
函数都接收一个Context类型的参数parent，并返回一个Context类型的值，这样就层层创建出不同的节点。

```c
ctx, cancel := context.WithCancel(context.Background())
```

### `context.WithCancel()`
最常用的一种 context 派生方式，接收一个父 context（可以是 background context，或其它 Context），返回一个派生的 context 以及一个用于控制的 cancel 函数对象。如下所示：
```go
func WithCancel(parent Context) (ctx Context, cancel CancelFunc)
```
额外的CancelFunc函数类型变量，该函数类型的定义为：
```c
type CancelFunc func()
```

WithCancel 返回一个继承的 context，这个 **context 在父 context 的 Done 被关闭时关闭自己的 Done 通道，或者在自己被 Cancel 的时候关闭自己的 Done**。（**注意：读关闭的 channel 返回类型零值**）

调用CancelFunc对象将撤销对应的Context对象，这就是主动撤销Context的方法。在父节点的Context所对应的环境中，通过WithCancel函数不仅可创建子节点的Context，同时也获得了该节点Context的控制权，一旦执行CancelFunc函数，则该节点Context就结束了，则子节点需要类似如下代码来判断是否已结束，并退出该Goroutine：
```c
select {  
    case <-cxt.Done():  
        // do some clean...  
}
```

范例如下所示：
```go
func job() {
    ctx, cancel := context.WithCancel(context.Background())
    go doSomething(ctx)

    time.Sleep(5 * time.Second)
    cancel()
}

func doSomething(ctx context.Context) {
    for {
        time.Sleep(1 * time.Second)
        select {
        case <-ctx.Done():
            fmt.Println("done")
            return
        default:
            fmt.Println("working")
        }
    }
}

```

### `context.WithTimeout()`
派生一个带有超时机制的 context。达到 Timeout 时长后，该 context 以及该 context 的子 context 会收到 cancel 信号退出。当然，如果在 Timeout 时长内调用 cancel，则会提前发送 cancel 信号退出。
```go
func WithTimeout(parent Context, timeout time.Duration) (Context, CancelFunc)
```
比如：
```go
ctx, cancel := context.WithTimeout(parentCtx, 5*time.Second)
```
### `context.WithDeadline()`
派生一个带有绝对时限的 context，与 `WithTimeout()` 作用基本相同，仅仅是时间设定方式上不同。
达到 deadline 设定的时间后，该 context 以及该 context 的子 context 会收到 cancel 信号退出。当然，如果在 deadline 之前调用 cancel，则会提前发送 cancel 信号退出。

```go
ctx, cancel := context.WithDeadline(parentCtx, time.Now().Add(5*time.Second))
```


### `context.WithValue()`
派生一个携带信息的 context 用于传递。比如在 Request 中携带认证信息，携带用户数据等。
```go
WithValue(parent Context, key, val interface{})
```
参数包含三个部分：
- **parent**，用于派生子 context 的父 context；
- **key**，携带信息的 key，interface{} 类型；
- **value**，携带信息的 value，interface{} 类型，通常在接收到信息后通过断言（`.(T)`）将 value 转换成正确的类型使用；

接收 context 携带的信息可以使用 `ctx.Value(K)` 接收到 value（**interface{}类型**）。


# 层级 Context 间的传递与控制
- **生命周期**：Context 对象的生命周期一般仅为一个请求的处理周期。即针对一个请求创建一个 Context 变量（它为 Context 树的根）；在请求处理结束后，撤销此 ctx 变量，释放资源。
- **传递方式**：每次创建一个 Goroutine，要么将原有的 Context 传递给 Goroutine，要么创建一个子 Context 并传递给 Goroutine。
- **安全读写**：Context能灵活地存储不同类型、不同数目的值，并且使多个 Goroutine 安全地读写其中的值。
- **控制权**：当通过父 Context 对象创建子 Context 对象时，可同时获得子 Context 的一个 Cancel 函数对象，这样就获得了对子任务的控制权。

# context的使用实践
- “根上下文” 一般为 background 上下文，通过调用 `context.Background()` 得到；
- 传递 Context 时，不应把 Context 放入 struct，而是以参数的方式显示地在函数间传递；一般把上下文作为第一个参数传递给入口请求和出口请求链路上的每一个函数，变量名推荐使用 `ctx`；
- 不要传递值为 `nil` 的上下文给函数或者方法，否则在追踪的时候，就会断了上下文树的连接；
- 同一个 context 可以传递到不同的 goroutine 中，且在多个 goroutine 可以安全访问；对它执行取消操作时，所有 Go 协程都会接收到取消信号
- 使用上下文的 `Value()` 方法应该传递必须的数据，不要什么数据都使用 `Value()` 方法传递；上下文传递数据是线程安全的。
>即：不要把本应该作为函数参数的类型塞到 context 中，context 存储的应该是一些共同的数据。例如：登陆的 session、cookie 等。

# 小结
Go 语言中的“上下文”的主要作用还是在多个 Go 协程或者模块之间同步取消信号或者截止日期，用于减少对资源的消耗和长时间占用，避免资源浪费。
虽然传值也是它的功能之一，但是这个功能我们还是很少用到。在真正使用传值的功能时我们也应该非常谨慎，不能将请求的所有参数都使用“上下文”进行传递，这是一种非常差的设计，比较常见的使用场景是传递请求对应用户的认证令牌以及用于进行分布式追踪的请求 ID。

在 Go 语言编程中，通常不能直接杀死 Go 协程，协程的关闭一般会用“管道 + `select`”的方式来控制。但是在某些场景下，例如为了处理一个请求衍生了很多 Go 协程，这些协程之间需要共享一些全局变量、有共同的截止时间等，而且可以同时被关闭，这种情况下再使用“管道 + `select`”就比较麻烦，而可以通过“上下文”就可以轻松应对。

# 使用场景
## 主动传递取消信号，结束任务
启动一个工作协程，接收到取消信号就停止工作。
```go
package main

import (
   "context"
   "fmt"
   "time"
)

func main() {
   ctx, cancelFunc := context.WithCancel(context.Background())
   go Working(ctx)

   time.Sleep(3 * time.Second)
   cancelFunc()

   // 等待一段时间，以确保工作协程接收到取消信号并退出
   time.Sleep(1 * time.Second)
}

func Working(ctx context.Context) {
   for {
      select {
      case <-ctx.Done():
         fmt.Println("下班啦...")
         return
      default:
         fmt.Println("陈明勇正在工作中...")
      }
   }
}

```
执行结果:
```text
······
······
陈明勇正在工作中...
陈明勇正在工作中...
陈明勇正在工作中...
陈明勇正在工作中...
陈明勇正在工作中...
下班啦...

```

```go
func process(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()
	respC := make(chan int)
	// business logic
	go func() {
		time.Sleep(time.Second * 2)
		respC <- 10
	}()
	// wait for signal
	for {
		select {
		case <-ctx.Done():
			fmt.Println("cancel")
			return errors.New("cancel")
		case r := <-respC:
			fmt.Println(r)
		}
	}
}

func main() {
	wg := new(sync.WaitGroup)
	ctx, cancel := context.WithCancel(context.Background())
	wg.Add(1)
	go process(ctx, wg)
	time.Sleep(time.Second * 5)
	// trigger context cancel
	cancel()
	// wait for gorountine exit...
	wg.Wait()
}
```
运行程序的输出：
```fallback
10
cancel
```
## 超时控制
```go
package main

import (
   "context"
   "fmt"
   "time"
)

func main() {
   // 使用 WithTimeout 创建一个带有超时的上下文对象
   ctx, cancel := context.WithTimeout(context.Background(), 3*time.Second)
   defer cancel()

   // 在另一个 goroutine 中执行耗时操作
   go func() {
      // 模拟一个耗时的操作，例如数据库查询
      time.Sleep(5 * time.Second)
      cancel()
   }()

   select {
   case <-ctx.Done():
      fmt.Println("操作已超时")
   case <-time.After(10 * time.Second):
      fmt.Println("操作完成")
   }
}

```
执行结果
```text
操作已超时
```
在上面的例子中，首先使用 `context.WithTimeout()` 创建了一个带有 **3** 秒超时的上下文对象 `ctx`。接下来，在一个新的 `goroutine` 中执行一个模拟的耗时操作，例如等待 **5** 秒钟。当耗时操作完成后，调用 `cancel()` 方法来取消超时上下文。

最后，在主 `goroutine` 中使用 `select` 语句等待超时上下文的完成信号。如果在 **3** 秒内耗时操作完成，那么会输出 "操作完成"。如果超过了 **3** 秒仍未完成，超时上下文的 `Done()` 通道会被关闭，输出 "操作已超时"。

```golang
func process(ctx context.Context, wg *sync.WaitGroup) error {
	defer wg.Done()

	for i := 0; i < 1000; i++ {
		select {
		case <-time.After(2 * time.Second):
			fmt.Println("processing... ", i)

		// receive cancelation signal in this channel
		case <-ctx.Done():
			fmt.Println("Cancel the context ", i)
			return ctx.Err()
		}
	}
	return nil
}

func main() {
    wg := new(sync.WaitGroup)
	ctx, cancel := context.WithTimeout(context.Background(), 4*time.Second)
	defer cancel()

	wg.Add(1)
	go process(ctx, wg)
	wg.Wait()
}
```
运行程序的输出：

```fallback
processing...  0
processing...  1
Cancel the context  2
```

## 在 Go 协程之间传递数据
```golang
func main() {
	ctx, cancel := context.WithCancel(context.Background())
	valueCtx := context.WithValue(ctx, "mykey", "myvalue")

	go watch(valueCtx)
	time.Sleep(10 * time.Second)
	cancel()

	time.Sleep(5 * time.Second)
}

func watch(ctx context.Context) {
	for {
		select {
		case <-ctx.Done():
			fmt.Println(ctx.Value("mykey"), "is cancel")
			return
		default:
			fmt.Println(ctx.Value("mykey"), "int goroutine")
			time.Sleep(2 * time.Second)
		}
	}
}
```
运行程序的输出：

```fallback
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue int goroutine
myvalue is cancel
```

## 打印上下文的所有 key/val
通过反射的方式，可以打印出 context 的所有 key-value，但是因为 key 和 value 可以是任何类型，不一定有 `String()`，打印的可能不容易理解。
```go
func DumpContextKV(ctx interface{}) map[any]any {
    ret := make(map[any]any)

    contextKeys := reflect.TypeOf(ctx).Elem()
    contextValues := reflect.ValueOf(ctx).Elem()
    if contextKeys.Kind() != reflect.Struct {
        return ret
    }

    var key, val any
    found := false
    for i := 0; i < contextValues.NumField(); i++ {
        reflectValue := contextValues.Field(i)
        reflectValue = reflect.NewAt(reflectValue.Type(), unsafe.Pointer(reflectValue.UnsafeAddr())).Elem()

        reflectField := contextKeys.Field(i)

        if reflectField.Name == "Context" {
            tmpMap := dumpContextInternals(reflectValue.Interface())
            for k, v := range tmpMap {
                ret[k] = v
            }
        } else if reflectField.Name == "cancelCtx" {
            tmpMap := dumpContextInternals(reflectValue.FieldByName("Context").Interface())
            for k, v := range tmpMap {
                ret[k] = v
            }
        } else if reflectField.Name == "key" {
            found = true
            key = reflectValue.Interface()
        } else if reflectField.Name == "val" {
            val = reflectValue.Interface()
        }
    }
    if found {
        ret[key] = val
    }
    return ret
}
```
# 参考
```c
# Go 上下文
https://morven.life/posts/golang-context/

# Go上下文context底层原理
https://juejin.cn/post/7106157600399425543
```