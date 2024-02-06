```table-of-contents
```
# 背景
通过这篇文章《[浅谈Go结构体的基本使用](https://juejin.cn/post/7198534533536481340 "https://juejin.cn/post/7198534533536481340")》，我们初步认识了**空结构体**。

使用`unsafe.SizeOf()`方法，明确知道了空结构体，它不占用存储空间（即“宽度”为0，宽度描述了一个类型的实例所占用的存储空间的字节数）
```go
s := struct{}{}
fmt.Println(unsafe.Sizeof(s)) //0
```
在项目代码中，我们经常都会看到空结构体`struct{}{}`的使用，所以肯定背后有一定的原因。那究竟它有什么作用，适合什么场景使用呢？

# 作用
因为**空结构体不占据内存空间**，因此**被广泛作为各种场景下的占位符使用**。
一是节省资源；
二是空结构体本身就具备很强的语义，即这里不需要任何值，仅作为占位符。

# 使用场景
**主要使用场景有3个**：

1. 实现集合类型
2. 实现空通道
3. 实现方法接收者

## 实现集合类型

**Go语言本身是没有集合类型（Set），通常是使用map来替代**。
但有个问题，就是==集合类型，只需要用到key（键），不需要用到value（值）==。

如果value使用bool来表示，实际会占用1个字节的空间，为了节省空间，这时空结构体就可以大显身手了。
```go
type Set map[int]struct{}

func main() {
  s := make(Set)
  s.add(1)
  s.add(2)
  s.add(3)
  s.remove(2)
  fmt.Println(s.exist(1))
  fmt.Println(s)

  //输出：
  //true
  //map[1:{} 3:{}]
}
func (s Set) add(num int) {
  s[num] = struct{}{}
}
func (s Set) remove(num int) {
  delete(s, num)
}
func (s Set) exist(num int) bool {
  _, ok := s[num]
  return ok
}

```
空结构体作为占位符，不会额外增加不必要的内存开销，很方便的就把问题给解决了

## 实现空通道
在Go的channel 的使用场景中，常常会遇到**通知型 channel**，其不需要发送任何数据，只是用于协调 Goroutine 的运行，用于流转各类状态或是控制并发情况。

这类情况就特别适合使用空结构体，只做个占位，不浪费内存空间。
```go
func main() {
  ch := make(chan struct{})
  go worker(ch)

  // Send a message to a worker.
  ch <- struct{}{}

  // Receive a message from the worker.
  <-ch
  println("AAA")

  //输出：
  //BBB
  //AAA
}

func worker(ch chan struct{}) {
  // Receive a message from the main program.
  <-ch
  println("BBB")

  // Send a message to the main program.
  close(ch)
}

```
## 实现方法接收者
使用结构体类型的变量作为方法接收者，有时结构体可以不包含任何字段属性。
可以用int或者string来替代，但它们都会占用内存空间，所以继续使用空结构体是比较合适的。
并且**也有利于未来针对该类型进行公共字段等的增加，容易扩展和维护**
```go
type T struct{}

func methodUse() {
  t := T{}
  t.Print()
  t.Print2()

  //输出：
  //哈哈哈Print
  //哈哈哈Print2
}

func (t T) Print() {
  fmt.Println("哈哈哈Print")
}
func (t T) Print2() {
  fmt.Println("哈哈哈Print2")
}

```
# 总结
针对空结构体的作用和使用场景，进行了详细的讲解。在之后的实际项目开发过程中，**只用占位不用实际含义，那么我们就都可以使用空结构体**，可以极大的节省不必要的内存开销。

# 参考
```c
https://juejin.cn/post/7199265829955223589
```