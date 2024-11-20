```table-of-contents
```
# 背景
Go 语言和典型的面向对象的语言不太一样，Go 在语法上是不支持面向对象的类、继承等相关概念的。但是，并不代表 Go 里面不能实现面向对象的一些行为比如继承、多态，在 Go 里面，通过 interface 完全可以实现诸如 C++ 里面的继承 和 多态的语法效果。

# 什么是接口
interface 就是字面意思——接口，C++中可以用虚基类表示；Java 中就是 interface。
接口 interface 是Go语言中一种抽象类型，它定义了一组方法(`interface`中没有成员变量)，但没有具体实现。这些方法可以被**任一**类型通过方法实现。因此接口就是对象行为的申明（不是定义，仅仅表示方法签名，也可以称作函数原型）。
通过接口，我们可以定义对象的行为和功能，而无需关心具体对象的类型。

方法签名由方法名，输入参数，返回值三部分组成。
> 注意：我多次强调 任一类型 ，Golang 中所有类型都可以实现自己的方法，为了便于理解，本文还是使用 struct（结构体）来做示例。

# 接口的声明与实现
## 接口类型的声明/定义

Go语言中，可以通过`type`关键字和`interface{}`类型来定义接口。接口的定义通常包含一系列方法签名，即方法的名称、参数列表和返回值。

接口可以定义一组方法，但不需要实现，并且接口中不能包含任何变量，到某个自定义类型要使用的时候，再根据具体情况把这些方法具体实现出来;

接口的定义格式如下：
```go
  type 接口类型名 interface{
	  方法名1( 参数列表1 ) 返回值列表1
	  方法名2( 参数列表2 ) 返回值列表2
	  …
  }

  参数说明:
    - 接口类型名：
      Go语言的接口在命名时，一般会在单词后面添加er，如有写操作的接口叫Writer，有关闭操作的接口叫closer等。
      接口名最好要能突出该接口的类型含义。

    - 方法名：
      当方法名首字母是大写且这个接口类型名首字母也是大写时，这个方法可以被接口所在的包（package）之外的代码访问。

    - 参数列表、返回值列表：
      参数列表和返回值列表中的参数变量名可以省略。

```




以下是一个示例代码，定义了一个简单的接口`Shape`：
```golang
type Shape interface {
    Area() float64
    Perimeter() float64
}


说明：
	在上述代码中，我们定义了一个名为`Shape`的接口，它包含了两个方法签名：`Area()`和`Perimeter()`。这两个方法都没有具体实现，只是定义了方法的名称和返回值。


```

```text
interface {
    Area() float64
    Perimeter() float64
}
就是定义了一个接口类型，空接口类型的定义是 interface {}，在函数参数中经常可以看到。
type Shape xxx 只不过是将定义的这个接口类型重命名为 Shape。

```

### 范例

![](attachments/Pasted%20image%2020231122181125.png)

从上面的例子中，我们可以发现，接口类型变量值和类型都是 nil 。这是因为，我们这里声明的是 **Shape** 变量，它还没有指定动态类型，更没有指定任何动态值。


接口包含两个类型：
1》静态类型
静态类型就是指接口本身的类型，比如上图中的 **Shape** 就是静态类型。

2》动态类型（具体类型）
接口的动态类型就是传递给这个接口类型变量的形参或者右值的类型。
有时候 接口 的动态类型也叫做具体类型，因为我们获取接口类型的时候，它返回的是隐藏的动态值的类型。

## 接口的实现

要实现一个接口，需要在具体类型上定义接口中定义的**所有方法**。只有所有方法都被实现，才能说该类型实现了相应的接口。

### 实现接口的条件
怎么实现接口？
- 接口的实现者必须是一个具体类型
- 类型定义的方法和接口里方法名、参数、返回值都必须一致
- 若接口有多个方法，那么要实现接口中的所有方法

> 注意：对于空接口 interface{} ，因为它没有定义任何的函数（方法），所以说Go中的所有类型都实现了空接口。



以下是一个示例代码，展示了如何在结构体类型上实现接口：
```golang
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    width  float64
    height float64
}

func (r Rectangle) Area() float64 {
    return r.width * r.height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.width + r.height)
}

在上述代码中，我们定义了一个名为`Rectangle`的结构体，并为它实现了`Shape`接口中定义的两个方法：`Area()`和`Perimeter()`。
```

### 范例

![](attachments/Pasted%20image%2020231122180739.png)

在上面的程序中，我们定义了一个 Shape接口和 Rect结构体。然后 Rect 实现了 Area 和Perimeter 方法，这就实现了 Shape接口的所有方法签名，所以我们就说 Rect 实现了 Shape 接口（这是 Golang 默认的，自动实现），我们并没有明确的显式指明 Rect实现了 Shape 接口。

我们可以看到，接口变量 s 的动态类型就是 **Rect**，动态值就是 **Rect** 结构体对象 {5,4}。
> 我们用**动态**这个词，是因为我们也可以给接口变量 s 赋值另一个实现了 Shape 接口的结构体类型， 所以 s 实际指向的对象类型不是固定的，是动态的。

当一个类型实现了某个接口（实现了这个接口的所有函数），这个类型的变量也可以用它所实现的接口类型表示（或者说用接口类型的变量去存放）。
> 或者说：interface 变量存储的是实现者的值。

![](attachments/Pasted%20image%2020231122182048.png)

![](attachments/Pasted%20image%2020231122182241.png)
上面程序中，我们删除了 **Perimeter** 方法，这个程序就编译不过并且抛出一个错误。
```go
program.go:22: cannot use Rect literal (type Rect) as type Shape in assignment:
        Rect does not implement Shape (missing Perimeter method)
```
从上面的错误中我们可以很容易的理解实现接口的要求：我们需要实现接口中申明的所有方法签名。

## 接口的使用

### 接口的赋值

==由于接口类型的值可以是任意一个实现了该接口的类型值。所以接口值除了需要记录具体值之外，还需要记录这个值属于的类型。
也就是说接口值由"类型"和"值"组成，鉴于这两部分会根据存入值的不同而发生变化，我们称之为接口的动态类型和动态值。==

接口未初始化时，其值为nil，调用其方法会引发panic。接口值可以被认为是包含两个部分的元组（tuple），即具体类型和该类型的值。

当我们声明了一个接口值却没有进行初始化（也就是没有赋值），那么这个接口的类型和值都是nil。在Go中，调用值为nil的函数是非法的，所以如果尝试在这样的接口值上调用方法，那么程序将触发运行时错误。
```go
type MyInterface interface {
    MyMethod()
}

func main() {
    var mi MyInterface // 声明一个接口，但未进行初始化
    mi.MyMethod()     // 运行时错误：在nil的接口值上调用方法
}
```

如果接口的值为nil，那么我们可以通过类型断言来判断接口的动态类型是否为nil，从而避免panic。
```go
type MyInterface interface {
    MyMethod()
}

func main() {
    var mi MyInterface // 声明一个接口，但未进行初始化
    if mi != nil {
        mi.MyMethod()
    }
}
```

```go
type MyInterface interface {
    MyMethod()
}

type MyType struct{}

func (mt *MyType) MyMethod() {
    if mt == nil {
        fmt.Println("nil receiver value")
    }
}

func main() {
    var mt *MyType    //声明并初始化为nil值
    var mi MyInterface = mt //将nil值赋给接口
    mi.MyMethod()     // "nil receiver value"
}
```

### 空接口类型的使用

对于空接口 interface{} ，因为它没有定义任何的函数（方法），所以说Go中的所有类型都实现了空接口。

#### 空接口切片数组

```go
package main

import (
    `fmt`
)

func main() {
    var a = 10
    // testInterface(a)

    var b = "hello world"
    // testInterface(b)

    var c = 3.1415926
    
    // 构造一个万能类型的数组list，什么都有存
    var list []interface{}
    list = append(list, a, b, c)

    findType(list)
}

func testInterface(a interface{})  {
    b, ok := a.(int)
    if ok {
        fmt.Printf("a is int: %d, type: %T\n", b, b) // a is int: 10, type: int
    }

    c, ok := a.(string)
    if ok {
        fmt.Printf("a is string: %s, type: %T\n", c, c)  // a is string: hello world, type: string
    }
}

func findType(a interface{})  {
    list, ok := a.([]interface{})
    if ok {
        for _, value := range list {
            switch v := value.(type) {
            case int:
                fmt.Printf("value: %d, type: %T\n", v, v)
            case string:
                fmt.Printf("value: %s, type: %T\n", v, v)
            case float64:
                fmt.Printf("value: %f, type: %T\n", v, v)
            case []interface{}:
                fmt.Println("type: []interface{}")
            default:
                fmt.Println("unknown type.")
            }
        }
    } else {
        fmt.Println("a is not []interface{} type.")
    }

}
```



### 非空接口类型的使用
在使用接口时，我们可以将实现了接口的对象，赋值给接口类型的变量。通过这种方式，我们可以隐藏具体对象的类型，只使用接口来调用方法。

以下是一个示例代码，展示了如何使用接口：

```golang

type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    width  float64
    height float64
}

func (r Rectangle) Area() float64 {
    return r.width * r.height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.width + r.height)
}

func main() {
    var s Shape
    s = Rectangle{width: 10, height: 5}

    fmt.Println("Area:", s.Area())
    fmt.Println("Perimeter:", s.Perimeter())
}

在上述代码中，我们创建了一个接口类型的变量`s`，并将一个具体对象`Rectangle`赋值给它。然后，我们可以使用接口变量`s`来调用`Area()`和`Perimeter()`方法，而无需关心具体对象的类型。
```

### 使用小结

- 1.接口本身不能创建实例，但是可以指向一个实现了该接口的自定义类型的变量;
    
- 2.只要是自定义数据类型，就可以实现接口，不仅仅是结构体类型，内置的数据类型起别名后依旧是可以让其实现接口的;
    
- 3.一个自定义类型可以实现多个接口;

- 4.一个接口(比如A接口)可以继承多个别的接口(比如B和C接口)，这时如果想要实现A接口，也必须将B和C接口的所有方法全部实现;
    
- 5.interface类型默认是一个指针(引用类型)，如果没有对interface初始化就使用，那么会输出nil;
    
- 6.空接口是指没有定义任何方法的接口类型，所以可以理解所有类型都实现了空接口，也可以理解为我们可以把任何一个变量赋值给空接口;
    
- 7.使用值接收者实现接口之后，不管是结构体类型还是对应的结构体指针类型的变量都可以赋值给该接口变量;
    
- 8.使用指针接收者实现接口之后，只有对应的结构体指针类型的变量都可以赋值给该接口变量;
    
- 9.一个接口的所有方法，不一定需要由一个类型完全实现，接口的方法可以通过在类型中嵌入其他类型或者结构体来实现;
    
- 10.由于接口类型的值可以是任意一个实现了该接口的类型值，所以接口值除了需要记录具体值之外，还需要记录这个值属于的类型;

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

# 类型断言type assertion
接口（interface）和类型断言（type assertion）是Go语言中重要的特性之一。
类型断言（type assertion）是用于断言某个变量是某种类型。类型断言只能在接口上进行。
类型断言是一种在Go语言中将接口类型转换为具体类型的操作。
通过类型断言，我们可以在运行时判断接口变量的底层类型，并将其转换为指定的类型。类型断言的存在使得我们可以在需要时以正确的类型使用接口变量。


## 语法

看起来像x.(T)被称为断言类型，这里x表示一个接口类型的变量，T表示一个具体的类型。
这个语法表示我们要判断`x`的实际类型是否是`T`，如果是，则将`x`转换为`T`类型的值，并返回给我们使用。
断言的过程是通过判断`x`的实际类型信息和`T`类型信息是否一致来完成的。如果一致，断言就成功了，我们可以继续使用`T`类型的值；

在Go语言中，可以使用以下语法进行类型断言：
```text
//非安全类型断言
<目标类型的值> := <表达式>.( 目标类型 ) 

// 安全类型断言 
<目标类型的值>，<布尔参数> := <表达式>.( 目标类型 )

// 基本格式
x.(T)
v := x.(T)
v, ok := x.(T)

```
**类型断言的必要条件是x是接口类型,非接口类型的x不能做类型断言。**


在进行类型断言时，我们应该知道接口变量的底层类型，但并不总是如此。这就是为什么类型断言表达式实际上会返回第二个可选值的原因：
```golang
var greeting interface{} = "42" 
greetingStr, ok := greeting.(string)
```
第二个值 `ok` 是一个布尔值，如果我们的断言正确，则为 `true`；否则为 `false`。
因此，**类型断言是在运行时执行**的。

## 分类


### 断言的类型T是一个具体类型
如果断言的类型T是一个具体类型，然后类型断言检查x的动态类型是否和T相同。
如果这个检查成功了，类型断言的结果是x的动态值，当然它的类型是T。
换句话说，**具体类型的类型断言**从它的操作对象中获得具体的值。

### 断言的类型T是一个接口类型

通过接口类型断言(`ins.(Type)`)，**如果Type是一个接口类型，就可以判断接口实例ins中所保存的类型是否也实现了Type接口**。

```go
var r io.Read
tty, err := os.OpenFile("/dev/tty", os.O_RDWR, 0)
if err != nil {
    return nil, err
}
r = tty

var w io.Writer
w = r.(io.Writer)

说明：
上面的r是io.Read接口的一个实例变量，它里面保存的是tty和它的类型，即`(tty, *os.File)`，
然后断言r的类型，探测它里面的类型`*File`是否也实现了io.Writer接口，
如果实现了，则保存到io.Writer接口的实例变量w中，这样w实例也将保存`(tty,*os.File)`。
```

#### 范例
```go
package main

import (
  "fmt"
)

// 定义两个接口
type Reader interface {
  Read() string
}

type Writer interface {
  Write(string)
}

// 定义一个结构体，实现了 Reader 和 Writer 接口
type File struct {
  content string
}

func (f *File) Read() string {
  return f.content
}

func (f *File) Write(content string) {
  f.content = content
}

func main() {
  // 创建一个 File 类型的实例
  var f File = File{"hello, file"}

  // 将 File 类型的实例赋值给 Reader 接口类型的变量
  var r Reader = &f

  // 类型断言：将 Reader 接口类型的变量断言为 Writer 接口类型
  if w, ok := r.(Writer); ok {
    // 断言成功，w 是 Writer 接口类型
    w.Write("Hello, World!")
    fmt.Println(r.Read()) // 输出: Hello, World!
  } else {
    // 断言失败
    fmt.Println("类型断言失败")
  }
}
```

测试结果：
```
go build -o test2 test2.go

$ ./test2
Hello, World!
```

## 用法
### 单个类型的类型断言
```golang
y, ok := x.(T)
x: 为 接口类型的变量x。
T：为实现了接口类型的动态类型
y：为转换后的类型为T的变量
ok：转换是否成功。
```

### 多个类型的 if/else 类型断言
```golang

if value1, ok := r.(Type1); ok {
    // 处理 Type1
} else if value2, ok := r.(Type2); ok {
    // 处理 Type2
}

```

### 多个类型的 switch 类型断言

格式：

```golang

switch v := interfaceVariable.(type) {
case Type1:
    // 处理 Type1
case Type2:
    // 处理 Type2
default:
    // 默认处理
}

```

范例如下所示：
```bash
var greeting interface{} = 42

switch g := greeting.(type) {
  case string:
    fmt.Println("g is a string with length", len(g))
  case int:
    fmt.Println("g is an integer, whose value is", g)
  default:
    fmt.Println("I don't know what g is")
}
```

## 范例
```golang
type Shape interface {
    Area() float64
    Perimeter() float64
}

type Rectangle struct {
    width  float64
    height float64
}

func (r Rectangle) Area() float64 {
    return r.width * r.height
}

func (r Rectangle) Perimeter() float64 {
    return 2 * (r.width + r.height)
}

// ....  Circle 也实现了接口类型。

func printArea(s Shape) {
    if rect, ok := s.(Rectangle); ok {
        fmt.Println("Rectangle Area:", rect.Area())
    } else if circle, ok := s.(Circle); ok {
        fmt.Println("Circle Area:", circle.Area())
    } else {
        fmt.Println("Unknown Shape")
    }
}

func main() {
    var s Shape
    s = Rectangle{width: 10, height: 5}
    printArea(s)

    s = Circle{radius: 5}
    printArea(s)
}

在上述代码中，我们定义了一个名为`printArea`的函数，它接收一个`Shape`类型的参数，
并对不同类型的形状计算并打印面积。
在函数内部，我们使用类型断言将接口变量`s`转换为具体类型`Rectangle`和`Circle`，然后调用它们的`Area()`方法。

```

## 类型断言和类型转换
这是一个 Go 中的类型断言：
```golang
var greeting interface{} = "hello world"
greetingStr := greeting.(string)

```

这是一个类型转换：
```golang
greeting := []byte("hello world")
greetingStr := string(greeting)
```

最明显的区别是它们具有不同的语法（`variable.(type)` vs `type(variable)`）。

### 背景

Go中的每种类型都定义了两个东西：

1. 变量如何存储（底层数据结构）
2. 可以对变量执行什么操作（可以用于其中的方法和函数）

### 范例

可以从基本类型或通过创建复合类型来声明新类型：
```go
// myInt 是一个新类型，其基础类型是 `int`
type myInt int

// AddOne 方法适用于 `myInt` 类型，但不适用于常规的 `int`
func (i myInt) AddOne() myInt { return i + 1}

func main() {
    var i myInt = 4
    fmt.Println(i.AddOne())
}
```

当我们声明 `myInt` 类型时，我们基于基本的 `int` 类型定义了变量数据结构，但是更改了我们可以使用 `myInt` 类型变量的方式（通过在其上声明一个新方法）。

由于 `int` 和 `myInt` 的底层数据结构相似，因此这些类型的变量可以相互转换：
```go
var i myInt = 4 
originalInt := int(i)
```

### 类型转换

只要底层数据结构相同，类型就可以互相转换。

#### 何时可以使用类型转换？

只有在底层数据结构相同的情况下，类型才可以相互转换。
看一个使用结构体的示例：
```go
type person struct {
    name string
    age int
}

type child struct {
    name string
    age int
}

type pet {
  name string
}

func main() {
    bob := person{
        name: "bob",
        age: 15,
        }
    babyBob := child(bob)
    // "babyBob := pet(bob)" bob转换为pet类型会导致编译错误
    fmt.Println(bob, babyBob)
}
```

分析：
```go
在这里，`person` 和 `child` 具有相同的数据结构，即：
struct {
    name string
    age int
}

因此它们可以相互转换。
具有不同底层数据结构的类型无法相互转换。

优化：
可以使用别名来声明具有相同数据结构的多个类型：
type child person

这意味着 `child` 基于与 `person` 相同的数据结构（类似于我们之前的整数示例）。
```


## 类型断言和接口反射
### 背景
如果某个函数的入参是interface{}，有下面几种方式可以获取入参对应的形参的类型：


#### 反射
```golang
import (
    "reflect"
    "fmt"
)
func main() {
    v := "hello world"
    fmt.Println(typeof(v))
}
func typeof(v interface{}) string {
    return reflect.TypeOf(v).String()
}
```

#### fmt 包
```golang
import "fmt"
func main() {
    v := "hello world"
    fmt.Println(typeof(v))
}
func typeof(v interface{}) string {
    return fmt.Sprintf("%T", v)
}
```

fmt.Printf(“%T”)里最终调用的还是`reflect.TypeOf()`。

```golang
func (p *pp) printArg(arg interface{}, verb rune) {
    ...
    // Special processing considerations.
    // %T (the value's type) and %p (its address) are special; we always do them first.
    switch verb {
    case 'T':
        p.fmt.fmt_s(reflect.TypeOf(arg).String())
        return
    case 'p':
        p.fmtPointer(reflect.ValueOf(arg), 'p')
        return
    }
    ...
}
```

#### 类型断言
```golang
func main() {
    v := "hello world"
    fmt.Println(typeof(v))
}
func typeof(v interface{}) string {
    switch t := v.(type) {
    case int:
        return "int"
    case float64:
        return "float64"
    //... etc
    default:
        _ = t
        return "unknown"
    }
}
```

### 谁的速度更快


## 接口和类型断言的关系

当一个函数的形参是 interface{} 时，意味着这个参数被自动的转为interface{} 类型，在函数中，如果想得到参数的真实类型，就需要对形参进行断言。

- 类型断言就是将接口类型的值x，转换成类型T，格式为：x.(T)
- 类型断言x必须为接口类型
- T可以是非接口类型，若想断言合法，则T必须实现x的接口。


### 接口的多态性
接口的多态性使得我们可以使用一个接口类型的变量来引用实现了该接口的不同的具体类型的对象。这种灵活性提供了很大的便利。

### 类型断言的安全性
类型断言是一种将接口类型转换为具体类型的操作，但在进行类型断言时，需要注意类型的匹配性。
对于 x.(T) 就是 接口类型的变量x 和 具体的类型 T 是否对应，也就是传递给接口类型变量x的实参的类型 是否为类型 T。



## 类型断言的性能

参考：[# [interface的类型断言是如何实现](https://segmentfault.com/a/1190000039894161)](https://segmentfault.com/a/1190000039894161)


## 注意事项
### 接口实例中存放的是什么类型，才能转换成什么类型

接口实例中可以存放各种实现了接口的类型实例，在有需要的时候，还可以==通过`ins.(Type)`或`ins.(*Type)`==的方式将接口实例ins直接转回Type类型的实例。

```go
var i int = 30
var ins interface{}

// 接口实例ins中保存的是int类型
ins = i
x := ins.(int)  // 接口转回int类型的实例i
println(x) //输出30
```


```go
var i int = 30
var ins interface{}

ins = i
x := ins.(int)
println("i addr: ",&i,"x addr: ",&x)

输出：
0xc042049f68 
0xc042049f60
```

注意，接口实例转回时，**接口实例中存放的是什么类型，才能转换成什么类型**。
==同类型的值类型实例和指针类型实例不能互转==，不同类型更不能互转。


#### `x.(Type)` 和  `x.(*Type)`的区别

```go
package main

import "fmt"

// Shaper 接口类型
type Shaper interface {
  Area() float64
  Add()
}

// Square struct类型
type Square struct {
  length float64
}

func (s Square) Area() float64 {
  return s.length * s.length
}

func (s Square) Add() {
  s.length++
}



type Circle struct {
    length float64
    tag  string
}

func (c *Circle) Area() float64 {
  return c.length * c.length * 3.14
}

func (c *Circle) Add() {
  c.length++
}



func main() {
  var ins1, ins2 Shaper

  // 指针类型的实例
  s1 := new(Square)
  s1.length = 3.0
  ins1 = s1
  ins1.Add()
  if v, ok := ins1.(*Square); ok {
    fmt.Printf("ins1: %T, %#v \n", v, v)
    fmt.Printf(" s1 = %p, v = %p, s1 info = %#v, v info = %#v \n", s1, v, s1, v)
  }

  // 值类型的实例
  s2 := Square{4.0}
  ins2 = s2
  ins1.Add()
  if v, ok := ins2.(Square); ok {
    fmt.Printf("ins2: %T, %#v\n", v, v)
    fmt.Printf("&s2 = %p, &v=%p, s2=%#v, v=%#v \n", &s2, &v, s2, v)
  }

  var ins3 Shaper
  c1 := new(Circle)
  c1.length = 5.0
  ins3 = c1
  ins3.Add()
  if v, ok := ins3.(*Circle); ok {
    fmt.Printf("ins3: %T, %#v \n", v, v)
    fmt.Printf(" c1 = %p, v = %p, c1 info = %#v, v info = %#v \n", c1, v, c1, v)
  }

}

```

```text
输出结果：

ins1: *main.Square, &main.Square{length:3}
 s1 = 0xc000138000, v = 0xc000138000, s1 info = &main.Square{length:3}, v info = &main.Square{length:3}
ins2: main.Square, main.Square{length:4}
&s2 = 0xc000138040, &v=0xc000138048, s2=main.Square{length:4}, v=main.Square{length:4}
ins3: *main.Circle, &main.Circle{length:6, tag:""}
 c1 = 0xc00012c018, v = 0xc00012c018, c1 info = &main.Circle{length:6, tag:""}, v info = &main.Circle{length:6, tag:""}
```

上面两个Printf都会输出，因为它们的类型判断都返回true。如果将`ins2.(Square)`改为`ins2.(*Square)`，第二个Printf将不会输出，因为ins2它保存的是值类型的实例。


### 接口类型变量（接口实例）的定义
**ins必须明确是接口实例**。例如，以下前两种声明是有效的，第三种推断类型是错误的，因为它可能是接口实例，也可能是类型的实例副本。

```go
var ins Shaper     // 正确
ins := Shaper(s1)  // 正确
ins := s1          // 错误
```

当ins不能确定是接口实例时，用它来进行测试，例如`ins.(Square)`将会报错：
```go
invalid type assertion:ins.(Square) (non-interface type (type of ins) on left)
```



# interface的 底层原理





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

# interface 实现者的方法 receiver 
## 范例
![](attachments/Pasted%20image%2020231122184415.png)

在上面的程序中 Area方法属于 `*Rect` 类型，因此 Area 的接收者会去获取是 `Rect` 类型的指针（即使使用 Rect 类型的值去调用，底层也会转换成 `*Rect` 类型去调用）。但是，上诉程序将会编译不通过，go 编译器会报编译错误。
```go
program.go:27: cannot use Rect literal (type Rect) as type Shape in assignment: Rect does not implement Shape (Area method has pointer receiver)
```

在 Golang 中 `Rect` 类型 和 `*Rect` 类型是两种不同的类型，在使用 Rect 类型变量调用方法的时候，Golang 底层会去自动转换成指定的接收者类型去调用该方法。但是，在接口的实现中，`Rect` 实现的方法，不代表 `*Rect` 就实现了该方法。就像上面的程序，`*Rect` 实现了 Area 方法，但是 `Rect` 没有实现 Area 方法，Golang 底层不会默认 Rect 也实现了 Area方法。

正确的写法如下所示：
![](attachments/Pasted%20image%2020231122185001.png)

## 指针接收者和值接收者
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
（1）如果是按 pointer 调用，go 会自动进行转换，因为有了指针总是能得到指针指向的值是什么。如果是 value 调用，go 将无从得知 value 的原始值是什么，因为 value 是份拷贝。
**go 会把指针进行隐式转换得到 value，但反过来则不行**。

（2）对于 receiver 是 value 的 method，任何在 method 内部对 value 做出的改变都不影响调用者看到的 value，这就是按值传递。

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


### 值接受者实现接口

```go
package main

import (
    "fmt"
)

// KongTiao  定义的一个空调接口类型
type KongTiao interface {
    ZhiLeng()  // 制冷
    ZhiRe()     // 制热
}

// GREE 格力结构体类型
type GREE struct {
    Name        string
    Price       float64
    Temperature float64
}

// Haier 海尔结构体类型
type Haier struct {
    Name        string
    Price       float64
    Temperature float64
}

// ZhiLeng 使用值接收者定义ZhiLeng方法实现KongTiao接口
func (g GREE) ZhiLeng() {
    fmt.Printf("价格为[%.2f]的[%s]开始制冷，温度控制在%.2f℃\n", g.Price, g.Name, g.Temperature)
}

// ZhiRe 使用值接收者定义ZhiRe方法实现KongTiao接口
func (g GREE) ZhiRe() {
    fmt.Printf("价格为[%.2f]的[%s]开始制热，温度控制在%.2f℃\n", g.Price, g.Name, g.Temperature)
}

func (h Haier) ZhiLeng() {
    fmt.Printf("价格为[%.2f]的[%s]开始制冷，温度控制在%.2f℃\n", h.Price, h.Name, h.Temperature)
}

func (h Haier) ZhiRe() {
    fmt.Printf("价格为[%.2f]的[%s]开始制热，温度控制在%.2f℃\n", h.Price, h.Name, h.Temperature)
}

func main() {

    // 声明一个KongTiao接口类型的变量kt
    var kt KongTiao

    // geli是GREE类型
    var geli = GREE{Name: "格力", Price: 2699.00, Temperature: 16}

    // 可以将geli赋值给变量x
    kt = geli
    kt.ZhiLeng()

    // haier是Haier指针类型
    var haier = &Haier{Name: "海尔", Price: 2199.00, Temperature: 28.5}

    // 也可以将haier赋值给变量kt
    kt = haier
    kt.ZhiRe()
}
```

使用值接收者实现接口之后，不管是结构体类型还是对应的结构体指针类型的变量都可以赋值给该接口变量。


### 指针接受者实现接口
```go
package main

import (
    "fmt"
)

// KongTiao 定义的一个空调接口类型
type KongTiao interface {
    ZhiLeng() //制冷
    ZhiRe()  // 制热
}

// GREE 格力结构体类型
type GREE struct {
    Name        string
    Price       float64
    Temperature float64
}

// Haier 海尔结构体类型
type Haier struct {
    Name        string
    Price       float64
    Temperature float64
}

// ZhiLeng 使用指针接收者定义ZhiLeng方法实现KongTiao接口
func (g *GREE) ZhiLeng() {
    fmt.Printf("价格为[%.2f]的[%s]开始制冷，温度控制在%.2f℃\n", g.Price, g.Name, g.Temperature)
}

// ZhiRe 使用指针接收者定义ZhiRe方法实现KongTiao接口
func (g *GREE) ZhiRe() {
    fmt.Printf("价格为[%.2f]的[%s]开始制热，温度控制在%.2f℃\n", g.Price, g.Name, g.Temperature)
}

func (h *Haier) ZhiLeng() {
    fmt.Printf("价格为[%.2f]的[%s]开始制冷，温度控制在%.2f℃\n", h.Price, h.Name, h.Temperature)
}

func (h *Haier) ZhiRe() {
    fmt.Printf("价格为[%.2f]的[%s]开始制热，温度控制在%.2f℃\n", h.Price, h.Name, h.Temperature)
}

func main() {
    // 结论: 使用指针接收者实现接口之后，只有对应的结构体指针类型的变量都可以赋值给该接口变量。

    // 声明一个KongTiao接口类型的变量kt
    var kt KongTiao

    // geli是GREE指针类型
    var geli = &GREE{Name: "格力", Price: 2699.00, Temperature: 16}

    // 可以将geli赋值给变量x
    kt = geli
    kt.ZhiLeng()

    // haier是Haier类型
    // var haier = Haier{Name: "海尔", Price: 2199.00, Temperature: 28.5}

    // 下面的代码无法通过编译，报错: "Haier does not implement KongTiao (method ZhiLeng has pointer receiver)"
    // kt = haier // haier是Haier类型，并不是指针，不能将harier当成KongTiao类型
    // kt.ZhiRe()

}

```

用指针接收者实现接口之后，只有对应的结构体指针类型的变量都可以赋值给该接口变量。

## 小结
### 指针接收者可以更改调用者
对于 receiver 是 value 的 method（即 接口方法的实现者是value ），任何在 method 内部对 value 做出的改变都不影响调用者看到的 value，这就是按值传递。

如果receiver 是 pointer 的method（即 接口方法的实现者是pointer ），在实现的方法内部可以更改调用者的成员变量的值。

### 值接收者可以接受调用者为值或者指针
（1）对于 receiver 是 value 的 method，调用者调用方法时可以传递值，也可以传递指针。

如果是按 pointer 调用，go 会自动进行转换，因为有了指针总是能得到指针指向的值是什么。**go 会把指针进行隐式转换得到 value，但反过来则不行**。

（2）对于 receiver 是 pointer 的 method，调用者调用方法时只能够传递指针。

如果是 value 调用，go 将无从得知 value 的原始值是什么，进而无法得到其地址，因为 value 是份拷贝。


# 空 interface
## 定义
`interface{}` 是一个**空的 interface 类型，即 {}中没有任何方法的定义。注意是一个类型，不是变量**。
一个 interface{} 类型的变量包含了2个指针，一个指针指向值的类型，另外一个指针指向实际的值。

一个类型如果实现了一个 interface 的所有方法就说该类型实现了这个 interface，空的 interface 没有方法，所以**可以认为所有的类型都实现了空接口类型 `interface{}`**。
如果定义一个函数参数是 `interface{}` 类型，这个函数应该可以接受任何类型作为它的参数。

## 原理
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

## 使用
```go
空接口的定义方式：
	type empty_int interface{}

声明一个空接口实例
	var i interface{}

函数使用空接口类型参数：
	func myfunc(i interface{})


```

在Go中很多地方都使用空接口类型的参数，用的最多的`fmt`中的Print类方法：
```go
$ go doc fmt Println
package fmt // import "fmt"

func Println(a ...interface{}) (n int, err error)
    Println formats using the default formats for its operands and writes to
    standard output. Spaces are always added between operands and a newline is
    appended. It returns the number of bytes written and any write error
    encountered.
```

### 成员为空接口类型的数据结构
可以定义一个空接口类型的array、slice、map、struct等，这样它们就可以用来存放任意类型的对象，因为任意类型都实现了空接口。

#### 空接口的slice

```go
package main

import "fmt"

func main() {
    any := make([]interface{}, 5)
    any[0] = 11
    any[1] = "hello world"
    any[2] = []int{11, 22, 33, 44}
    for _, value := range any {
        fmt.Println(value)
    }
}

结果：
11
hello world
[11 22 33 44]
<nil>
<nil>
```

#### 含有空接口类型成员的struct
比如，某个struct中，如果有一个字段想存储任意类型的数据，就可以将这个字段的类型设置为空接口：

```go
type my_struct struct {
    anything interface{}
    anythings []interface{}
}
```

### 拷贝数据结构到空接口数据结构
前面解释了任意类型的对象都能赋值给空接口实例。如下所示：
```go
var any interface{}
any = "hello world"
any = 11
```

空接口是一种接口，它是一种指针类型的数据类型，虽然不严谨，但它确实保存了两个指针，一个是对象的类型(或iTable)，一个是对象的值。所以上面的赋值过程是让空接口any保存各个数据对象的类型和对象的值。
换一种角度考虑，空接口有自己的内存布局方式：两个指针，占用两个机器字长。


#### 范例

将某个slice中的数据拷贝到空接口slice中将报错。
```go
package main

import "fmt"

func main() {
    testSlice := []int{11,22,33,44}

    // 成功拷贝
    var newSlice []int
    newSlice = testSlice
    fmt.Println(newSlice)

    // 拷贝失败
    var any []interface{}
    any = testSlice
    fmt.Println(any)
}

```

分析：
```text
这是因为每个空接口的内存布局都占用两个机器字长的内容。对于长度为N的空接口slice来说，它的每个元素都是以2机器字长为单元的连续空间，共占用`N*2`个机器字长的空间。

而普通的slice，例如上面的testSlice，它的每个元素是int类型的，int类型的内存布局和空接口不一样。

这些对象的内存布局在编译期间就已经确定好了，所以没法直接将不同内存布局的数据结构进行拷贝。
```

要想完成期待的拷贝，可以使用for-range的方式，将testSlice中的每个元素赋值给空接口slice的空接口元素：也就是一个个的空接口实例。

```go
var any []interface{}
for _,value := range testSlice{
    any = append(any,value)
}
```

这样，==空接口Slice中的每个空接口实例都指向更底层的各个数据对象。而不是像前面错误的拷贝方式：每个空接口元素想要当作这些数据对象。不仅空接口的Slice如此，其它包含空接口的数据结构，也都类似==。


## 注意

### interface {} 空接口是一个数据类型

空接口是一种接口，它是一种指针类型的数据类型，虽然不严谨，但它确实保存了两个指针，一个是对象的类型(或iTable)，一个是对象的值。所以上面的赋值过程是让空接口any保存各个数据对象的类型和对象的值。

换一种角度考虑，空接口有自己的内存布局方式：两个指针，占用两个机器字长。





## QA
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

## eface和iface对比
**无函数的`eface`**
![](attachments/Pasted%20image%2020231123105124.png)

**有函数的`iface`**
![](attachments/Pasted%20image%2020231123105133.png)

# 多接口
一个类型可以实现多个接口，也可以理解为多继承。
![](attachments/Pasted%20image%2020231122180859.png)

## 一个类型实现多个接口
```go
package main

import "fmt"

type Telephone interface {
    DaDianHua()
}

type YouXiTing interface {
    PlayGame()
}

type Iphone struct {
    Brand string
}

func (i Iphone) DaDianHua() {
    fmt.Printf("%s品牌手机可以打电话\n", i.Brand)
}

func (i Iphone) PlayGame() {
    fmt.Printf("%s品牌手机可以玩游戏哟~\n", i.Brand)

}

func main() {
    // 一个类型可以同时实现多个接口，而接口间彼此独立，不知道对方的实现。
    var phone = Iphone{Brand: "苹果"}

    var dianhua Telephone = phone
    var youxi YouXiTing = phone

    // 对Telephone类型调用DaDianHua方法
    dianhua.DaDianHua()

    // 对YouXiTing类型调用PlayGame方法
    youxi.PlayGame()
}
```

## 其他
### 多种类型实现同一个接口
```go
package main

import "fmt"

// CNI 定义一个名为"CNI"接口类型
type CNI interface {
    ContainerNetworkInterface()
}

// Calico 定义一个名为"Calico"的结构体
type Calico struct {
    Name      string
    Advantage string
}

// Flannel 定义一个名为"Flannel"的结构体
type Flannel struct {
    Name      string
    Advantage string
}

// Calico类型实现CNI接口
func (c Calico) ContainerNetworkInterface() {
    fmt.Printf("%s实现了Kubernetes的CNI接口，它的优势在于:%s\n", c.Name, c.Advantage)
}

// Flannel类型实现CNI接口
func (f Flannel) ContainerNetworkInterface() {
    fmt.Printf("%s实现了Kubernetes的CNI接口，它的优势在于:%s\n", f.Name, f.Advantage)
}

func main() {

    // Go语言中不同的类型还可以实现同一接口。
    var obj CNI

    // 将Calico结构体赋值给CNI接口是可以实现的
    obj = Calico{Name: "Calico", Advantage: "容器的网络通信和网络策略"}
    obj.ContainerNetworkInterface()

    // 将Flannel结构体赋值给CNI接口是可以实现的
    obj = Flannel{Name: "Flannel", Advantage: "容器的网络通信"}
    obj.ContainerNetworkInterface()

}

```

# 接口的嵌套和继承
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

## 接口的继承

在Go语言中，接口（interface）是一种定义方法集合的类型，它并不包含方法的具体实现，只是规定实现该接口的类型必须提供这些方法的实现。
而接口之间并不能像类那样直接进行继承，但Go语言提供了接口组合（interface composition）的方式来实现类似继承的效果。

 接口组合是指一个接口可以嵌入（embed）另一个或多个接口，从而继承这些接口的方法集合。所谓组合，是指一个结构体作为另一个结构体的字段。
 通过接口组合，我们可以实现多个接口的功能合并，从而定义更加复杂和灵活的类型。

### 使用方法
Go语言的继承是通过字段嵌套的方式实现的，所谓嵌套，是指一个结构体作为另一个结构体的字段。内嵌结构体的所有字段和方法都成为了外部结构体的成员。

- 定义被嵌套的结构体：被继承的结构体需要先定义，包含一系列字段和方法
- 定义继承结构体：在新定义的结构体中，包含被继承的结构体作为匿名字段
- 使用继承：被继承的结构体的所有字段和方法都可以被新的结构体使用，就好像它自己拥有的一样



### 范例
```go
type Reader interface {
    Read(p []byte) (n int, err error)
}

type Writer interface {
    Write(p []byte) (n int, err error)
}

```
现在，我们想要定义一个`ReadWrite`接口，该接口同时拥有`Reader`和`Writer`接口的功能。我们可以通过在`ReadWrite`接口中嵌入`Reader`和`Writer`接口来实现：

```go
type ReadWrite interface {
	//嵌入匿名结构体
    Reader
    Writer
}
```

这样，任何实现了`ReadWrite`接口的类型都必须同时提供`Read`和`Write`方法的实现。例如，我们可以定义一个结构体`File`，并为其实现`ReadWrite`接口：

```go
type File struct {
    name string
}

func (f *File) Read(p []byte) (n int, err error) {
    // 读取文件的实现
    return len(p), nil
}

func (f *File) Write(p []byte) (n int, err error) {
    // 写入文件的实现
    return len(p), nil
}

```

现在，我们可以将`File`类型作为`ReadWrite`接口来使用：
```go
func main() {
    file := &File{name: "example.txt"}
    var rw ReadWrite
    rw = file

    // 使用 Read 方法
    buf := make([]byte, 10)
    n, err := rw.Read(buf)
    if err != nil {
        fmt.Println("Read error:", err)
    }
    fmt.Printf("Read %d bytes\n", n)

    // 使用 Write 方法
    data := []byte("Hello, world!")
    n, err = rw.Write(data)
    if err != nil {
        fmt.Println("Write error:", err)
    }
    fmt.Printf("Wrote %d bytes\n", n)
}

```

在这个示例中，我们通过接口组合的方式实现了`ReadWrite`接口，该接口同时拥有`Reader`和`Writer`接口的功能。`File`类型实现了`ReadWrite`接口，因此我们可以将其赋值给`ReadWrite`类型的变量，并使用其提供的`Read`和`Write`方法。


## 嵌套结构体实现接口
```go
package main

import (
    "fmt"
)

// WeChat 微信
type WeChat interface {
    Voice()
    video()
}

// VoiceCall 语音电话
type VoiceCall struct {
    Name string
}

// videoCall 视频电话
type videoCall struct {
    //嵌入语音电话匿名结构体
    VoiceCall
}

// 实现WeChat接口的Voice()方法
func (vo VoiceCall) Voice() {
    fmt.Printf("'%s'微信号正在使用语音功能\n", vo.Name)
}

// 实现WeChat接口的video()方法
func (vi videoCall) video() {
    fmt.Println("正在开启视频美颜，滤镜功能...")
}

func main() {
  // 总结: 一个接口的所有方法，不一定需要由一个类型完全实现，接口的方法可以通过在类型中嵌入其他类型或者结构体来实现。

    // 声明一个videoCall类型变量
    var weixin videoCall

    // 定义微信名称
    weixin.Name = "JasonYin2020"

    // 调用WeChat接口方法
    weixin.Voice()
    weixin.video()
}
```


## 使用小结

1. **接口分离原则**：设计小而专一的接口，而不是大而全面的接口。这样可以提高代码的复用性和灵活性。
2. **接口组合**：通过接口嵌入组合多个小接口，形成一个有具体职责的大接口。
3. **避免接口污染**：不要在接口中添加不必要的方法，以保持接口的纯净和实用性。

# 接口的比较

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

## 接口值比较
```go
package main

import "fmt"

type CNI interface {
    ContainerNetworkInterface()
}

type Calico struct {
    Name      string
    Advantage string
}

type Flannel struct {
    Name      string
    Advantage string
}

func (c Calico) ContainerNetworkInterface() {
    fmt.Printf("%s实现了Kubernetes的CNI接口，它的优势在于:%s\n", c.Name, c.Advantage)
}

func (f Flannel) ContainerNetworkInterface() {
    fmt.Printf("%s实现了Kubernetes的CNI接口，它的优势在于:%s\n", f.Name, f.Advantage)
}

func main() {

    /*
    由于接口类型的值可以是任意一个实现了该接口的类型值，所以接口值除了需要记录具体值之外，还需要记录这个值属于的类型。

    也就是说接口值由"类型"和"值"组成，鉴于这两部分会根据存入值的不同而发生变化，我们称之为接口的动态类型和动态值。

    */

    // 此时，接口变量obj是接口类型的零值，也就是它的类型和值部分都是nil
    var obj CNI

    // 我们可以使用"obj == nil"来判断此时的接口值是否为空。
    fmt.Printf("obj是否为空: %t, 类型为: %T, 数据为: %v\n", obj == nil, obj, obj)

    // 我们不能对一个空接口值调用任何方法，否则抛出panic异常: "runtime error: invalid memory address or nil pointer dereference"
    // obj.ContainerNetworkInterface()

    // 接下来，我们将一个"*Calico"结构体指针赋值给变量obj，此时，接口值obj的动态类型会被设置为"*Calico"，动态值为结构体变量的拷贝。
    obj = &Calico{Name: "Calico", Advantage: "容器的网络通信和网络策略"}
    fmt.Printf("obj是否为空: %t, 类型为: %T, 数据为: %v\n", obj == nil, obj, obj)
    // 此时就可以调用接口的方法啦~
    // obj.ContainerNetworkInterface()

    obj = &Flannel{Name: "Flannel", Advantage: "容器的网络通信"}
    fmt.Printf("obj是否为空: %t, 类型为: %T, 数据为: %v\n", obj == nil, obj, obj)
    // obj.ContainerNetworkInterface()

    // 接口值是支持相互比较的，当且仅当接口值的动态类型和动态值都相等时才相等。
    var (
        flannel  CNI = new(Flannel)
        calico   CNI = new(Calico)
        flannel2 CNI = new(Flannel)
    )
    fmt.Printf("flannel == calico ---> %t\n", calico == flannel)
    fmt.Printf("flannel == flannel2 ---> %t\n", flannel == flannel2)

    // 如果接口值保存的动态类型相同，但是这个动态类型不支持互相比较（比如切片），
    // 那么对它们相互比较时就会引发: "panic: runtime error: comparing uncomparable type []int"
    // var (
    //  x interface{} = []int{10, 20, 30}
    //  y interface{} = []int{10, 20, 30}
    // )
    // fmt.Printf("x == y ---> %t\n", x == y)

}
```


```text
结果如下所示：
obj是否为空: true, 类型为: <nil>, 数据为: <nil>
obj是否为空: false, 类型为: *main.Calico, 数据为: &{Calico 容器的网络通信和网络策略}
obj是否为空: false, 类型为: *main.Flannel, 数据为: &{Flannel 容器的网络通信}
flannel == calico ---> false
flannel == flannel2 ---> false
```

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



# interface的优缺点
## 优点/用处

### 泛型的使用
> 比如在一个函数中需要能接收不同类型的参数或者返回不同类型的值，而不是一开始就指定参数或者返回值的类型。
```go
func FuncName(arg1 interface{}, rest ...interface{}) interface{} {  
    // ...  
}
```

### 多态的使用。
在程序设计过程中，可能需要抽象出某些对象共同拥有的方法，这时候多种类型需要实现同一接口，然后通过接口变量指向具体对象来操作这些方法。

## 缺点

每个类型实现的接口并不需要在该类型中显式声明。这可能导致开发人员在不经意间“实现”了一个接口，然后当该接口发生更改时，损坏现有的代码。总的来说，Go的这种接口实现方式提供了很大的灵活性，但也可能带来隐藏的陷阱。

Go语言的接口中并没有标识符，如果一个类型实现了多个接口，并且这些接口中有同名的方法，可能会造成某些问题。

Go的接口是隐式实现的，只要类型实现的方法满足接口定义，那么就算实现了该接口，不需要显式地声明这一点。
```go
type InterfaceA interface {
    DoSomething()
}

type InterfaceB interface {
    DoSomething()
}

type MyType struct{}

func (m MyType) DoSomething() {
    fmt.Println("Doing something")
}
```
在这个例子中，MyType实现了`InterfaceA`和`InterfaceB`，即使它们都有一个同名的方法`DoSomething`。

如果接口有同名但参数或者返回值不同的方法，那么该类型就不能同时实现这两个接口.

```go
type InterfaceA interface {
    DoSomething(int)
}

type InterfaceB interface {
    DoSomething(string)
}
```
在这种情况下，你不能让一个类型同时实现这两个接口，因为无法确保一个方法同时满足两个签名。

要解决这个问题，你可以让你的类型实现一个方法，该方法接受一个空接口参数（可以接受任何类型），然后在方法内部检查并处理不同类型的参数：

```go
type MyType struct{}

func (m MyType) DoSomething(value interface{}) {
    switch v := value.(type) {
    case int:
        fmt.Printf("Doing something with int: %d\n", v)
    case string:
        fmt.Printf("Doing something with string: %s\n", v)
    default:
        fmt.Printf("Don't know how to do something with %T\n", v)
    }
}
```

这种解决办法并不完美，它会使代码变得复杂且难以理解，因此尽可能地避免在不同的接口中使用同名但签名不同的方法。在设计接口时，应尽量保证接口的方法是唯一的，并且清晰地表达了其行为。

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

# Go基础系列：接口类型断言和type-switch 【骏马金龙】
https://www.cnblogs.com/f-ck-need-u/p/9893347.html 

#【骏马金龙系列文章】
https://www.cnblogs.com/f-ck-need-u/p/9832538.html
```