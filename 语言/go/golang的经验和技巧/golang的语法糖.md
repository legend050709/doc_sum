```table-of-contents
```
# 语法糖介绍

语法糖（Syntactic sugar）的概念是由英国计算机科学家彼得·兰丁提出的，用于表示编程语言中的某种类型的语法，这些语法不会影响功能，但使用起来却很方便。  
语法糖，也称糖语法，这些语法不仅不会影响功能，**编译后的结果跟不使用语法糖也一样。**  
语法糖，有可能让代码编写变得简单，也有可能让代码可读性更高，但有时也会给你一个意外让您的代码出问题。

# `...`
## 可变长参数
### 介绍
`Go`语言允许一个函数有任意数量的值作为参数，`Go`语言内置了`...`操作符，在函数的最后一个形参才能使用`...`操作符。

### 注意事项
使用`...`操作符必须注意如下事项：

- （1）可变长参数必须在函数列表的最后一个；
- （2）把可变长参数当切片(传入实参时，切片展开；形参当切片处理)来解析，可变长参数没有值时就是`nil`切片
- （3）可变长参数的类型必须相同（如果需要是不同类型的可以定义为 interface{}类型）；


## 声明不定长数组

数组是有固定长度的，我们**在声明数组时一定要声明长度，因为数组在编译时就要确认好其长度**；
但是有些时候对于想偷懒的我，就是不想写数组长度，有没有办法让他自己算呢？当然有，使用`...`操作符声明数组时，你只管填充元素值，其他的交给编译器自己去搞就好了；

```go
a := [...]int{1, 3, 5} // 数组长度是3，等同于 a := [3]{1, 3, 5}
```

有时我们想声明一个大数组，但是某些`index`想设置特别的值也可以使用`...`操作符搞定：
```go
a := [...]int{1: 20, 999: 10} 
// 数组长度是1000, 下标1的元素值是20，下标999的元素值是10，其他元素值都是0
```


# 短变量申明`:=`
我们可以使用 **name := expression** 的语法形式来声明和初始化局部变量，相比于使用`var`声明的方式可以减少声明的步骤：

```go
var a int = 10
等用于
a := 10
```

## 注意
### 多变量赋值可能会重新声明

使用 `:=` 一次可以声明多个变量，例如：
```go
i, j := 0, 0
j, k := 1, 1

```

(1) 当 := 左侧存在新的变量时（如 k）,那么已经声明的变量（如 j）会被重新声明。**这并没有引入新的变量，只是把变量的值改变了。**

(2) 当 := 左侧没有新变量编译报错.
如下示例由于左侧没有新变量编译会提示" No new variables on the left side of ':=' "错误。
```go
i,j := 2,3
i,j := 6,8
```

### 不能用于函数外部
- `:=` 这种简短变量声明只能用于函数中，用来初始化全局变量是不行的。
    
- 可以理解成 `:=` 会拆分成两个语句，即声明和赋值。赋值语句不能出现在函数外部的，因为在任何函数外，语句都应该以关键字开头，例如 type、var这样的关键字。


#### 全局变量的申明
**在函数外部声明的变量是全局变量，它们具有包级别的作用域**。
在包级别作用域中，变量的声明通常是显式的，不需要使用短变量声明语法糖。而且在全局变量的声明中，必须指定变量的类型，这是因为编译器需要知道变量的大小和布局信息，以便在编译时为它们分配内存。

因此，如果要在包级别声明变量，需要使用 var 关键字或 const 关键字进行显式声明，不能使用 := 语法糖。

```go
package main
 
import "fmt"
 
// 使用 var 关键字显式声明全局变量
var globalVar = 10
 
func main() {
    // 在函数内部使用 := 语法糖声明局部变量
    localVar := 20
    fmt.Println(globalVar, localVar)
}
```

# 特殊的返回值
在Go语言中，允许您使用return语句从一个函数返回多个值。换句话说，在函数中，单个return语句可以返回多个值。
返回值的类型类似于参数列表中定义的参数的类型。


## 返回多个返回值
如果一个函数要返回多个值，在Java中可以使用定义一个新的类来承载返回值，或者偷个懒使用map来接也是可以的。go支持多个返回值。

### 一个返回值
```go
func func1(a string, b int) int {
   fmt.Println("func1------------")
   fmt.Println("a1 = ", a)
   fmt.Println("b1 = ", b)

   c := 100

   return c
}
```

### 多个返回值
```go
func func2(a string, b int) (int, int) {
	fmt.Println("func2------------")
	fmt.Println("a2 = ", a)
	fmt.Println("b2 = ", b)

	return 12, 33
}
```

### 定义并赋值返回值

```go
func func4(a string, b int) (r1 int, r2 int) {
	fmt.Println("func4------------")
	//r1 r2输入fool3的形参，初始化默认的值是0
	//r1 r2 作用域空间是 func4 整个函数体的{}空间
	fmt.Println("r1 = ", r1)
	fmt.Println("r2 = ", r2)
	r1 = b * 2
	r2 = 2000

	return
}
```

`return`关键字处并没有返回`r1`和`r2`变量。r1和r2是返回值，直接给其定义并赋值。


# init函数

`Go`语言提供了先于`main`函数执行的`init`函数，初始化每个包后会自动执行`init`函数，每个包中可以有多个`init`函数，每个包中的源文件中也可以有多个`init`函数。

## 加载顺序
加载顺序如下：

（1）从当前包开始：
如果当前包包含多个依赖包，则先初始化依赖包，层层递归初始化各个包；
（2）在每一个包中：
按照源文件的字典序从前往后执行；
（3）在每一个源文件中：
优先初始化常量、变量，最后初始化`init`函数，当出现多个`init`函数时，则按照顺序从前往后依次执行。
每一个包完成加载后，递归返回，最后在初始化当前包！

## 注意
`init`函数实现了`sync.Once`，无论包被导入多少次，`init`函数只会被执行一次。

## 应用
使用`init`可以应用在服务注册、中间件初始化、实现单例模式等等，比如我们经常使用的`pprof`工具，他就使用到了`init`函数，在`init`函数里面进行路由注册：
```go
//go/1.15.7/libexec/src/cmd/trace/pprof.go
func init() {
 http.HandleFunc("/io", serveSVGProfile(pprofByGoroutine(computePprofIO)))
 http.HandleFunc("/block", serveSVGProfile(pprofByGoroutine(computePprofBlock)))
 http.HandleFunc("/syscall", serveSVGProfile(pprofByGoroutine(computePprofSyscall)))
 http.HandleFunc("/sched", serveSVGProfile(pprofByGoroutine(computePprofSched)))
 
 http.HandleFunc("/regionio", serveSVGProfile(pprofByRegion(computePprofIO)))
 http.HandleFunc("/regionblock", serveSVGProfile(pprofByRegion(computePprofBlock)))
 http.HandleFunc("/regionsyscall", serveSVGProfile(pprofByRegion(computePprofSyscall)))
 http.HandleFunc("/regionsched", serveSVGProfile(pprofByRegion(computePprofSched)))
}
```

# 忽略标识符`_`

## 忽略导包
### 背景
Go语言在设计师有代码洁癖，在设计上尽可能避免代码滥用，所以`Go`语言的导包必须要使用，如果导包了但是没有使用的话就会产生编译错误。

### 忽略导包的处理
但有些场景我们会遇到只想导包，但是不使用的情况，比如上文提到的`init`函数，我们只想初始化包里的`init`函数，但是不会使用包内的任何方法，这时就可以使用 `_` 操作符号重命名导入一个不使用的包：
```go
import _ "github.com/asong"
```

## 忽略字段

有没有办法可以不处理不要的返回值呢？当然有，还是 `_` 操作符，将不需要的值赋给空标识符：
```go
_, ok := test(a, b int)
```

## struct tag 忽略某个字段

### 背景
大多数业务场景我们都会对`struct`做序列化操作，但有些时候我们想要`json`里面的某些字段不参加序列化，`_`操作符可以帮我们处理，`Go`语言的结构体提供标签功能，在结构体标签中使用 `_` 操作符就可以对不需要序列化的字段做特殊处理。

使用如下：
```go
type Person struct{
  name string `json:"-"`
  age string `json: "age"`
}
```

# 切片循环
在Go中提供了`for range`语法来快速迭代对象。**数组、切片、字符串、map、channel**等等类型都可以使用这种方式进行遍历。

总结起来有以下几种形式：

## 只遍历不关心数据

适用于切片、数组、字符串、map、channel

```go
for range T {}
```

## 遍历获取索引或数据
切片，数组、字符串就是索引，map就是key；
channel就是数据

```go
for key := range T{...}
```

## 遍历获取索引和数据
适用于切片、数组、字符串，map。
第一个参数就是索引，第二个参数就是对应的元素值。
map 第一个参数就是key，第二个参数就是对应的值；

```go
for key, value := range T{...}
```

在实际开发中，我们会大概率会遇到遍历map时，只关心map中的数据，不关心key的情况。这个时候我们就是使用最后一种方式，这个key声明了但是没有用，Go这个时候就会提示一个语法错误key没有使用，那我们只好使用Go的另外一个语法糖`_`忽略标识符(就是一个下划线)忽略key，具体如下：
```go
for _, value := range T{...}
```

## 其他

### 切片为 nil
在Go中如果一个切片是nil的时候，我们对他进行遍历或者append操作的时候，是不会出现报错的，这一点很不错，省的像用Java时遍历对象需要判断他是否为null。

```go
package main
import (
	"fmt"
)
func main() {

	temp := make([]int, 0)
	temp = nil

	for _, val := range temp {
		fmt.Println("val=", val)
	}
	temp = append(temp, 1)
	fmt.Println("val=", temp)
}

测试：
# ./aa
val= [1]
```


# 判断map的key是否存在
Go语言提供语法 `value, ok := m[key]`来判断`map`中的`key`是否存在，如果存在就会返回key所对应的值，不存在就会返回空值：

```go
import "fmt"
 
func main() {
    dict := map[string]int{"asong": 1}
    if value, ok := dict["asong"]; ok {
        fmt.Printf(value)
    } else {
      fmt.Println("key:asong不存在")
    }
}
```

# select控制结构

`Go`语言提供了`select`关键字 来进行IO多路复用。
`select`配合`channel`能够让`Goroutine`同时等待多个`channel`读或者写，在`channel`状态未改变之前，`select`会一直阻塞当前线程或`Goroutine`。

```go
func fibonacci(ch chan int, done chan struct{}) {
    x, y := 0, 1
    for {
        select {
            case ch <- x:
                x, y = y, x+y
            case <-done:
                fmt.Println("over")
                return
        }
    }
}

func main() {
    ch := make(chan int)
    done := make(chan struct{})
    go func() {
        for i := 0; i < 10; i++ {
            fmt.Println(<-ch)
        }
        done <- struct{}{}
    }()
    fibonacci(ch, done)
}
```

`select`与`switch`具有相似的控制结构，与`switch`不同的是，`select`中的`case`中的表达式必须是`channel`的收发操作，当`select`中的两个`case`同时被触发时，会随机执行其中的一个。
为什么是随机执行的呢？
随机的引入就是为了避免饥饿问题的发生；如果我们每次都是按照顺序依次执行的，若两个`case`一直都是满足条件的，那么后面的`case`永远都不会执行。

上面例子中的`select`用法是阻塞式的收发操作，直到有一个`channel`发生状态改变。我们也可以在`select`中使用`default`语句，那么`select`语句在执行时会遇到这两种情况：
- 当存在可以收发的`Channel`时，直接处理该`Channel` 对应的 `case`；
- 当不存在可以收发的`Channel` 时，执行 `default` 中的语句；

# 方法接收者
在 Go 中，对于自定义类型 T，为它定义方法时，其接收者可能是类型 T 本身，也可能是 T 类型的指针 `*T`。

## 指针类型接收者

```go
type Instance struct{}
 
func (ins *Instance) Foo() string {
	 return ""
}


```
在上例中，我们定义了 Instance 的 Foo 方法时，其接收者是一个指针类型（`*Instance`）。

```go
func main() {
 var _ = Instance{}.Foo() 
 // 编译错误：cannot call pointer method on Instance{} .
 // 变量的地址是不可变的（该变量没有地址，不能对其进行寻址操作）
}
```



```go
type Instance struct{}
 
func (ins *Instance) String() string {
    return ""
}
 
func main() {
     var ins Instance
     _ = ins.String() // 编译通过；
     // 编译器会自动获取 ins 的地址并将其转换为指向 Instance 类型的指针, 等价于  (&ins).String()
}
```
如上，即使是我们在实现 Foo 方法时的接收者是指针类型，上面 ins 调用的使用依然没有问题。

Ins 值属于 `Instance` 类型，而非 `*Instance`，却能调用 Foo 方法，这是为什么呢？这其实就是 Go 编译器提供的语法糖！

### 小结

当一个变量可变时（也就是说，该变量是一个具有地址的变量，我们可以对其进行寻址操作），我们对类型 T 的变量直接调用 接收者为`*T`类型的 方法是合法的，因为 Go 编译器隐式地获取了它的地址。

变量可变意味着变量可寻址，因此，上文提到的 `Instance{}.Foo()` 会得到编译错误，就在于 `Instance{}` 值不能寻址。


## 实例类型接收者
```go
type Instance struct{}
 
func (ins Instance) Foo() string {
 return ""
}
 
func main() {
 var _ = Instance{}.Foo() // 编译通过
}
```
此时，如果我们将 Foo 方法的接收者改为 Instance 类型，就没有问题。

## 其他
### 未初始化的变量的内存空间
在 Go 中，即使变量没有被显式初始化，编译器仍会为其分配内存空间，因此变量仍然具有内存地址。
不过，由于变量没有被初始化，它们在分配后仅被赋予其类型的默认零值，而不是初始值。当然，这些默认值也是存储在变量分配的内存空间中的。

例如，下面的代码定义了一个整型变量 `x`，它没有被显式初始化，但是在分配内存时仍然具有一个地址：

```go
var x int
fmt.Printf("%p\n", &x) // 输出变量 x 的内存地址

输出结果类似于：0xc0000120a0，表明变量 x 的内存地址已经被分配了。但是由于变量没有被初始化，x 的值将为整型的默认值 0。　　
```

## 范例

```go
package main
 
type B struct {
    Id int
}
 
func New() B {
    return B{}
}
 
func New2() *B {
    return &B{}
}
 
func (b *B) Hello() {
    return
}
 
func (b B) World() {
    return
}
 
func main() {
    // 方法的接收器为 *T 类型
    New().Hello() // 编译不通过
 
    b1 := New()
    b1.Hello() // 编译通过
 
    b2 := B{}
    b2.Hello() // 编译通过
 
    (B{}).Hello() // 编译不通过
    B{}.Hello()   // 编译不通过
 
    New2().Hello() // 编译通过
 
    b3 := New2()
    b3.Hello() // 编译通过
 
    b4 := &B{} // 编译通过
    b4.Hello() // 编译通过
 
    (&B{}).Hello() // 编译通过
 
    // 方法的接收器为 T 类型
    New().World() // 编译通过
 
    b5 := New()
    b5.World() // 编译通过
 
    b6 := B{}
    b6.World() // 编译通过
 
    (B{}).World() // 编译通过
    B{}.World()   // 编译通过
 
    New2().World() // 编译通过
 
    b7 := New2()
    b7.World() // 编译通过
 
    b8 := &B{} // 编译通过
    b8.World() // 编译通过
 
    (&B{}).World() // 编译通过
}
```

结果如下所示：
```text
./main.go:25:10: cannot call pointer method on New()
./main.go:25:10: cannot take the address of New()
./main.go:33:10: cannot call pointer method on B literal
./main.go:33:10: cannot take the address of B literal
./main.go:34:8: cannot call pointer method on B literal
./main.go:34:8: cannot take the address of B literal
```

## 小结
假设 `T` 类型的方法上接收器既有 `T` 类型的，又有 `*T` 指针类型的，那么就不可以在==不能寻址==的 `T` 值上调用 `*T` 接收器的方法。

- `&B{}` 是指针，可寻址
- `B{}` 是值，不可寻址
- `b := B{}` b是变量，可寻址

在 Golang 中，当一个变量是可变的（也就是说，该变量是一个具有地址的变量，我们可以对其进行寻址操作），我们可以通过对该变量的指针进行方法调用来执行对该变量的操作，否则就会导致编译错误。

# 参考
```bash

```