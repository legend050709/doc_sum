```table-of-contents
```
# 背景
我们知道 Golang 是一门擅长高并发的编程语言，可以通过 Goroutine 快速地创建并发任务，但是如何有效地管理这些执行的 Goroutine 是一个值得思考的问题。
# Goroutine 并发控制的几种方式
通常我们有下面几种方式实现 Goroutine 的控制：

- 使用 **WaitGroup**，根 goroutine 通过 `add()` 来记录已经开启的 Goroutine 数量，通过 `wait()` 来等待执行任务的 goroutine 的 `done()`，实现同步的工作；
- 使用 for/select + stop channel，通过向 stop channel 中传递 stop signal 实现 goroutine 的结束；
- 使用 **Context**， 可以控制具有复杂层级关系的 goroutine 任务，此时使用前两种方式实现会比较复杂，使用 context 会更优雅。


# 参考
```c

```