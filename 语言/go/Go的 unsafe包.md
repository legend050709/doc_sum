```table-of-contents
```
# 背景
Go语言在设计的时候，为了编写方便、效率高以及降低复杂度，被设计成为一门强类型的静态语言。强类型意味着一旦定义了，它的类型就不能改变了；静态意味着类型检查在运行前就做了。同时为了安全的考虑，Go语言是不允许两个指针类型进行转换的。

# golang的指针
## 基础知识
指针保存着一个值的内存地址，类型 `*T`代表指向`T` 类型值的指针。其零值为nil。
`&`操作符为它的操作数生成一个指针。
```go
i := 42
p = &i
```
`*`操作符则会取出指针指向地址的值，这个操作也叫做“解引用”。
```go
fmt.Println(*p) // 通过指针p读取存储的值*p = 42
```
## 为什么需要指针类型
```go
package main
import "fmt"
func double(x int) { 
   x += x
}

func main() { 
   var a = 3 
   double(a)
   fmt.Println(a) // 3
}
```
在 double 函数里将 a 翻倍，但是例子中的函数却做不到。因为 Go 语言的函数传参都是值传递。double 函数里的 x 只是实参 a 的一个拷贝，在函数内部对 x 的操作不能反馈到实参 a。

把参数换成一个指针就可以解决这个问题了。
```go
package main
import "fmt"

func double(x *int) { 
   *x += *x
    x = nil
}

func main() {
    var a = 3
    double(&a)
    fmt.Println(a) // 6
    p := &a
    double(p)
    fmt.Println(a, p == nil) // 12 false
}
```
## 指针的限制
相较于 C 语言指针的灵活，Go 语言里指针多了不少限制。不过这让我们：既可以享受指针带来的便利，又避免了指针的危险性。
### 限制一：指针不能参与运算
也就是说 Go 不允许对指针进行数学运算。
```go
package main
import "fmt"

func main() {
    a := 5
    p := a
    fmt.Println(p)
    p = &a + 3
}
```
上面的代码将不能通过编译，会报编译错误：
```text
invalid operation: &a + 3 (mismatched types *int and int)
```
### 限制二：不同类型的指针不允许相互转换
下面的程序同样也不能编译成功:
```go
package main
func main() {
    var a int = 100
    var f *float64
    f = *float64(&a)
}
```
### 限制三：不同类型的指针不能比较和相互赋值
这条限制同上面的限制二，因为指针之间不能做类型转换，所以也没法使用`==`或者`!=`进行比较了，同样不同类型的指针变量相互之间不能赋值。
比如下面这样，也是会报编译错误。
```go
package main
func main() {   
    var a int = 100
    var f *float64
    f = &a
}
```
# unsafe 包
## 介绍
go官方是不推荐使用unsafe的操作因为它是不安全的，它绕过了golang的内存安全原则，容易使你的程序出现莫名其妙的问题，不利于程序的扩展与维护。但是在很多地方却是很实用。在一些go底层的包中unsafe包被很频繁的使用。

Go 语言在 unsafe 包里其实还通过 unsafe.Pointer 提供了通用指针，通过这个通用指针以及 unsafe 包的其他几个功能又让使用者能够绕过 Go 语言的类型系统直接操作内存。例如：指针类型转换，读写结构体私有成员这样操作。
正是因为功能强大同时伴随着操作不慎读写了错误的内存地址即会造成的严重后果所以 Go 语言的设计者才会把这些功能放在 unsafe 包里。其实也没有想得那么不安全，掌握好了使用得当还是能带来很大的便利的，在一些偏向底层的源码中 unsafe 包使用的频率还是不低的。

## unsafe 定义
```go
package unsafe
//ArbitraryType仅用于文档目的，实际上并不是unsafe包的一部分,它表示任意Go表达式的类型。
type ArbitraryType int
//任意类型的指针，类似于C的*void
type Pointer *ArbitraryType
//确定结构在内存中占用的确切大小
func Sizeof(x ArbitraryType) uintptr
//返回结构体中某个field的偏移量
func Offsetof(x ArbitraryType) uintptr
//返回结构体中某个field的对其值（字节对齐的原因）
func Alignof(x ArbitraryType) uintptr
```

# unsafe.Pointer 
## 背景
## 介绍
## 特点
关于`unsafe.Pointer`的4个规则。
1. 任何指针都可以转换为`unsafe.Pointer`
2. `unsafe.Pointer`可以转换为任何指针
3. `uintptr`可以转换为`unsafe.Pointer`
4. `unsafe.Pointer`可以转换为`uintptr`

## 应用
### 实现指针类型转换
`unsafe.Pointer`是一种特殊意义的指针，它可以包含任意类型的地址，有点类似于C语言里的`void*`指针，全能型的。
```go
func main() {
	i:= 10
	ip:=&i

	var fp *float64 = (*float64)(unsafe.Pointer(ip))
	
	*fp = *fp * 3

	fmt.Println(i)
}

```
### 读写结构体的私有成员
我们都知道指针类型`*T`是不能计算偏移量的，也不能进行计算，但是`uintptr`可以。
我们可以把指针转为`uintptr`再进行偏移计算，这样我们就可以访问特定的内存了，达到对不同的内存读写的目的。
> 注：`uintptr`计算偏移，可以类比C语言中的Char* 或者 C 中的 uintptr。

```go
func main() {
	u:=new(user)
	fmt.Println(*u)

	pName:=(*string)(unsafe.Pointer(u))
	*pName="张三"

	pAge:=(*int)(unsafe.Pointer(uintptr(unsafe.Pointer(u))+unsafe.Offsetof(u.age)))
	*pAge = 20

	fmt.Println(*u)
}

type user struct {
	name string
	age int
}

```
第一个修改`user`的`name`值的时候，因为`name`是第一个字段，所以不用偏移，我们获取`user`的指针，然后通过`unsafe.Pointer`转为`*string`进行赋值操作即可。

第二个修改`user`的`age`值的时候，因为`age`不是第一个字段，所以我们需要内存偏移，内存偏移牵涉到的计算只能通过`uintptr`，所我们要先把`user`的指针地址转为`uintptr`，然后我们再通过`unsafe.Offsetof(u.age)`获取需要偏移的值，进行地址运算(+)偏移即可。

### string 和 []byte 零拷贝转换
实现字符串和 bytes 切片之间的零拷贝转换。

# unsafe.Offsetof
同上
# unsafe.Sizeof
## 作用

|类型|大小|
|---|---|
|`bool`|1个字节|
|`intN, uintN, floatN, complexN`|N/8个字节（例如float64是8个字节）|
|`int, uint, uintptr`|1个机器字|
|`*T`|1个机器字|
|`string`|2个机器字（data、len）|
|`[]T`|3个机器字（data、len、cap）|
|`map`|1个机器字|
|`func`|1个机器字|
|`chan`|1个机器字|
|`interface`|2个机器字（type、value）|
## 注意
Sizeof函数返回的大小只包括数据结构中固定的部分，例如字符串对应结构体中的指针和字符串长度部分，但是并不包含指针指向的字符串的内容。
```go
Sizeof takes an expression x of any type and returns the size in bytes of a hypothetical variable v as if v was declared via var v = x. The size does not include any memory possibly referenced by x. For instance, if x is a slice, Sizeof returns the size of the slice descriptor, not the size of the memory referenced by the slice.
```
如果x为一个切片，sizeof返回的大小是切片的描述符，而不是切片所指向的内存的大小。
如果x是一个string，sizeof返回的大小是stringHeader大小，而不是字符串所指向内存的大小。


- **字符串的sizeof大小**
```go
var str string = "hello"
var str2 string

fmt.Println(unsafe.SizeOf(str), unsafe.SizeOf(str2))
```
结构两个打印出来都是16。

```go
type StringHeader struct {
    Data uintptr
    Len  int
}
```
实际上字符串类型对应一个结构体，该结构体有两个域，**第一个域是指向该字符串的指针，第二个域是字符串的长度**，在64位机器上，每个域占8个字节，但是并不包含指针指向的字符串的内容，这也就是为什么sizeof始终返回的是16。

- **切片的sizeof大小**
```go
slice := []int{1,2,3}
fmt.Println(unsafe.Sizeof(slice)) //24
```
上面声明了一个切片，然后打印出sizeof的值为24，但是不管slice里的元素为多少，sizeof返回的数据都是24。
切片的底层实现。Slice 的数据结构定义，如下所示：
```go
type slice struct {
	array unsafe.Pointer
	len   int
	cap   int
}
```

![](attachments/Pasted%20image%2020231123202134.png)
切片（slice）是对数组一个连续片段的引用，所以切片是一个引用类型。

- **数组的sizeof大小**
这里如果换成一个数组呢？而不是一个切片：
```go
arr := [...]int{1,2,3,4,5}
fmt.Println(unsafe.Sizeof(arr)) //40
arr2 := [...]int{1,2,3,4,5,6}
fmt.Println(unsafe.Sizeof(arr)) //48
```
可以看到sizeof(arr)的值是在随着arr的元素的个数的增加而增加。
这是因为：**sizeof总是在编译期就进行求值，而不是在运行时，这意味着，sizeof的返回值可以赋值给常量**。在编译期求值，还意味着可以获得数组所占的内存大小，因为数组总是在编译期就指明自己的容量，并且在以后都是不可变的。


# 参考
```c
# Golang指针的使用限制和unsafe.Pointer的突破之路
https://studygolang.com/articles/32744
https://www.flysnow.org/2017/07/06/go-in-action-unsafe-pointer

```