```table-of-contents
```
# 前言
因为`Go`里面本来就有`Mutex`、`RWMutex`等锁，所以我们一般不会自己重新去实现一个锁，不过我们可以通过这个实现加深对锁和通道的理解。
# 原理
缓冲区长度为1的通道。
## lock
加锁的时候我们只是`向通道发送一个数据`，由于通道的缓冲区长度为1，因此后续的Lock操作都会被阻塞，直到通道里的数据被取出。

## unlock
解锁我们只需要`取出通道里的数据`，这样其他等待锁的Goroutine就可以往通道里面发送数据，从而获取到锁。

## try-lock
`select`的`default`支持**非阻塞的向通道发送数据**，因此我们可以使用select的default非阻塞的尝试向通道发送数据。

## 公平性
该锁是一个公平锁，这是因为channel保证了`先向channel发送数据的goroutine会先得到发送数据的权利`。主要是通过把被阻塞的goroutine加入一个`发送等待队列`，并在channel缓冲区里的数据被取出时`将发送等待队列头的数据放入缓冲区`，并把这个发送者goroutine添加到当前处理器的runnext，这样调度器在下次会唤醒这个由于拿不到锁而被阻塞的goroutine。

# 实现
## 数据结构
其实只是封装了一个channel，但是由于我们需要指定channel长度为1，因此添加了一个NewChannelLock()构造方法。
```go
type ChannelLock struct {
   c chan struct{}
}

func NewChannelLock() *ChannelLock {
   return &ChannelLock{
      c: make(chan struct{}, 1),
   }
}

```

## TryLock()、Lock()、Unlock()
注意，这里的Unlock()必须是在加锁的情况下才能调用，否则直接退出进程（和Mutex的语义保持一致）
```go
func (l *ChannelLock) TryLock() bool {
   select {
   case l.c <- struct{}{}:
      return true
   default:
      return false
   }
}

func (l *ChannelLock) Lock() {
   l.c <- struct{}{}
}

func (l *ChannelLock) Unlock() {
   select {
   case <-l.c:
   default:
      log.Fatal("sync: unlock of unlocked mutex")
   }
}

```
# 总结
- 可以很容易的使用channel实现一个公平锁，由于Go语言是不自带公平锁的，所以如果需要公平锁可以使用这个实现。当然，大部分情况下直接使用channel更加优雅。

- 大部分情况下还是推荐使用Mutex和RWMutex，因为这些锁已经保证了一定的公平性，可以防止饥饿的出现。

# 参考
```bash
# 使用通道实现一个公平锁
https://juejin.cn/post/7070109360482942983
```