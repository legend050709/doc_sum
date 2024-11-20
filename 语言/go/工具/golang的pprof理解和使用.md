```table-of-contents
```
# pprof 能做什么
![](attachments/Pasted%20image%2020240207160743.png)
通过 Golang 提供的 pprof 工具，我们可以很轻松的对程序的 CPU，Heap，Goroutine 等进行采样，并且 pprof 工具会帮我们分析好采样数据并且以网页或可视化的方式让我们可以很直观的观察采样结果。

# pprof 排查
[https://github.com/wolfogre/go-pprof-practice](https://github.com/wolfogre/go-pprof-practice) 我们可以使用这个代码仓库来探索 pprof 的使用

## 前置准备
```go
package main
import (
    // 会自动注册 handler 到 http server，方便通过 http 接口获取程序运行采样报告
    _ "net/http/pprof"
    ...
)

func main() {
    runtime.GOMAXPROCS(1) // 限制 CPU 使用数，避免过载
    runtime.SetMutexProfileFraction(1) // 开启对锁调用的跟踪
    runtime.SetBlockProfileRate(1) // 开启对阻塞操作的跟踪
    go func() {
        // 启动一个 http server
        // 注意 pprof 相关的 handler 已经自动注册过了
        if err := http.ListenAndServe(":6060", nil); err != nil{
            log.Fatal(err)
        }
        os.Exit(0)
    }()
}

```

然后我们就可以在浏览器中输入 `http://localhost:6060/debug/pprof/` 查看代码运行情况了，效果如下：
![](attachments/Pasted%20image%2020240207160835.png)

## CPU
### 采样
```bash
go tool pprof http://localhost:6060/debug/pprof/profile?seconds=10
```

可以获得一个采样文件：
![](attachments/Pasted%20image%2020240207160919.png)

### top N
输入 `top 10`，可以看到消耗资源前十的方法：
![](attachments/Pasted%20image%2020240207160943.png)

flat 表示样本中当前函数占用的 CPU 时间；
cum 表示当前函数以及当前函数调用其他函数的时间；
cum - flat 则为当前函数调用其他函数耗费的时间

所以：
1. flat == cum: 表示函数中没有调用其他函数
2. flat == 0: 表示函数中只有其他函数的调用

### list
输入 `list regx`，可以根据输入的正则表达式查找代码行
![](attachments/Pasted%20image%2020240207161124.png)
可以很明显的看出来哪行代码可能有问题

## Heap
### 采样
```bash
go tool pprof http://localhost:6060/debug/pprof/heap?seconds=10

```
可以获得一个采样文件：
![](attachments/Pasted%20image%2020240207161234.png)

## goroutine
### 采样
```bash
go tool pprof http://localhost:6060/debug/pprof/goroutine?seconds=10
```
可以获得一个采样文件：
![](attachments/Pasted%20image%2020240207161413.png)

### 火焰图
![](attachments/Pasted%20image%2020240207161453.png)
通过火焰图我们可以看出来是 Drink 函数中调用的 time.sleep 占用资源最多，用 source 视图查看：
![](attachments/Pasted%20image%2020240207161514.png)

## Mutex、Block
也是类似的排查方式

# pprof 指标采样的流程和原理
## CPU 采样
- 采样对象：函数调用和它们的占用时间
- 采样率：100 次/秒，固定值
- 采样时间：从手动启动到手动结束

![](attachments/Pasted%20image%2020240207161609.png)

采样过程：

- 操作系统：每 10ms 向进程发送一次 SIGPROF 信号
- 进程：每次接收到 SIGPROF 会记录调用栈
- 写缓冲：每 100ms 读取一次已经记录的调用栈并写入输出流

![](attachments/Pasted%20image%2020240207161627.png)

## Goroutine & ThreadCreate 采样
**Goroutine**
记录所有用户发起且在运行中的 goroutine（即入口非 runtime 开头的）和 runtime.main 的调用栈信息


**ThreadCreate**
记录程序创建的所有系统线程的信息

![](attachments/Pasted%20image%2020240207161746.png)


## Heap 采样
- 采样程序通过内存分配器在**堆上**分配和释放的内存，记录分配/释放的**大小和数量**
- 采样率：每分配 512KB 记录一次，可在运行开头修改，设置为 1 表示每次分配均记录
- 采样时间：从程序运行开始到采样时
- 采样指标：alloc_space, alloc_objects, inuse_space, inuse_objects
- 计算方式：inuse = alloc - free

# 参考
```text
# Golang 性能问题排查定位
https://wenzhiquan.github.io/2021/05/21/2021-05-21-golang-pprof/
```