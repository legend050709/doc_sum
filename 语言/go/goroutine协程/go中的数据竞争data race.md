```table-of-contents
```

# 什么是data race
在并发编程中，当多个协程、线程或者进程同时对某一块内存区域进行读写操作时，可能会出现数据竞争（Data Race）的问题。数据竞争是指两个或多个线程在没有合理同步的情况下同时访问同一个共享变量，并且至少有一个线程在写入变量的值。

**总结**
即 多个线程/协程，对于共享内存的访问，且至少存在一个写线程/协程。

# 为什么发生data race

数据竞争通常发生在并发执行的 goroutine 之间，当它们尝试同时读取和修改共享变量时。没有使用同步机制（如锁、channel 等）来控制这些访问顺序，便会产生数据竞争。

# 检测data race
## 介绍
 Go 版本 1.1 引入了一个竞态检测工具 (race detector), 这个竞态检测工具是**在编译流程中内置到你程序的代码**, 一旦你的程序开始运行, 它能够发现和报告任何他所检测到的竞态情况。
 
## 检测方法

Go 的自带工具链条中提供了若干个工具来检测数据竞争，分别是：
**（1）编译时加race，然后运行时检测**
```go
go run -race mysrc.go

等价于

go build -race -o mysrc mysrc.go
./mysrc
```

**(2) 测试代码时检测**
```go
go test -race ./...

它会检测整个测试套件中的数据竞争。
```

### 存在data race的检测
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var Wait sync.WaitGroup
var Counter int = 0

func main() {
    for routine := 1; routine <= 2; routine++ {
        Wait.Add(1)
        go Work(routine)
    }

    Wait.Wait()
    fmt.Printf("Final Counter: %d\n", Counter)
}

func Work(id int) {
    for i := 0; i < 10; i++ {
        Counter++
        time.Sleep(1 * time.Nanosecond)
    }

    Wait.Done()
}

```

当他检测到有 Data Race 时，就会以这样的格式报错：
```go
[root@admin go-race]# go build -race -o main.out main.go
[root@admin go-race]# ls
[root@admin go-race]# ./main.out 
==================
WARNING: DATA RACE
Read at 0x000000609908 by goroutine 8:
  main.Work()
      /root/code/go_work/project/gotour/go-race/main.go:24 +0x47

Previous write at 0x000000609908 by goroutine 7:
  main.Work()
      /root/code/go_work/project/gotour/go-race/main.go:24 +0x64

Goroutine 8 (running) created at:
  main.main()
      /root/code/go_work/project/gotour/go-race/main.go:15 +0x75

Goroutine 7 (running) created at:
  main.main()
      /root/code/go_work/project/gotour/go-race/main.go:15 +0x75
==================
Final Counter: 20
Found 1 data race(s)
```


虽然内容很多，但是，通常你只需要看两个关键词即可：
- **Read by goroutine**：这表示当出现 Data Race 时，读同一块内存的代码是谁；
- **Previous write by goroutine**：这表示当出现 Data Race 时，写同一块内存的代码是谁；

### 不存在data race的检测
加锁, 修改之:
```go
package main

import (
    "fmt"
    "sync"
    "time"
)

var Wait sync.WaitGroup
var Counter int = 0
var CounterLock sync.Mutex

func main() {
    for routine := 1; routine <= 2; routine++ {
        Wait.Add(1)
        go Work(routine)
    }

    Wait.Wait()
    fmt.Printf("Final Counter: %d\n", Counter)
}

func Work(id int) {
    for i := 0; i < 10; i++ {
        CounterLock.Lock()
        Counter++
        CounterLock.Unlock()
        time.Sleep(1 * time.Nanosecond)
    }

    Wait.Done()
}

```

再次运行, 检测通过，不存在data race。

## 检测原理

根据 Data Race 的定义，我们知道，要出现 Data Race，那么一定是有两个人对同一个内存进行同时的读写，这期间出现了交叉的过程，那么 Go 其实就是通过这个原理出发进行实现的。

**==Go 检测 Data Race 和我们加锁有点类似，就是在内存访问之前和访问之后都进行打点==**。

这么一段原始代码：
```go
# cat raw.go
func main() {
    go func() {
        x = 1
    }()
    fmt.Println(x)
}
```

当开启了 Data Race（`-race`）之后，Go 生成的代码可能就变成了：
```go
# cat compile.go
func main() {
    go func() {
        // 通知 Race Detector 写操作即将发生
        race.WriteAcquire(&x)
        x = 1
        // 通知 Race Detector 写操作已完成
        race.WriteRelease(&x)
    }()
    // 通知 Race Detector 读操作即将发生
    race.ReadAcquire(&x)
    value := x
    // 通知 Race Detector 读操作已完成
    race.ReadRelease(&x)
    fmt.Println(value)
}
```

相当于在内存操作前后进行了打点，然后有一个专门用来检测 Data Race 的组件：Data Race Detector，它可以用来检测对于同一块内存的访问是否有冲突的地方，整体的流程为：

 **(1) Race Detector 使用一个名为 Shadow Memory 的数据结构来存储内存访问的元数据**:
对于每个内存地址，Shadow Memory 会记录最近的两个访问操作，包括操作类型（读或写）、操作的 Goroutine 以及操作发生的时刻。

**(2) 检查并发访问**：
当 Race Detector 检测到一个内存访问操作时，它会检查与该操作相关的 Shadow Memory 记录。如果发现以下条件之一，那么就认为存在数据竞争：
(2.1)  当前操作是写操作，且与最近的两个访问操作之一（无论是读还是写）并发发生（即没有 happens-before 关系），且这两个访问操作来自不同的 Goroutine。

(2.2) 当前操作是读操作，且与最近的一个写操作并发发生（即没有 happens-before 关系），且这两个操作来自不同的 Goroutine。

**(3) 报告数据竞争**：
当检测到数据竞争时，Race Detector 会生成详细的报告，包括数据竞争发生的位置、涉及的 Goroutine 以及栈跟踪等信息。

## Data Race 的 各个组件

在运行时，Race Detector 的各个功能分布在多个 Goroutine 和线程中。
下面我总结了一些比较常用到的组件：
### Stub Code
Go 编译器会在内存访问和同步操作处插入额外的代码，以便与 Race Detector 通信。这些插入的代码在程序运行期间执行，直接在发生内存访问的 Goroutine 中运行。

### Shadow Memory
Race Detector 使用 Shadow Memory 来存储内存访问的元数据。Shadow Memory 的管理和更新在发生内存访问操作的 Goroutine 中进行，以保证元数据的实时性。

### Race Detector 的数据竞争检测

Race Detector 的数据竞争检测逻辑通常在发生内存访问操作的 Goroutine 中执行。这意味着，当一个 Goroutine 执行一个内存访问操作时，Race Detector 会在同一个 Goroutine 中检查是否存在数据竞争。

### 报告和诊断
当 Race Detector 检测到数据竞争时，它会生成详细的报告，包括数据竞争发生的位置、涉及的 Goroutine 以及栈跟踪等信息。报告生成可能会涉及多个 Goroutine 和线程，因为它需要收集和整理各种上下文信息。

### 小结
（1）当一个 Goroutine 发生内存访问操作时，Race Detector 会检查与之竞争的其他 Goroutine 是否存在。这是通过分析 Shadow Memory 中的元数据来完成的，元数据包含了每个内存地址的访问历史和同步关系。

（2） 如果检测到数据竞争，Race Detector 会收集有关竞争 Goroutine 的信息，包括其 ID、栈跟踪、发生竞争的内存地址以及相关的源代码位置。

（3）Race Detector 可以同时检测到多个数据竞争事件。对于每个事件，它都会生成一个独立的报告。报告中会包含详细的竞争 Goroutine 信息，以帮助开发者理解和解决问题。

（4）当程序执行结束或者在检测到数据竞争时，Race Detector 会将收集到的所有报告打印到标准错误输出（stderr）。每个报告都会单独显示，并按照发生的顺序排列。这样，开发者可以逐个查看和分析数据竞争事件。

## Data Race 检测的性能影响

虽然 Go 的 Data Race 启用很方便，并且 Go 编译器和 Race Detector 会尽可能地检测所有内存访问的数据竞争，但它们不能保证 100% 的准确性和完整性。

增加了 Data Race 检测之后，内存占用肯定会增加，执行速度也会降低，根据官方的文档说明：内存占用会有 5-10 倍的增加，执行时间会有 2-20 倍的降低。

# data race的常见范例

## for 中多个协程共享for的迭代变量

预期：多个协程合作，打印01234.

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    for i := 0; i < 5; i++ {
        go func() {
            fmt.Print(i)
        }()
    }
    // 让我们在退出之前等待goroutines执行完成
    time.Sleep(100 * time.Millisecond)
}
```

上述代码中 循环内的i被多个goroutine同时读取，代码执行结果有可能是44455或55555而不是 01234.

`i` 变量实际上是一个单独的变量，它接受每个数字的值。每个迭代执行的闭包函数都绑定同一个值，在运行这段代码时，因为 goroutine 很有可能是在循环结束后才开始执行，循环结束时 `i` 的值是 `5`，所以看到每次打印最后一个数字，而不是按顺序打印每个值。

### 分析

从本质上讲，这种数据竞争 是由并发 goroutine 之间不安全地**共享可变变量**（在本例中为 `i`）引起的。

### 解决
使用 goroutines 实现在 for 循环中异步处理一系列值的用例的正确方法是什么？答案就是**停止共享可变状态**，即在**每次for迭代中复制状态变量**（在本例中为 `i`）并将其传递给每个 goroutine。

```go
...
// no data race
for i := 0; i<5; i++ {
    go func(n int) {
        fmt.Print(n)
    }(i)
}
...

```

在此实现中，将 `i` 作为参数 (`n`) 按值传递给 goroutine； `n` 现在是 `i` 的 goroutine-local 副本，对 `i` 的后续更改不会反映在 `n` 上。


除此之外，在循环体内声明的变量不会在迭代之间共享，因此可以在闭包中单独使用。以下代码使用公共索引变量 `i` 来创建单独的 `n`，从而产生预期的行为：
```go
...
// no data race
for i := 0; i<5; i++ {
    n := i
    go func() {
        fmt.Print(n)
    }()
}
...
```

## 多个协程共享外部作用域的变量

```go
// ParallelWrite writes data to file1 and file2, returns the errors.
func ParallelWrite(data []byte) chan error {
	res := make(chan error, 2)
	f1, err := os.Create("file1")
	if err != nil {
		res <- err
	} else {
		go func() {
			// This err is shared with the main goroutine,
			// so the write races with the write below.
			_, err = f1.Write(data)
			res <- err
			f1.Close()
		}()
	}
	f2, err := os.Create("file2") // The second conflicting write to err.
	if err != nil {
		res <- err
	} else {
		go func() {
			_, err = f2.Write(data)
			res <- err
			f2.Close()
		}()
	}
	return res
}
```

上述代码：goroutine内共享了外部作用域的变量err ，导致数据读取出错.

### 解决
```go
...
_, err := f1.Write(data)
...
_, err := f2.Write(data)
...
```

## 多个协程共享命名返回值

如下所示，对于函数的返回值进行了命名，在函数内就可能存在多个协程对于该命名返回值进行读写。


```go
func NamedReturnCallee () ( result int) {
    result = 10
    if ... {
        return // this has the effect of " return 10"
    }

    go func () {
        ... = result // read result
    }()

    return 20 // this is equivalent to result =20
}

func Caller () {
    retVal := NamedReturnCallee ()
}

```


# golang 协程间 避免data race的方法
## 加锁
Go提供了多种同步原语如`sync.Mutex`、`sync.RWMutex`等，可以通过加锁的方式保证同一时间只有一个`goroutine`可以访问共享数据。

```go
import "sync"

var mutex sync.Mutex // 创建一个互斥锁

func modifySharedData(data *int) {
    mutex.Lock()      // 加锁
    // 修改共享数据的代码
    mutex.Unlock()   // 解锁
}
```

## 原子操作
`sync/atomic`包提供了低级的原子内存操作支持。原子操作可以保证任何时刻只有一个`goroutine`可以操作数据。

## 使用channel

Channel是Go语言中的一种通信机制，可以用于在goroutine之间传递消息和数据。
通过使用Channel，可以将共享数据的访问限制在发送和接收操作的边界上，从而避免数据竞争。

## 避免共享变量

尽量减少共享状态的必要性。
（1）通过将状态==局部化到每个goroutine中==，可以减少数据竞争的可能性。

## 使用并发安全的数据结构
一些数据结构，例如线程安全的队列、哈希表等，可以避免 Data Race 问题。
比如：`sync.map`

## 使用等待组（sync.WaitGroup）控制先后关系

只有一读一写时，在写操作完成之前阻止读访问。
```go
package main

import (
    "fmt"
    "sync"
)

func main() {
    fmt.Println(getNumber())
}

func getNumber() int {
    var i int
    // 初始化一个等待组变量
    var wg sync.WaitGroup
    // `Add(1) 表示我们需要等待 1 个任务
    wg.Add(1)
    go func() {
        i = 5
        // 调用 `wg.Done` 表示我们已经完成了等待的任务
        wg.Done()
    }()
    // `wg.Wait` 将会阻塞，直到 `wg.Done` 被调用的次数与我们的任务数相同
    wg.Wait()
    // 写完成之后，才读取返回。
    return i
}
```

# 如何在微服务中(多个进程中)避免数据竞争

在微服务架构中避免数据竞争的建议：
- **隔离状态**：将各微服务状态隔离，避免不同服务访问相同的数据。
- **使用一致性存储**：使用分布式锁或事务来确保微服务之间的一致性。
- **借助消息队列**：微服务之间可以通过消息队列异步通信，避免直接数据共享。


# 参考
```bash
# Golang 数据竞争详解：Data Race 原因、检测方法与实用解决方案
https://blog.axiaoxin.com/post/golang-data-race/

# go 的竞态检测机制 (race)
https://blog.csdn.net/wan212000/article/details/128816444
```