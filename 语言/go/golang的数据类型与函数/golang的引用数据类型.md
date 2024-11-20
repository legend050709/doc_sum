```table-of-contents
```
# slice
## slice 与 array 的区别
讲到 slice，一定会与 array 做一个对比，那么两者的区别是什么呢？
### 区别一
其实最浅层的区别就是，slice 是可以变长的而 array 是定长的。
这是由于两者的底层数据结构决定的：
```go
type slice struct {  
    array unsafe.Pointer  
    len int  
    cap int  
}
```
slice 在空间不足的情况下，再 append，会发生扩容新建一个 newSlice，在空间扩展之后，通过 copy，将原有的 slice 拷贝到新的 newSlice 中。因此，扩容时，还会有一个内存地址变化。
如果将 slice 作为函数参数，并且在函数中修改 slice 的话，如果没有发生扩容，函数内的修改会更改函数外的源 slice，而如果在方法内部发生扩容的话，修改会发生在新的内存中，函数外的源 slice 不会受到影响。

而 array 需要在初始化的时候指明长度，并且这个长度是不可改变的，是 array 的类型中的组成部分。
### 区别二
除此之外两者还有一个重要的区别，slice 是 引用类型，数组是值类型。在函数参数中进行值传递时，slice 传的是指针的地址，而 array 会发生拷贝。所以在传递函数参数时注意一定不要传一个大 array，会十分消耗资源。

## Slice 使用姿势
上面说到 Slice 的一个特点就是可以自动扩容，那么他是怎么进行扩容的呢？

- 当 cap < 1024 时，每次扩容容量 * 2
- 当 cap >= 1024 时，每次扩容容量 *1.25

所以如果我们的 Slice 可以预知将会有比较大的容量时，提前分配容量大小可以节约大量的扩容时间，提升性能。此外，如果使用的 Slice 容量是提前知道的话，直接使用 index 赋值会比 append 性能更高。

## slice json marshal 的小坑
如果使用 `var x []Type` 初始化 slice，json marshal 之后的值会是 nil

如果使用 `[]Type{}` 或 `make([]Type)` 初始化 slice，json marshal 之后返回的值会是 `[]`。

# map
map 也是我们常用的数据结构，比较特别的一点是 Golang 中没有 set 这个数据结构，如果我们需要使用 set 的话，需要使用 `map[interface{}]struct{}` 来实现。
需要注意的一点是 Golang 中的 map 不是一个线程安全的数据结构，其数据结构如下：

![](attachments/Pasted%20image%2020240207144323.png)
![](attachments/Pasted%20image%2020240207144353.png)

如上所示，map 的内部使用 buckets 来存储 hash 后的数据。
map 实际上的值是一个指针（`*hmap`类型），如果函数传参使用 map 其实传的是这个 map 的指针（`*hmap`类型），所以在函数中修改 map 会修改到函数外源 map 的值，此时，如果有多个地方使用了这个 map，全部都会受到影响，这个需要十分注意，如果对 map 产生了并发写入，会直接抛出 fatal，不会被 recover(）捕获。

另外还有一点需要注意的就是如果删除了 map 中的 key，这个 map 并不会自动缩容，如果需要缩容的话可以让这个 map = make（新 map, 缩容后长度）)，然后将剩余的值赋给新 map。

# channel
最重要的一点就是 `channel 是有锁的`，是有锁的，是有锁的，重要的事情说三遍，所以所谓的用 channel 实现代码从而减少锁的使用是一个伪命题。
另外，channel也是引用类型，在传递channel时，实际传递的是`*hchan`类型。

channel 的数据结构如下：
![](attachments/Pasted%20image%2020240207144602.png)

可以看出来其底层其实是一个 ringbuffer，而 channel 放在 Golang 的 runtime 包里最重要的意义在于 channel 可以直接触发 goroutine 的调度，提高 cpu 的利用率。而由于其底层其实是有锁的，所以 channel 其实不适合用于高并发高性能编程的场景。


# 参考
```bash
https://wenzhiquan.github.io/2021/04/16/2021-04-16-slice-map-channel/
```