```table-of-contents
```
# 技巧
## 减少持有时间
### 缩小临界区，注意 defer 的使用
通过缩小临界区的方式，可以避免在加锁和解锁之间，由于有较为耗时的代码，导致锁持有时间过长，从而造成性能问题。

如果代码像下面这样
```go
Func SomeFunc() {  
    // do sth  
    mu.Lock()  
    defer mu.Unlock()  
    ...  
    ...// long code  
    ...  
}
```
如果 defer 之后的代码特别耗时，那这个 mu 锁的时间就会非常长了，会拖慢整个程序的速度。

#### 不用defer释放锁的问题
有人可能会问了，那我显示调用 mu.Lock() 和 mu.Unlock() 不就好了，像下面这样
```go
func SomeFunc() {
    mu.Lock()
    ... // do sth
    mu.Unlock()
    ... // long code
}
```
答案是不太行，除非 do sth 的时候绝对不会出错，否则 mu 将无法得到释放，如果有令一个线程再次对 mu 加锁将产生死锁，而用 `defer mu.Unlock()` 的话，能够保证就算出错了 mu 也可以正确释放，这点比提升性能更加重要。

#### 匿名函数包装
使用匿名函数的方式把需要加锁的代码包一层进行调用，可以避免上述提到的影响
```go
func CheckUser(name, password string) bool {  
    var realPwd string  
    var exist bool  
    func () {  
        mu.Lock()  
        defer mu.Unlock()  
        realPwd, exist: = Users[name]  
    }()
    return exist & realPwd == password  
}
```

## 优化锁的粒度
最常用的方式就是使用分段锁。比如全局锁 ------> 基于key的锁（锁的数组）。
![](attachments/Pasted%20image%2020240207154542.png)

使用分段锁的方式：
```go
type SafeRander struct {  
    pos     uint32  
    randers [128]*rand.Rand  
    locks   [128]*sync.Mutex  
}  
  
func (sr *SafeRander) Intn(n int) int {  
    x := atomic.AddUint32(&sr.pos, 1)  
    x %= 128  
    sr.locks[x].Lock()  
    n = sr.randers[x].Intn(n)  
    sr.locks[x].Unlock()  
    return n  
}
```

## 读写分离
使用读写锁可以大大降低整个锁的持有时间：
```go
type Counter struct {
    count int
    mutex sync.Mutex
}

func (w *Counter) Count() {
    w.mutex.Lock()
    defer w.mutex.Unlock()
    w.count++
    time.Sleep(time.Microsecond)
}

func (w *Counter) Read() {
    w.mutex.Lock()
    defer w.mutex.Unlock()
    _ = w.count
    time.Sleep(time.Microsecond)
}

type RWCounter struct {
    count int
    mutex sync.RWMutex
}

func (w *RWCounter) Count() {
    w.mutex.Lock()
    defer w.mutex.Unlock()
    w.count++
    time.Sleep(time.Microsecond)
}

func (w *RWCounter) Read() {
    w.mutex.RLock()
    defer w.mutex.RUnlock()
    _ = w.count
    time.Sleep(time.Microsecond)
}

```
![](attachments/Pasted%20image%2020240207154838.png)
通过 Benchmark 可以看出使用读写锁比单纯使用普通锁的性能更好，效率更高

## 使用原子操作
相比读写锁，使用原子操作具有更高的性能，因为原子操作不会触发 Go 的调度，也不会阻塞执行流，可以使用 Golang 的 sync/atomic 包中的提供的方法。
![](attachments/Pasted%20image%2020240207155246.png)

使用原子操作的 AtomicCounter 的 Benchmark，证明原子操作的性能是更高的
```go
type AtomicCounter struct {
    count int32
}

func (c *AtomicCounter) Count() {
    atomic.AddInt32(&c.count, 1)
}

func (c *AtomicCounter) Read() {
    _ = atomic.LoadInt32(&c.count)
}

```
![](attachments/Pasted%20image%2020240207155137.png)

# 避免踩坑
## 不要拷贝 Mutex
如果在我们使用过程中直接传入 mutex 对象作为参数的话，会由于传值而发生拷贝，所以会生成新的 Mutex，导致无法正确的加锁。

Goland 会对这种不正确的用法进行提示，非常的人性化。
![](attachments/Pasted%20image%2020240207155435.png)

所以如果要使用同一个锁进行加锁可以使用传递指针的形式：
```go
func Worker(m *sync.Mutex) {
    m.Lock()
    defer m.Unlock()
    // do sth, like counting
}

func main() {
    var mu sync.Mutex
    go Worker(&mu)
    time.Sleep(time.Second)
}
```

## Mutex 不可重入
下面这段代码会发生死锁，原因是 Golang 中 Mutex 是不可重入的，两次加锁会导致自己等待加锁的自己解锁，形成死锁。
```go
func example() {
    var m sync.Mutex

    m.Lock()
    defer m.Unlock()

    m.Lock()
    defer m.Unlock()
}
```
## atomic.Value 误用
下面这段代码对 p 这个 map 可能会进行并发读写，从而产生 fatal 错误，使用 atomic 的做法原则上存入的对象都应该是只读的。
```go
func ProcessRequest() {
    p := pacing.Load().(map[string]int) // 取数据，转化为 `map[string]int` 类型
    value = p[x]
    ...
    p[x] = adjust(value)
    pacing.Store(p)
}
```

即：**atomic.Value 应当只存入只读对象**。


# 参考
```bash
# Golang 的强人锁难
https://wenzhiquan.github.io/2021/04/24/2021-04-24-golang-lock/
```