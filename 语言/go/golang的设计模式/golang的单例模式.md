```table-of-contents
```
# 概述
在面向对象编程语言中，单例模式(Singleton pattern)确保一个类只有一个实例，并提供对该实例的全局访问。

那么Go语言中，单例模式确认一个类型只有一个实例，并提供对改实例的全局访问，一般就是直接访问全局变量即可。

# 介绍
单例模式是一种创建型设计模式，它保证一个类只有一个实例，并提供一个全局访问点来访问这个实例。在整个应用程序中，所有对于这个类的访问都将返回同一个实例对象。

# 实现

## 全局变量实现
在 Go 语言中，全局变量会在程序启动时自动初始化。因此，如果在定义全局变量时给它赋值，则对象的创建也会在程序启动时完成，可以通过此来实现单例模式。

```go
type MySingleton struct {
    // 字段定义
}

var mySingletonInstance = &MySingleton{
    // 初始化字段
}

func GetMySingletonInstance() *MySingleton {
    return mySingletonInstance
}

```


## 加锁实现

### 思路

通常而言， 单例实例会在结构体首次初始化时创建。 
为了实现这一操作， 我们在结构体中定义一个 `get­Instance`获取实例方法。 该方法将负责创建和返回单例实例。 创建后， 每次调用 `get­Instance`时都会返回相同的单例实例。

协程方面又有什么需要注意的吗？ 每当多个协程想要访问实例时， 单例结构体就必须返回相同的实例。

### 实现方法 
```go
package main

import (
    "fmt"
    "sync"
)

var lock = &sync.Mutex{}

type single struct {
}

var singleInstance *single

func getInstance() *single {
    if singleInstance == nil {
        lock.Lock()
        defer lock.Unlock()
        if singleInstance == nil {
            fmt.Println("Creating single instance now.")
            singleInstance = &single{}
        } else {
            fmt.Println("Single instance already created.")
        }
    } else {
        fmt.Println("Single instance already created.")
    }

    return singleInstance
}
```

#### 分析
**分析**：
- (1)最开始时会有 `nil`检查， 确保 `single­Instance`单例实例在最开始时为空。 这是为了防止在每次调用 `get­Instance`方法时都去执行消耗巨大的锁定操作。 如果检查不通过， 则就意味着 `single­Instance`字段已被填充。
    
- (2)`single­Instance`结构体将在锁定期间创建。
    
- (3)在获取到锁后还会有另一个 `nil`检查。 这是为了确保即便是有多个协程绕过了第一次检查， 也只能有一个可以创建单例实例。 否则， 所有协程都会创建自己的单例结构体实例。

### 测试
```go
package main

import (
    "fmt"
)

func main() {

    for i := 0; i < 30; i++ {
        go getInstance()
    }

    // Scanln is similar to Scan, but stops scanning at a newline and
    // after the final item there must be a newline or EOF.
    fmt.Scanln()
}

结果：
Creating single instance now.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
Single instance already created.
```

##  sync.Once实现
```go
 package singleton

 import "sync"

 type singleton struct {
     // 单例对象的状态
 }

 var (
     instance *singleton
     once     sync.Once
 )

 func GetInstance() *singleton {
     once.Do(func() {
         instance = &singleton{}
         // 初始化单例对象的状态
     })
     return instance
 }

```
在 `GetInstance` 函数中，我们使用 `once.Do` 方法来执行一个初始化单例对象。由于 `once.Do` 方法是基于原子操作实现的，因此可以保证并发安全，即使有多个协程同时调用 `GetInstance` 函数，最终也只会创建一个对象。

### sync.once 的实现
```go
func (o *Once) Do(f func()) {
    if atomic.LoadUint32(&o.done) == 0 {
        // Outlined slow-path to allow inlining of the fast-path.
        o.doSlow(f)
    }
}

func (o *Once) doSlow(f func()) {
    o.m.Lock()
    defer o.m.Unlock()
    if o.done == 0 {
        defer atomic.StoreUint32(&o.done, 1)
        f()
    }
}

```

如上所示，sync.once 的实现和上面的 加锁实现单例模式，基本类似。

### 优点

相对于`init` 方法和使用全局变量定义赋值单例模式的实现，`sync.Once` 实现单例模式可以==延迟初始化==，即在第一次使用单例对象时才进行创建和初始化。这可以避免在程序启动时就进行对象的创建和初始化，以及可能造成的资源的浪费。

相对于使用互斥锁实现单例模式，使用 `sync.Once` 实现单例模式的优点在于更为简单和高效。sync.Once提供了一个简单的接口，只需要传递一个初始化函数即可。相比互斥锁实现方式需要手动处理锁、判断等操作，使用起来更加方便。

# 参考
```bash
# Go 常用设计模式
https://refactoringguru.cn/design-patterns/go

# go 的 设计模式
https://refactoringguru.cn/design-patterns/factory-method
```