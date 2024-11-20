```table-of-contents
```
# 背景

## 零值

Go 有四类数据类型：

- 基本类型：数字、字符串和布尔值
- 聚合类型：数组array和结构strcut
- 引用类型：指针pointer、切片slice、映射map、函数function和通道channel
- 接口类型：接口

零值是指基本数据类型和指针的初始值。
数值型零值为`0`、string的零值为`""`、bool的零值为`false`、指针的零值为`nil`。

# var
声明一个变量：
```go
var 变量名 数据类型

如下所示：
var a int
var b string
```

我们使用 var 关键字，声明两个变量，然后就可以在程序中使用。
当我们不指定变量的默认值的时候呢，这些变量的默认值是它所属类型的零值。
> 比如上面的 int 型它的零值为 0，string 的零值为 ""，引用类型的零值为 nil。

## 声明多个变量

```go

var (
	a int
	b bool
	c string
	
)
```


# new
```go
// The new built-in function allocates memory. The first argument is a type, // not a value, and the value returned is a pointer to a newly // allocated zero value of that type.

func new(Type) *Type
```

因此，`new` 的作用是**根据传入的类型分配一片内存空间并返回指向这片内存空间的指针**。
> 注：**new(Type) 等价于 &Type{ }**

```go
i := new(int)

var v int
i := &v

上述代码片段中的两种不同初始化方法是等价的，它们都会创建一个指向 `int` 零值的指针。
```


```go
package main

import `fmt`

func main() {
    var a *int = new(int)
    fmt.Println(a) // a是一个int指针类型 0xc00000a0c8
    // a = 100      // 此时不能用a直接赋值，因为a是int指针类型，而100是int类型，两者类型不同，所以不同赋值
    *a = 100 // 必须要在a前面加一个星号*，*a 表示内存地址指向的那块内存空间，*a=100表示将数字100存储到a(内存地址)指向的那块内存空间中；
    fmt.Println(*a)

    var b *[]int = new([]int)
    // (*b)[0] = 1      // 此时b是一个指针类型，它的值是一个内存地址
    fmt.Println(b, &b)       // b的值是一个内存地址，而这个内存地址也需要一块内存空间去存储，所以 &b是取b的内存地址
    fmt.Printf("%p\n", b)    // 打印：0xc0000044c0, 这是b的值
    fmt.Println(*b)          // 是一个空切片，打印：[]
    *b = make([]int, 5, 100) // 给b(内存地址)分配一块内存空间并初始化切片长度为5，容量为100，这里才可以进行索引操作，如下
    (*b)[0] = 10
    (*b)[1] = 20
    (*b)[2] = 30
    fmt.Println(*b) // 打印：[10 20 30 0 0]

    var c = make([]int, 5, 20)
    c[0] = 100
    c[1] = 200
    fmt.Println(c) // 打印：[100 200 0 0 0]
    // 通过上面的例子可以看出，用new()初始化引用类型的变量是多余的
}
```



## new 实际较少使用
 如下**struct 初始化的过程**，可以说明不使用 new 一样可以完成 struct 的初始化工作。
 
```go
type Foo struct {  
    name string  
    age  int  
}  
  
//声明初始化  
var foo1 Foo  
fmt.Printf("foo1 --> %#v\n ", foo1) //main.Foo{age:0, name:""}  
foo1.age = 1  
fmt.Println(foo1.age)  
  
//struct literal 初始化  
foo2 := Foo{}  
fmt.Printf("foo2 --> %#v\n ", foo2) //main.Foo{age:0, name:""}  
foo2.age = 2  
fmt.Println(foo2.age)  
  
//指针初始化  
foo3 := &Foo{}  
fmt.Printf("foo3 --> %#v\n ", foo3) //&main.Foo{age:0, name:""}  
foo3.age = 3  
fmt.Println(foo3.age)  
  
//new 初始化  
foo4 := new(Foo)  
fmt.Printf("foo4 --> %#v\n ", foo4) //&main.Foo{age:0, name:""}  
foo4.age = 4  
fmt.Println(foo4.age)  
  
//声明指针并用 new 初始化  
var foo5 *Foo = new(Foo)  
fmt.Printf("foo5 --> %#v\n ", foo5) //&main.Foo{age:0, name:""}  
foo5.age = 5  
fmt.Println(foo5.age)
```


## 范例
```go
 a := new(int)
 fmt.Printf("类型为:%T, 值为:%v\n", a, a)
 fmt.Printf("类型为:%T, 值为:%v\n", *a, *a)
 
 b := new(string)
 fmt.Printf("类型为:%T, 值为:%v\n", b, b)
 fmt.Printf("类型为:%T, 值为:%v\n", *b, *b)
 
 c := new(*int)
 fmt.Printf("类型为:%T, 值为:%v\n", c, c)
 fmt.Printf("类型为:%T, 值为:%v\n", *c, *c)
 
 运行结果:
 类型为:*int, 值为:0xc0000a6058
 类型为:int, 值为:0               
 类型为:*string, 值为:0xc000088220
 类型为:string, 值为:             
 类型为:**int, 值为:0xc0000ca020  
 类型为:*int, 值为:<nil>  
```


# make
make() 用来分配引用类型的内存，比如map、slice以及channel。并且直接返回这三种类型本身，而不是类型指针。

**new()用来分配除了引用类型以外的所有其他类型的内存，比如int、数组等； new返回类型的指针**；

```go
// The make built-in function allocates and initializes an object of type
// slice, map, or chan (only). Like new, the first argument is a type, not a
// value. Unlike new, make's return type is the same as the type of its
// argument, not a pointer to it. The specification of the result depends on
// the type:
//  Slice: The size specifies the length. The capacity of the slice is
//  equal to its length. A second integer argument may be provided to
//  specify a different capacity; it must be no smaller than the
//  length. For example, make([]int, 0, 10) allocates an underlying array
//  of size 10 and returns a slice of length 0 and capacity 10 that is
//  backed by this underlying array.
//  Map: An empty map is allocated with enough space to hold the
//  specified number of elements. The size may be omitted, in which case
//  a small starting size is allocated.
//  Channel: The channel's buffer is initialized with the specified
//  buffer capacity. If zero, or the size is omitted, the channel is
//  unbuffered.
func make(t Type, size ...IntegerType) Type

```


注：
new 的作用是根据传入的类型分配一片内存空间并返回指向这片内存空间的指针，此时编译器知道你需要使用多少内存。
而 make 的作用是初始化内置的数据结构，也就是slice、map和 channel，此时编译器不知道你需要使用多少内存，因为这些数据结构占用的内存是运行时才能知晓的。
slice、map、channel 这些类型它是复合类型数据结构，通常是一个结构体+堆内存，因此 make 的额外作用就是初始化这些数据和指针，从这一点看，make 的作用是申请内存，并且初始化数据。
```go
slice := make([]int, 0)
mp := make(map[string]string, 100)
ch := make(chan int, 5)
```
![](attachments/Pasted%20image%2020231213150439.png)

## make(T, args) 返回的是 T 的 引用
如果不特殊声明，go 的函数默认都是按值传参，即通过函数传递的参数是值的副本，在函数内部对值修改不影响值的本身。
make(T, args) 返回的值通过函数传递参数之后可以直接修改，即 map，slice，channel 这些**引用类型** 通过函数传参之后，在函数内部的修改将影响函数外部的值。

```go
func modifySlice(s []int) {  
    s[0] = 1  
}  
  
s2 := make([]int, 3)  
fmt.Printf("%#v", s2) //[]int{0, 0, 0}  
modifySlice(s2)  
fmt.Printf("%#v", s2) //[]int{1, 0, 0}
```

这说明 **make(T, args) 返回的是引用类型，在函数内部可以直接更改原始值，对 map 和 channel 也是如此**。

```go
func modifyMap(m map[int]string) {  
m[0] = "string"  
}  
  
func modifyChan(c chan string) {  
c <- "string"  
}  
  
m2 := make(map[int]string)  
if m2 == nil {  
fmt.Printf("m2 is nil --> %#v \n ", m2)  
} else {  
fmt.Printf("m2 is not nill --> %#v \n ", m2) //map[int]string{}  
}  
  
modifyMap(m2)  
fmt.Printf("m2 is not nill --> %#v \n ", m2) // map[int]string{0:"string"}  
  
  
c2 := make(chan string)  
if c2 == nil {  
fmt.Printf("c2 is nil --> %#v \n ", c2)  
} else {  
fmt.Printf("c2 is not nill --> %#v \n ", c2)  
}  
  
go modifyChan(c2)  
fmt.Printf("c2 is not nill --> %#v ", <-c2) //"string"
```


## 引用类型的初始化

如下所示，map类型的初始化：
```go
package main

import `fmt`

func main() {
    var m map[string]int  // map声明后未初化是nil
    fmt.Println(m == nil) // true

    // 初始化 方式一：
    m = map[string]int{}
    // 对未初始化的map直接赋值会抛出异常 panic: assignment to entry in nil map
    m["cai"] = 22000
    m["kung"] = 20000
    fmt.Println(m, len(m))

    // 初始化 方式二：
    n := make(map[string]int)
    n["guang"] = 12000
    n["wang"] = 10000
    n["guo"] = 8000
    n["liang"] = 9000
    fmt.Println(n, len(n))

    value, ok := n["wangg"]
    if ok {
        fmt.Println("value:", value)
    } else {
        fmt.Println("value is not exists.")
    }
}
```

## 范例
```go
slice := make([]int, 0, 100)
hash := make(map[int]bool, 10)
ch := make(chan int, 5)

注：
	slice 是一个包含 data、cap 和 len 的结构体 reflect.SliceHeader；
	hash 是一个指向 runtime.hmap 结构体的指针；
	ch 是一个指向 runtime.hchan 结构体的指针；
```

# new和make的区别

make()用来分配引用类型的内存，比如map、slice以及channel；

new()用来分配除了引用类型以外的所有其他类型的内存，比如int、数组等； new返回类型的指针；

- （1）make 只能用来分配及初始化类型为 slice、map、chan 这样引用类型的数据，而 new 可以分配任意类型的数据。
> new常给int、string、数组分配内存。实际上，new函数并不常用，大家更喜欢使用短语句声明的方式。即 new(Type) 等价于 &Type{}。
- （2）new 分配返回的是指针，即类型 *Type。make 返回引用，即 Type。
> new返回的是指向类型的指针；而make的返回类型与其参数的类型相同，而不是指向它的指针，因为这三种数据类型(slice, map, chan)本身就是引用类型。
- （3）new分配空间后，是将内存清零，并没有初始化内存；而make分配空间后，是初始化内存，而不是清零。

## 范例
```go
package main

import "fmt"

func main() {
    p := new([]int) //p == nil; with len and cap 0
    fmt.Println(p)

    v := make([]int, 10, 50) // v is initialed with len 10, cap 50
    fmt.Println(v)

    /*********Output****************
        &[]
        [0 0 0 0 0 0 0 0 0 0]
    *********************************/

    (*p)[0] = 18        // panic: runtime error: index out of range
                        // because p is a nil pointer, with len and cap 0
    v[1] = 18           // ok
    
}
```


# 小结
![](attachments/Pasted%20image%2020231213150659.png)

# 参考
```c
https://qqailab.cn/2019/10/17/golang-make-new.html
```