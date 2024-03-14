```table-of-contents
```
# 概述
`Mutex`是一个互斥锁，零值`Mutex`为未上锁状态。当一个 goroutine 获得了这个锁的拥有权后， 其它请求锁的 goroutine 就会阻塞在 Lock 方法的调用上，直到锁被释放。
使用起来也比较简单：
```go
package main  
  
import "sync"  
  
func main() {  
 m := sync.Mutex{}  
 m.Lock()  
 defer m.Unlock()  
  // do something  
}
```


# 锁结构
```go
type Mutex struct {
 state int32 // 锁竞争的状态值
 sema  uint32 // 信号量
}
```
上述两个加起来只占 8 字节空间的结构体表示了 Go语言中的互斥锁。

![](attachments/Pasted%20image%2020240206201339.png)

## state(状态)
```go
const (
    mutexLocked = 1 << iota // mutex is locked
    mutexWoken  //2
    mutexStarving //4
    mutexWaiterShift = iota //3
)
```
![](attachments/Pasted%20image%2020240206195558.png)

最低三位分别表示 mutexLocked、mutexWoken 和 mutexStarving，剩下的位置用来表示当前有多少个 Goroutine 等待互斥锁的释放.

`state`是int32类型，即有32位。

- 第1位比特用于Locked，1表示Mutex被锁上了，0表示没被锁上。
- 第2位比特用于Woken，1表示有协程从等待队列中被唤醒了，0表示没有协程被唤醒。
- 第3位比特用于Starving，1表示饥饿模式，0表示正常模式，后面讲两者不同处。
- 剩下的29位都用于WaiterShift计数，表示等待协程的数量有多少。


## sema(信号量)
`sema`是一个信号量，将没拿到锁的协程放入等待队列。

具体讲解：忽略，有些复杂。

# 运行模式
Golang 实现了一套「效率优先，兼顾公平」的Mutex锁。
![](attachments/Pasted%20image%2020240207160022.png)


整体的流程如下：
大致分为三个步骤：原子操作直接获取锁，如果失败进入自旋，如果自旋获取锁失败四次则进入等待队列
![](attachments/Pasted%20image%2020240207160109.png)

## 正常模式（非公平模式）
所有等待锁的goroutine按照FIFO顺序等待。唤醒的goroutine不会直接拥有锁，而是会和新请求锁的goroutine竞争锁的拥有。

新请求锁的goroutine具有优势：因为新来的`goroutine`很多已经占有了`CPU`，所以刚刚唤醒的goroutine有很大可能在锁竞争中失败。在这种情况下，这个被唤醒的goroutine会加入到等待队列的前面。

**正常模式切换为饥饿模式**：
如果一个等待的goroutine超过1ms没有获取锁，那么会将`Mutex`切换为饥饿模式。。


### Mutex.Lock 加锁
- 协程间互相竞争锁
- 某协程得不到锁后，可能会自旋几次，类似加载中转圈圈。目的是小等一会，期间其它协程释放锁了就趁机获得锁
- 多次尝试失败，就进入sema队列休眠


#### 分析
首先有多个协程竞争锁
![](attachments/Pasted%20image%2020240206202741.png)

某个协程得到锁了，其余协程自旋
![](attachments/Pasted%20image%2020240206202759.png)

自旋期间，有可能第一个协程释放锁，随之第二个协程竞争得到锁，而第三个协程自旋几次得不到锁，就放入sema等待队列休眠
![](attachments/Pasted%20image%2020240206202822.png)


### Mutex.Unlock 解锁
- 先解锁，将state的`Locked位`赋值为0
- 从等待队列队头中唤醒一个协程，唤醒的协程还得竞争锁，如果竞争不到继续放入等待队列，该协程很可能会饥饿。

## 饥饿模式（公平模式）
有协程饥饿了，为了兼顾公平，优先处理等待队列的协程。

锁的所有权直接交给交给等待队列中的第一个。新来的goroutine将不会尝试去获得锁，即使锁看起来是unlock状态, 也不会去尝试自旋操作，而是放在等待队列的尾部。

**饥饿模式切换为正常模式**：
如果一个等待的goroutine获取了锁，并且满足一以下其中的任何一个条件：
(1)它是队列中的最后一个；
(2)它等待的时候小于1ms。
它会将锁的状态转换为正常状态。
![](attachments/Pasted%20image%2020240206200248.png)


### Mutex.Lock 加锁
- 当有协程等待时间超过1ms，等太久不公平了，进入饥饿模式
- 饥饿模式下，新来的协程不需要自旋了，直接进入sema队列队尾
- `Mutex.Unlock`唤醒等待队列中的协程后，该协程将直接获得锁
- 直到等待队列中没有协程了，才切换回正常模式
#### 分析
首先等待队列中有协程，同时有个新协程打算竞争锁。
![](attachments/Pasted%20image%2020240207101453.png)

将等待队列出队，把队头的`sudog`封装的协程唤醒，该协程直接获得锁。新进入的协程直接加入等待队列队尾。
![](attachments/Pasted%20image%2020240207101501.png)

![](attachments/Pasted%20image%2020240207101531.png)
直到sema等待队列出队完了，Mutex才切换回正常模式。

### Mutex.Unlock 解锁
- 先解锁，将state的Locked置为0
- 从等待队列队头中唤醒一个协程，该协程在唤醒后会直接加锁

## 对比
正常模式有更好的性能，新来的`goroutine`通过几次竞争可以直接获取到锁，尽管当前仍有等待的`goroutine`。
饥饿模式也是非常重要的，防止等待队列中的`goroutine`永远没有机会获取锁；它能阻止尾部延迟的现象。

## 小结
互斥锁有两种操作模式：正常模式和饥饿模式。

**正常模式**
等待者按照先进先出的顺序排队，但是被唤醒的等待者不直接拥有互斥锁，而是与新到达的goroutine竞争所有权。新到达的goroutine有优势——它们已经在CPU上运行，并且可能有很多，因此被唤醒的等待者很有可能失败。
如果等待者未能在1毫秒内获取互斥锁，则它将切换到饥饿模式。

**饥饿模式**
互斥锁的所有权直接从解锁的goroutine移交给队列前面的等待者。即使互斥锁看起来已经解锁，新到达的goroutine也不会尝试获取互斥锁，也不会尝试自旋。相反，它们会将自己排队在等待队列的尾部。

**状态的切换**
在正常模式下，一旦Goroutine超过1ms没有获取到锁，它就会将当前互斥锁切换饥饿模式。
如果一个goroutine 获得了互斥锁并且它在队列的末尾或者它等待的时间少于 1ms，那么当前的互斥锁就会切换回正常模式。

# 函数实现
## Lock
上锁大致分为`fast-path`和`slow-path`
### Fast-path
lock通过调用`atomic.CompareAndSwapInt32`来竞争更新`m.state`，成功则获得锁；失败，则进入`slow-path`
```go
func (m *Mutex) Lock() {
 // Fast path: grab unlocked mutex.
 if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
  if race.Enabled {
   race.Acquire(unsafe.Pointer(m))
  }
  return
 }
 // Slow path (outlined so that the fast path can be inlined)
 m.lockSlow()
}
```
可以看到，当前互斥锁的状态为0时，尝试将当前锁状态设置为更新锁定状态，且这些操作是原子的。
若当前状态不为0，则进入`lockSlow`方法.

### slow-path
如果`goroutine`fast-path失败，则调用`m.lockSlow()`进入`slow-path`，函数内部主要是一个`for{}`死循环，进入循环的`goroutine`大致分为两类：

- 新来的`gorountine`
- 被唤醒的`goroutine`

`Mutex`默认为正常模式，若新来的`goroutine`抢占成功，则另一个就需要阻塞等待；阻塞等待一旦超过阈值1ms则会将`Mutex`切换到饥饿模式，这个模式下新来的`goroutine`只能阻塞等待在队列尾部，没有抢占的资格。当然等待阻塞->唤醒->参与抢占锁，这个过程显示不是很高效，所以这里有一个`自旋`的优化。
```go
func (m *Mutex) lockSlow() {
 var waitStartTime int64
 starving := false
 awoke := false
 iter := 0
 old := m.state
 for {
    //饥饿模式下不能自旋,也没有资格抢占，锁是手递手给到等待的goroutine
  if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {//当Mutex处于正常模式且能够自旋
      //设置mutexWoken为1 告诉unlock操作，存在自旋gorountine unlock后不需要唤醒其他goroutine
   if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
    atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
    awoke = true
   }
   runtime_doSpin()
   iter++
   old = m.state
   continue
  }
  //  自旋完了 还是没拿到锁
  new := old
    //当mutex处于正常模式，将new的mutexLocked设置为1 即准备抢占锁
  if old&mutexStarving == 0 {
   new |= mutexLocked
  }
    //加锁状态或饥饿模式下 新来的goroutine进入等待队列
  if old&(mutexLocked|mutexStarving) != 0 {
   new += 1 << mutexWaiterShift
  }

    //将Mutex切换为饥饿模式，若未加锁则不必切换
    //Unlock操作希望饥饿模式存在等待者
  if starving && old&mutexLocked != 0 {
   new |= mutexStarving
  }
  if awoke {
      // 当前goroutine自旋过 已被被唤醒，则需要将mutexWoken重置
   if new&mutexWoken == 0 {
    throw("sync: inconsistent mutex state")
   }
   new &^= mutexWoken //重置mutexWoken
  }
  if atomic.CompareAndSwapInt32(&m.state, old, new) {
      // 当前goroutine获取锁前mutex处于未加锁 正常模式下
   if old&(mutexLocked|mutexStarving) == 0 {
    break // 使用CAS成功抢占到锁
   }
   // waitStartTime!=0表示当前goroutine是等待状态唤醒的 
      // 为了与第一次调用Lock的goroutine划分不同的优先级
   queueLifo := waitStartTime != 0
   if waitStartTime == 0 {
        //开始记录等待时间
    waitStartTime = runtime_nanotime()
   }
      // 将被唤醒但是没有获得锁的goroutine插入到当前等待队列队首
      // 使用信号量阻塞当前goroutine
   runtime_SemacquireMutex(&m.sema, queueLifo, 1)
      // 当goroutine等待时间超过starvationThresholdNs，mutex进入饥饿模式
   starving = starving || runtime_nanotime()-waitStartTime > starvationThresholdNs
   old = m.state
   if old&mutexStarving != 0 {
        //如果当前goroutine被唤醒且mutex处于饥饿模式 则将锁手递手交给当前goroutine
    if old&(mutexLocked|mutexWoken) != 0 || old>>mutexWaiterShift == 0 {
     throw("sync: inconsistent mutex state")
    }
        //等待状态的goroutine - 1
    delta := int32(mutexLocked - 1<<mutexWaiterShift)
        //如果等待时间小于1ms 或 当前goroutine是队列中最后一个
    if !starving || old>>mutexWaiterShift == 1 {
      // 退出饥饿模式
     delta -= mutexStarving
    }
    atomic.AddInt32(&m.state, delta)
    break
   }
   awoke = true
   iter = 0
  } else {
   old = m.state
  }
 }
}
```
## Unlock
![](attachments/Pasted%20image%2020240206203901.png)

解锁分两种情况

1. 当前只有一个goroutine占有锁 unlock完 直接结束
```go
func (m *Mutex) Unlock() {

 // 去除加锁状态
 new := atomic.AddInt32(&m.state, -mutexLocked)
 if new != 0 {//存在等待的goroutine
  m.unlockSlow(new)
 }
}
```

# 潜在问题
## 锁拷贝
### 范例
```go
mu1 := &sync.Mutex{}
mu1.Lock()
mu2 := mu1
mu2.Unlock()
```
此时`mu2`能够正常解锁，那么我们再试试解锁`mu1`呢
```go
mu1 := &sync.Mutex{}
mu1.Lock()
mu2 := mu1
mu2.Unlock()
mu1.Unlock()
```
![](attachments/Pasted%20image%2020240206204046.png)
可以看到发生了error

## panic导致没有unlock
当lock()之后，可能由于代码问题导致程序发生了panic，那么mutex无法被及时unlock()，由于其他协程还在等待锁，此时可能触发死锁。

### 范例
```go
func TestWithLock() {
   nums := 100
   wg := &sync.WaitGroup{}
   safeSlice := SafeSlice{
      s:    []int{},
      lock: new(sync.RWMutex),
   }
   i := 0
   for idx := 0; idx < nums; idx++ { // 并行nums个协程做append
      wg.Add(1)
      go func() {
         defer func() {
            if r := recover(); r != nil {
               log.Println("recover")
            }

            wg.Done()
         }()

         safeSlice.lock.Lock()
         safeSlice.s = append(safeSlice.s, i)
         if i == 98{
            panic("123")
         }
         i++
         safeSlice.lock.Unlock()

      }()
   }
   wg.Wait()
   log.Println(len(safeSlice.s))
}
```
![](attachments/Pasted%20image%2020240206204438.png)

修改：
```go
func TestWithLock() {
   nums := 100
   wg := &sync.WaitGroup{}
   safeSlice := SafeSlice{
      s:    []int{},
      lock: new(sync.RWMutex),
   }
   i := 0
   for idx := 0; idx < nums; idx++ { // 并行nums个协程做append
      wg.Add(1)
      go func() {
         defer func() {
            if r := recover(); r != nil {
               
            }
            safeSlice.lock.Unlock()
            wg.Done()
         }()

         safeSlice.lock.Lock()
         safeSlice.s = append(safeSlice.s, i)
         if i == 98{
            panic("123")
         }
         i++
      }()
   }
   wg.Wait()
   log.Println(len(safeSlice.s))
}
```

# struct 结构体遇上 Mutex
如果你看过有关 sync 相关类型的介绍或者相关源码时, 在 `sync` 包里面的所有类型都有句这样的注释: `must not be copied after first use`。
 可能很多人却并不知道这句话有什么作用, 顶多看到相关介绍时还记得 `sync` 相关类型的变量不能复制, 可能真正使用 Mutex, WaitGroup, Cond时, 早把这个注释忘的一干二净.

究其原因, 我觉得有下面两点原因:
1. 不明白什么叫 sync 类型变量复制
2. sync 类型的变量复制了会出现怎样的结果


## 拷贝的场景
**（1）嵌套在 struct 里面, struct 变量间的互相赋值**
```go
type URL struct {  
  Ip       string  
  Port     string  
  mux     sync.RWMutex  
  params    url.Values  
}  
  
func main() {  
  var url1 URL  
  url2 := url1  
}
```
当 struct 嵌套 **「不可复制」** 类型时, 就需要开始小心了

**（2）struct 类型变量的值传递作为返回值**
```go
type URL struct {  
  Ip       string  
  mux     sync.RWMutex  
}  
  
func (c *URL) Clone() URL {  
  newUrl := URL{}  
  newUrl.Ip = c.Ip  
  return newUrl  
}
```

**（3）struct 类型变量的值传递作为 receiver**
```go
type URL struct {  
  Ip       string  
  mux     sync.RWMutex  
}  
  
func (c URL) String() string {  
  c.mux.Lock()  
  defer c.mux.Unlock()  
  // dosomething
}
```

## 拷贝的结果
![](attachments/Pasted%20image%2020240207104910.png)
如上所示，结果并不是预期的100.

![](attachments/Pasted%20image%2020240207105216.png)
如上所示，Reduce 协程发生了死锁。

看到这里我们就能发现, 当 struct 嵌套了 Mutex, 如果以值传递的方式使用时, 有可能造成程序死锁, 有可能需要互斥的变量并不能达到互斥.

所以不管是单独使用 **「不能复制」** 类型的变量, 还是嵌套在 struct 里面都不能进行值传递的方式使用.

## 原因分析
以 Mutex 为例
```go
type Mutex struct {  
  state int32  
  sema  uint32  
}
```

我们使用 Mutex 是为了不同 goroutine 之间共享某个变量, 所以需要让这个变量做到能够互斥, 不然该变量就会被互相被覆盖. Mutex 底层是由 `state` `sema` 控制的, 当 Mutex 变量被复制时, Mutex 的 state, sema 当时的状态也被复制走了, 但是由于不同 goroutine 之间的 Mutex 已经不是同一个变量了, 这样就会造成要么某个 goroutine 死锁或者不同 goroutine 共享的变量达不到互斥

## struct 如何与 不可复制 的类型一块使用
由上面可以看到不只是 sync 相关类型变量自身不能被复制，而且 sturct 嵌套 **「不可复制」** 类型变量时, 同样也不能被复制.

但是如果我将嵌套的不可复制变量改成指针类型变量呢, 是不是就解决了不能复制的问题 ?
```go
type URL struct {  
  Ip       string  
  mux     *sync.RWMutex  
}
```

这样确实解决了上述的不能复制问题. 但也引出了另外一个问题. 众所周知 Go 没有构造函数, 这就导致我们使用 URL 的时候都需要先去初始化 RWMutex, 不然就会造成同样很严重的空指针问题, 这个问题同样很棘手，也许哪个位置就忘了初始化这个 RWMutex.

根据 google groups 的讨论 [How to copy a struct which contains a mutex](https://groups.google.com/g/golang-nuts/c/imxjBLNJ9OY)。
![](attachments/Pasted%20image%2020240207110846.png)

发现大家的观点基本上都是一致的, 都不会去选用 struct 去嵌套指针类型的变量, 由此不建议 struct 去嵌套 「不可复制的」的指针类型变量. 最重要的原因: 没有一个工具能去准确的检测空指针.

所以一般情况下, 当 struct 嵌套了 **「不可复制」** 类型的变量时, 都需要传递的是 struct 类型变量的指针. 如:
```go
type URL struct {  
  Ip       string  
  mux     sync.RWMutex  
}  
  
func (c *URL) Clone() *URL {  
  newUrl := &URL{}  
  newUrl.Ip = c.Ip  
  return newUrl  
}
```

## 如何防止复制了不该复制的变量呢?
由于 Go 并不提供`重载`的功能, 所以并不能做到去重载 struct 的相关的被复制的方法.
Go 本身还不提供不能被复制的相关的编译强约束. 这样就有可能导致出现「不能被复制的类型」被复制过后蒙混过关. 那我们需要怎么做呢 ?

Go 提供了另外一个工具 `go vet` 来做补充, 用这个工具是能检测出来不可复制的类型是否被复制过.
```go
func main() {  
  var amux sync.Mutex  
  b := amux  
  b.Lock()  
  b.Unlock()  
}
```
```text
$ go vet main.go  
# command-line-arguments  
./main.go:7:7: assignment copies lock value to b: sync.Mutex
```

我们怎么把 go vet 与 日常开发结合起来呢?
1. 目前的 Goland, Vscode 都会集成 go vet 的相关功能, 如果你强迫症比较严重的话, 你就能发现有相关提示.    
2. 把 go vet 与 CI 流程结合起来, 其实更推荐使用 `golangci-lint` 这个 lint 工具来做 CI

## 不可复制的类型有哪些?
Go 提供的不可复制的类型基本上就是 sync 包内的所有类型: atomic.Value, Mutex, Cond, RWMutex, sync.Map, sync.Pool, WaitGroup.

这些内置的不可被复制的类型当被复制时配合 go vet是能够发现的. 但是下面这种场景你是否遇见过?
```go
package main  
  
import "fmt"  
  
type Books struct {  
  someImportantData []int  
}  
  
func DoSomething(otherBook Books) Books {  
  newBook := otherBook  
  // do something  
  for k := range newBook.someImportantData {  
    newBook.someImportantData[k]++ // just like this  
  }  
  return otherBook  
}  
  
func main() {  
  oldBook := Books{  
    someImportantData: make([]int, 0, 100),  
  }  
  
  oldBook.someImportantData = append(oldBook.someImportantData, 1, 2, 3)  
  fmt.Println("before DoSomething, old book:", oldBook.someImportantData)  
  DoSomething(oldBook)  
  fmt.Println("after DoSomething, old book:", oldBook.someImportantData)  
  // 使用oldBook.someImportantData 继续做某些事情  
}
```

结果：
```text
before DoSomething, old book: [1 2 3]  
after DoSomething, old book: [2 3 4]
```

# 参考
```bash
# Golang 的强人锁难
https://wenzhiquan.github.io/2021/04/24/2021-04-24-golang-lock/

# golang-mutex使用及原理分析
https://beangogo.cn/2021/02/19/golang-mutex/#gsc.tab=0

# Go 动手实操来了解Mutex互斥锁原理
https://juejin.cn/post/7227033124856496186
```