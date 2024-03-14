```table-of-contents
```
# 背景
Go 语言和典型的面向对象的语言不太一样，Go 在语法上是不支持面向对象的类、继承等相关概念的。但是，并不代表 Go 里面不能实现面向对象的一些行为比如继承、多态，在 Go 里面，通过 interface 完全可以实现诸如 C++ 里面的继承 和 多态的语法效果。
# 什么是接口
interface 就是字面意思——接口，C++中可以用虚基类表示；Java 中就是 interface。
interface 是方法签名的一个集合(`interface`中没有成员变量)，这些方法可以被**任一**类型通过方法实现。因此接口就是对象行为的申明（不是定义，仅仅表示方法签名，也可以称作函数原型）。
方法签名由方法名，输入参数，返回值三部分组成。
> 注意：我多次强调 任一类型 ，Golang 中所有类型都可以实现自己的方法，为了便于理解，本文还是使用 struct（结构体）来做示例。

# 接口的声明与实现
## 接口的声明
类似 **struct** 的申明方法，我们需要使用 interface 关键字来定义类型别名来方便使用接口。
![](attachments/Pasted%20image%2020231122181125.png)

从上面的例子中，我们可以发现，接口类型变量值和类型都是 _nil_ 。这是因为，我们这里声明的是 **Shape** 变量，它还没有指定动态类型，更没有指定任何动态值。


接口包含两个类型：
1》静态类型
静态类型就是指接口本身的类型，比如上图中的 **Shape** 就是静态类型。

2》动态类型（具体类型）
接口的动态类型就是传递给这个接口类型变量的形参或者右值的类型。
有时候 接口 的动态类型也叫做具体类型，因为我们获取接口类型的时候，它返回的是隐藏的动态值的类型。

## 接口的实现
![](attachments/Pasted%20image%2020231122180739.png)

在上面的程序中，我们定义了一个 Shape接口和 Rect结构体。然后 Rect 实现了 Area 和_Perimeter_方法，这就实现了 Shape接口的所有方法签名，所以我们就说 Rect 实现了 Shape 接口（这是 Golang 默认的，自动实现），我们并没有明确的显式指明 Rect实现了 Shape 接口。
我们可以看到，接口变量 _s_ 的动态类型就是 **Rect**，动态值就是 **Rect** 结构体对象 {5,4}。
> 我们用**动态**这个词，是因为我们也可以给接口变量 s 赋值另一个实现了 Shape 接口的结构体类型， 所以 s 实际指向的对象类型不是固定的，是动态的。

当一个类型实现了某个接口，这个类型的变量也可以用它所实现的接口类型表示（或者说用接口类型的变量去存放）。
> 或者说：interface 变量存储的是实现者的值。

![](attachments/Pasted%20image%2020231122182048.png)

![](attachments/Pasted%20image%2020231122182241.png)
上面程序中，我们删除了 **Perimeter** 方法，这个程序就编译不过并且抛出一个错误。
```go
program.go:22: cannot use Rect literal (type Rect) as type Shape in assignment:
        Rect does not implement Shape (missing Perimeter method)
```
从上面的错误中我们可以很容易的理解实现接口的要求：我们需要实现接口中申明的所有方法签名。
# interface 分类
Go的interface是由两种类型来实现的：`iface`和 `eface`。其中， `iface`表示的是包含方法的interface，例如：
```go
type Person interface {
    Print()
}
```
而 `eface`代表的是不包含方法的interface，即
```go
type Person interface {}
或者
var person interface{} = xxxx实体
```

# interface 的特性
## interface 是一种类型
```go
type I interface {  
    Get() int  
}
```
首先 **interface 是一种类型**，从它的定义可以看出来用了 type 关键字。
更准确的说 interface 是一种**具有一组方法的类型**，这些方法定义了 interface 的行为。存在方法的interaface称之为iface。

go 允许不带任何方法的 interface ，这种类型的 interface 叫 **empty interface， 空接口**， 或者称为eface。
**如果一个类型实现了一个 interface 中所有方法，我们说类型实现了该 interface**，所以所有类型都实现了 empty interface，因为任何一种类型至少实现了 0 个方法。go 没有显式的关键字用来实现 interface，只需要实现 interface 包含的方法即可。

### iface 底层原理
`iface`结构体中要同时储存方法信息，其数据结构如下图所示。
```go
type iface struct {
    tab  *itab
    data unsafe.Pointer
}

type itab struct {
    inter  *interfacetype
    _type  *_type
    link   *itab
    hash   uint32 // copy of _type.hash. Used for type switches.
    bad    bool   // type does not implement interface
    inhash bool   // has this itab been added to hash?
    unused [2]byte
    fun    [1]uintptr // variable sized
}

type interfacetype struct {
    typ     _type
    pkgpath name
    mhdr    []imethod
}

type _type struct {
    // 类型大小
    size       uintptr
    ptrdata    uintptr
    // 类型的 hash 值
    hash       uint32
    // 类型的 flag，和反射相关
    tflag      tflag
    // 内存对齐相关
    align      uint8
    fieldalign uint8
    // 类型的编号，有bool, slice, struct 等等等等
    kind       uint8
    alg        *typeAlg
    // gc 相关
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}

```
如上所示，`itab`结构体封装了`_type`结构体，同样利用`_type`储存类型信息。
`hash`是对`_type`结构体中`hash`的拷贝，提高类型断言的效率。`fun`指向方法信息的具体地址。
`interfacetype`，他描述的是接口静态类型（static interface type）信息。
`fun` 字段放置和接口方法对应的具体数据类型的方法地址，实现接口调用方法的动态分派，一般在每次给接口赋值发生转换时会更新此表，或者直接拿缓存的 itab。
![](attachments/Pasted%20image%2020231123104728.png)


### 静态类型和动态混合类型
Go语言中，每个变量都有唯一个**静态类型**（static interface type），这个类型是编译阶段就可以确定的。有的变量可能除了静态类型之外，还会有**动态混合类型**（dynamic concrete type）。
```go
//带函数的interface  
var r io.Reader   
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)  
if err != nil {  
    return nil, err  
}  
r = tty  
//不带函数的interface  
var empty interface{}  
empty = tty
```
**有函数的`iface`的例子**
第1行，`var r io.Reader`
![](attachments/Pasted%20image%2020231123105642.png)
第4行至第七行就是简单的赋值，得到一个`*os.File`的实例,暂且不看了。最后一句第十句`r = tty`
![](attachments/Pasted%20image%2020231123105719.png)

**无函数的`eface`的例子**
`var empty interface{}`
![](attachments/Pasted%20image%2020231123105758.png)

最后是`empty = tty`
![](attachments/Pasted%20image%2020231123105812.png)
## interface 变量存储的是实现者的值

```go
//1  
type I interface {      
    Get() int  
    Set(int)  
}  
  
//2  
type S struct {  
    Age int  
}  

// Get的接收者类型为：S
func(s S) Get()int {  
    return s.Age  
}  

// Set的接收者类型为 *S
func(s *S) Set(age int) {  
    s.Age = age  
}  
  
//3  
func f(i I){  
    i.Set(10)  
    fmt.Println(i.Get())  
}  
  
func main() {  
    s := S{}   
    f(&s)  //4  
}
```
这段代码在 `#1` 定义了 interface I，在 `#2` 用 struct S 实现了 I 定义的两个方法，接着在 `#3` 定义了一个函数 f 参数类型是 I，S 实现了 I 的两个方法就说 S 是 I 的实现者，执行 `f(&s)` 就完了一次 interface 类型的使用。

interface 的重要用途就体现在**函数 f 的参数中**，如果有多种类型实现了某个 interface，**这些类型的值都可以直接使用 interface 的变量存储**。

```go
s := S{}  
var i I //声明 i  
i = &s //赋值 s 到 i  
fmt.Println(i.Get())
```
不难看出 interface 的变量中存储的是实现了 interface 的类型的对象值。
在使用 interface 时，不需要显式在 struct 上声明要实现哪个 interface ，只需要实现对应 interface 中的所有方法即可（**必须是所有的 method**），go 会自动进行 interface 的检查，并在运行时执行从其他类型到 interface 的自动转换。即使实现了多个 interface，go 也会在使用对应 interface 时实现自动转换。

## interface 实现者的方法 receiver 如何选择
### 范例
![](attachments/Pasted%20image%2020231122184415.png)

在上面的程序中 Area方法属于 `*Rect` 类型，因此 Area 的接收者会去获取是 `Rect` 类型的指针（即使使用 Rect 类型的值去调用，底层也会转换成 `*Rect` 类型去调用）。但是，上诉程序将会编译不通过，go 编译器会报编译错误。
```go
program.go:27: cannot use Rect literal (type Rect) as type Shape in assignment: Rect does not implement Shape (Area method has pointer receiver)
```

在 Golang 中 `Rect` 类型 和 `*Rect` 类型是两种不同的类型，在使用 Rect 类型变量调用方法的时候，Golang 底层会去自动转换成指定的接收者类型去调用该方法。但是，在接口的实现中，`Rect` 实现的方法，不代表 `*Rect` 就实现了该方法。就像上面的程序，`*Rect` 实现了 Area 方法，但是 `Rect` 没有实现 Area 方法，Golang 底层不会默认 Rect 也实现了 Area方法。

正确的写法如下所示：
![](attachments/Pasted%20image%2020231122185001.png)

### 指针接收者和值接收者
**go 中函数都是按值传递即 passed by value**。interface 定义时并没有严格规定实现者的方法 receiver 是个 value receiver 还是 pointer receiver。

在我们上文的例子中调用 f 是 `f(&s)` 也就是 S 的指针类型，为什么不能是 `f(s)` 呢，如果是 s 会有什么问题？改成 f(s) 然后执行代码。
```go
cannot use s (type S) as type I in argument to f:  
	S does not implement I (Set method has pointer receiver)
```
这个错误的意思是 S 没有实现 I，哪里出了问题？**关键点是 S 中 set 方法的 receiver 是个 pointer *S** 。

上面代码中的 S 的 Set receiver 是 pointer，也就是实现 I 的两个方法的 receiver 一个是 value 一个是 pointer，使用 `f(s)`的形势调用，传递给 f 的是个 s 的一份拷贝，在进行 s 的拷贝到 I 的转换时，s 的拷贝不满足 Set 方法的 receiver 是个 pointer，也就没有实现 I。

那反过来会怎样，如果 receiver 是 value，函数用 pointer 的形式调用？
```go
type I interface {  
	Get() int  
	Set(int)  
}  
  
type SS struct {  
	Age int  
}  
  
func (s SS) Get() int {  
	return s.Age  
}  
  
func (s SS) Set(age int) {  
	s.Age = age  
}  
  
func f(i I) {  
	i.Set(10)  
	fmt.Println(i.Get())  
}  
  
func main(){  
  	ss := SS{}  
	f(&ss) //ponter  
	f(ss)  //value  
}
```
I 的实现者 SS 的方法 receiver 都是 value receiver，执行代码可以看到无论是 pointer 还是 value 都可以正确执行。

导致这一现象的原因是什么？
如果是按 pointer 调用，go 会自动进行转换，因为有了指针总是能得到指针指向的值是什么。如果是 value 调用，go 将无从得知 value 的原始值是什么，因为 value 是份拷贝。
**go 会把指针进行隐式转换得到 value，但反过来则不行**。
对于 receiver 是 value 的 method，任何在 method 内部对 value 做出的改变都不影响调用者看到的 value，这就是按值传递。

**范例**
```go
package main  
  
import (  
	"fmt"  
)  
  
type Animal interface {  
	Speak() string  
}  
  
type Dog struct {  
}  
  
func (d Dog) Speak() string {  
	return "Woof!"  
}  
  
type Cat struct {  
}  
  
//1  
func (c *Cat) Speak() string {  
	return "Meow!"  
}  
  
type Llama struct {  
}  
  
func (l Llama) Speak() string {  
	return "?????"  
}  
  
type JavaProgrammer struct {  
}  
  
func (j JavaProgrammer) Speak() string {  
	return "Design patterns!"  
}  
  
func main() {  
	animals := []Animal{Dog{}, Cat{}, Llama{}, JavaProgrammer{}}  
	for _, animal := range animals {  
		fmt.Println(animal.Speak())  
	}  
}
```
`//1` Cat 的 speak receiver 是 pointer，interface Animal 的 slice，Cat 的值是一个 value，同样会因为 receiver 不一致而导致无法执行。


## 如何判断 interface 变量存储的是哪种类型
一个 interface 被多种类型实现时，有时候我们需要区分 interface 的变量究竟存储哪种类型的值，这种区分能力叫 Type assertions (类型断言)。

go 可以使用 `value, ok := em.(T)`的形式做区分。
> **注：
> em 是 interface 类型的变量；
> T代表要断言的类型；
> value 是 interface 变量存储的值， 即接口对应的动态值；
> ok 是 bool 类型表示是否为该断言的类型 T ；如果不是，那么 ok 就是 false， 并且 vlaue 就等于 nil。
> **

![](attachments/Pasted%20image%2020231122182913.png)
上面的程序中，**Shape** 类型的变量 _s_ 的动态值是 **Cube** 类型的。我们可以使用 _s.(Cube)_ 语法来获取这个 s 的动态值，并且赋值给c。这样我们就可以用 _c_ 调用 **Area** 和*_Volume_*方法了，因为 c 是 **Cube** 类型的， **Cube** 同时实现了这两个方法。

### interface区分多种动态类型
如果需要区分多种类型，可以使用 switch 断言，更简单直接。
Switch语法很类似之前用的 类型断言的语法：`i.(type)`
> **注: i 是一个接口变量，type 是一个内置关键字。i.(type) 得到的是这个接口的动态类型，而不是类型断言中得到的是动态值。
> 另外，i.(type) 这个语法的断言方式只能在 switch 语句中使用。
>**

![](attachments/Pasted%20image%2020231122183849.png)

## 空 interface
### 定义
`interface{}` 是一个空的 interface 类型。一个 interface{} 类型的变量包含了2个指针，一个指针指向值的类型，另外一个指针指向实际的值。

一个类型如果实现了一个 interface 的所有方法就说该类型实现了这个 interface，空的 interface 没有方法，所以可以认为所有的类型都实现了 `interface{}`。
如果定义一个函数参数是 `interface{}` 类型，这个函数应该可以接受任何类型作为它的参数。
### 原理
`eface`的具体结构是：
```go
type eface struct {
    _type *_type
    data  unsafe.Pointer
}
```
`_type` 字段，表示空接口所承载的具体的实体类型，以及`data` 描述了具体的值。
`data`字段是`iface` 和 `eface`都有的结构，这个是一个内存指针，指向interface{}实例对象信息的存储地址，在这里，我们可以获取对象的具体属性的数值信息。
而interface{}的类型信息是存放在`_type`结构体中的。
```go
type _type struct {
    // 类型大小
    size       uintptr
    ptrdata    uintptr
    // 类型的 hash 值
    hash       uint32
    // 类型的 flag，和反射相关
    tflag      tflag
    // 内存对齐相关
    align      uint8
    fieldalign uint8
    // 类型的编号，有bool, slice, struct 等等等等
    kind       uint8
    alg        *typeAlg
    // gc 相关
    gcdata    *byte
    str       nameOff
    ptrToThis typeOff
}

```
我们可以看到`size`,`ptrdata`等表示interface{}对象的类型信息。
我们可以看到`size`,`ptrdata`等表示interface{}对象的类型信息，`hash`是其对应的哈希值，用于map等的哈希算法，`tflag`与反射相关，而`align`与`fieldalign`是用来内存对齐的，这与Go底层的内存管理机制有关，Go的内存管理机制类似于Linux中的伙伴系统（Buddy System），是以固定大小的内存块进行内存分配的，与这个大小进行对齐消除外碎片，提高内存利用率。

### 范例
```c
func doSomething(v interface{}){      
}
```

![](attachments/Pasted%20image%2020231122183414.png)

### QA
Q：如果函数的参数 v 可以接受任何类型，那么函数被调用时在函数内部 v 是不是表示的是任何类型？
> A: 并不是，虽然函数的参数可以接受任何类型，并不表示 v 就是任何类型，在函数 doSomething 内部 v 仅仅是一个 interface 类型。
> 之所以函数可以接受任何类型是在 go 执行时传递到函数的任何类型都被自动转换成 `interface{}`。


Q：既然空的 interface 可以接受任何类型的参数，那么一个 `interface{}`类型的 slice 是不是就可以接受任何类型的 slice ?
```go
func printAll(vals []interface{}) { //1  
	for _, val := range vals {  
		fmt.Println(val)  
	}  
}  
  
func main(){  
	names := []string{"stanley", "david", "oscar"}  
	printAll(names)  
}
```
上面的代码是按照我们的假设修改的，执行之后竟然会报 `cannot use names (type []string) as type []interface {} in argument to printAll` 错误，why？
> 说明：这个错误说明 go 没有帮助我们自动把 slice 转换成 `interface{}` 类型的 slice，所以出错了。**go 不会对 类型是`interface{}` 的 slice 进行转换** 。

`interface{}` 会占用两个字长的存储空间，一个是自身的 methods 数据，一个是指向其存储值的指针，也就是 interface 变量存储的值。因而 slice []interface{} 其长度是固定的`N*2`，但是 []T 的长度是`N*sizeof(T)`，两种 slice 实际存储值的大小是有区别的(文中只介绍两种 slice 的不同，至于为什么不能转换猜测可能是 runtime 转换代价比较大)。
参考：[# InterfaceSlice](https://github.com/golang/go/wiki/InterfaceSlice)
![](attachments/Pasted%20image%2020231122174113.png)

但是我们可以手动进行转换来达到我们的目的。
```go
var dataSlice []int = foo()  
var interfaceSlice []interface{} = make([]interface{}, len(dataSlice))  
for i, d := range dataSlice {  
	interfaceSlice[i] = d  
}
```

### eface和iface对比
**无函数的`eface`**
![](attachments/Pasted%20image%2020231123105124.png)

**有函数的`iface`**
![](attachments/Pasted%20image%2020231123105133.png)

## 多接口
一个类型可以实现多个接口，也可以理解为多继承。
![](attachments/Pasted%20image%2020231122180859.png)

## 接口的嵌套
在 Go 语言中，一个接口不能实现或者扩展另一个接口，只能通过组合的方式，把多个接口组合成一个新的接口。就像匿名嵌套结构体，这是可行的，所有内嵌接口中的方法签名也都属于父接口，父接口可以随意访问。
io package 中的一个接口：
```go
// ReadWriter is the interface that groups the basic Read and Write methods.
type ReadWriter interface {
    Reader
    Writer
}
```
ReadWriter 接口嵌套了 io.Reader 和 io.Writer 两个接口，实际上，它等同于下面的写法：
```go
type ReadWriter interface {
    Read(p []byte) (n int, err error) 
    Write(p []byte) (n int, err error)
}
```

![](attachments/Pasted%20image%2020231122184037.png)


上面的程序中，因为 Cube实现了 Area 和 Volume方法，所以 Cube同时实现了 Shape和Object接口。又因为 Material 接口是由 Shape 和Object组合而成，所以 Cube 也实现了 Material 接口。

> 注意，Go 语言中的接口不能递归嵌套，如下：
```go
// illegal: Bad cannot embed itself
type Bad interface {
    Bad
}
// illegal: Bad1 cannot embed itself using Bad2
type Bad1 interface {
    Bad2
}
type Bad2 interface {
    Bad1
}

```

## 接口的比较
两个接口可以通过比较运算符 `==` ，`!=`来进行比较。如果是两个空接口，那么这两个总是相等的。所以  `==` 操作符返回 true。
```go
var a, b interface{}
fmt.Println(a==b)	// true
fmt.Println(a!=b)	// false
```
如果两个不是空接口，那么只有当他们的动态类型和动态值都相等的情况下，他们才相等（`==`操作符返回 true）。
上面的情况都是基于动态类型都能比较的情况下进行的，如果动态类型不能比较呢（比如：slice，map，array， function 和结构体等），那么，执行比较操作的时候会抛出运行时异常。

interface 与 nil 的比较是挺有意思的，例子是最好的说明，如下例子：
```go
package main

import (
	"fmt"
	"reflect"
)

type State struct{}

func testnil1(a, b interface{}) bool {
	return a == b
}

func testnil2(a *State, b interface{}) bool {
	return a == b
}

func testnil3(a interface{}) bool {
	return a == nil
}

func testnil4(a *State) bool {
	return a == nil
}

func testnil5(a interface{}) bool {
	v := reflect.ValueOf(a)
	return !v.IsValid() || v.IsNil()
}

func main() {
	var a *State
	fmt.Println(testnil1(a, nil))
	fmt.Println(testnil2(a, nil))
	fmt.Println(testnil3(a))
	fmt.Println(testnil4(a))
	fmt.Println(testnil5(a))
}

```
运行后返回的结果如下
```go
false
false
false
true
true
```
因为一个 interface{} 类型的变量包含了2个指针，一个指针指向值的类型，另外一个指针指向实际的值。对一个 interface{} 类型的 nil 变量来说，它的两个指针都是0；
但是 `var a *State` 传进去后，指向的类型的指针不为0了，因为有类型了， 所以比较为 false。 interface 类型比较， 要是两个指针都相等，才能相等。


# interface的使用场景
## 实现泛型
## 隐藏具体的实现
隐藏具体的实现，是说我们提供给外部的一个方法(函数)，但是我们是通过 interface 接口的方式提供的，对调用方来说，只能通过 interface 里面的方法来做一些操作，但是内部的具体实现是完全不知道的。

例如我们常用的 context 包，就是这样设计的，如果熟悉 Context 具体实现的就会很容易理解。详细代码如下：
```go
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc) {
        c := newCancelCtx(parent)
        propagateCancel(parent, &c)
        return &c, func() { c.cancel(true, Canceled) }
    }
    
```
可以看到 WithCancel 函数返回的还是一个 Context interface，但是这个 interface 的具体实现是 cancelCtx struct。
```go
        // newCancelCtx returns an initialized cancelCtx.
        func newCancelCtx(parent Context) cancelCtx {
            return cancelCtx{
                Context: parent,
                done:    make(chan struct{}),
            }
        }
        
        // A cancelCtx can be canceled. When canceled, it also cancels any children
        // that implement canceler.
        type cancelCtx struct {
            Context     //注意一下这个地方
        
            done chan struct{} // closed by the first cancel call.
            mu       sync.Mutex
            children map[canceler]struct{} // set to nil by the first cancel call
            err      error                 // set to non-nil by the first cancel call
        }
        
        func (c *cancelCtx) Done() <-chan struct{} {
            return c.done
        }
        
        func (c *cancelCtx) Err() error {
            c.mu.Lock()
            defer c.mu.Unlock()
            return c.err
        }
        
        func (c *cancelCtx) String() string {
            return fmt.Sprintf("%v.WithCancel", c.Context)
        }

```

尽管内部实现上，下面三个函数返回的具体 struct （都实现了 Context interface）不同，但是对于使用者来说是完全无感知的。

```go
    func WithCancel(parent Context) (ctx Context, cancel CancelFunc)    //返回 cancelCtx
    func WithDeadline(parent Context, deadline time.Time) (Context, CancelFunc) //返回 timerCtx
    func WithValue(parent Context, key, val interface{}) Context    //返回 valueCtx


```
## 实现面向对象编程中的多态用法
interface 只是定义一个或一组方法函数，但是这些方法只有函数签名，没有具体的实现，这个 C++ 中的虚函数非常类似。在 Go 里面，如果某个数据类型实现 interface 中定义的那些函数，则称这些数据类型实现（implement）了这个接口 interface，这是我们常用的 OO 方式。
```go
    // 定义一个 SimpleLog 接口
    type SimpleLog interface {
        Print()
    }
    
    func TestFunc(x SimpleLog) {}
   
    // 定义一个 PrintImpl 结构，用来实现 SimpleLog 接口
    type PrintImpl struct {}
    // PrintImpl 对象实现了SimpleLog 接口的所有方法(本例中是 Print 方法)，就说明实现了  SimpleLog 接口
    func (p *PrintImpl) Print() {
    
    }
    
    func main() {
        var p PrintImpl
        TestFunc(p)
    }

```

## 空接口可以接受任何类型的参数
空接口比较特殊，它不包含任何方法：interface{}。
在 Go 语言中，所有其它数据类型都实现了空接口，如下：
```go
var v1 interface{} = 1
var v2 interface{} = "abc"
var v3 interface{} = struct{ X int }{1}
```
因此，当我们给 func 定义了一个 interface{} 类型的参数(也就是一个空接口)之后，那么这个参数可以接受任何类型，官方包中最典型的例子就是标准库 fmt 包中的 Print 和 Fprint 系列的函数。

# interface的作用
接口的用处总结：
- 泛型的使用
> 比如在一个函数中需要能接收不同类型的参数或者返回不同类型的值，而不是一开始就指定参数或者返回值的类型。
```go
func FuncName(arg1 interface{}, rest ...interface{}) interface{} {  
    // ...  
}
```
- 多态的使用。
在程序设计过程中，可能需要抽象出某些对象共同拥有的方法，这时候多种类型需要实现同一接口，然后通过接口变量指向具体对象来操作这些方法。



# 参考
```c
# Golang interface 接口详细原理和使用技巧
https://juejin.cn/post/7171288417324498980

## 理解 Go interface 的 5 个关键点
https://sanyuesha.com/2017/07/22/how-to-understand-go-interface/

# Golang 中的 Interface(接口)，全面解析
https://xie.infoq.cn/article/71d2e8c9ed41d036ad7f3ee94

# 深入了解Go的interface{}底层原理
https://juejin.cn/post/7105423957565636639
```