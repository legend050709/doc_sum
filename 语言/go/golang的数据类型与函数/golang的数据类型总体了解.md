```table-of-contents
```

# 数据类型概述

Go 是一种强类型语言。 这意味着你声明的每个变量都绑定到特定的数据类型，并且只接受与此类型匹配的值。

# 分类

Go 有四类数据类型：

- 基本类型：数字、字符串和布尔值
- 聚合类型：数组array和结构struct
- 引用类型：指针pointer、切片slice、映射map、函数function和通道channel
- 接口类型：接口interface

## 基本类型
### 数字类型
- 整数型(int, int8, int16, int32, int64, uint, uint8, uint16, uint32, uint64, byte)
- 浮点类型(float32, float64)
- 复数类型(complex64, complex128 )

#### 整型

一般来说，定义整数类型的关键字是 `int`。 但 Go 还提供了 `int8`、`int16`、`int32` 和 `int64` 类型，其大小分别为 8、16、32 或 64 位的整数。 当你只使用 `int` 时，32 位系统上的大小为 32 位，64 位系统上则为 64 位（大多数情况下如此，不过在不同计算机上或有所不同）。 如果需要将值表示为无符号数字，则可以使用 `uint`，但仅当有特定原因时才使用此类型。 此外，Go 还提供 `uint8`、`uint16`、`uint32` 和 `uint64` 类型。

> 注： int 和 uint：根据底层平台，表示 32 或 64 位有符号和无符号整数。


|类型|说明|范围|
|---|---|---|
|int8|有符号 8 位整型，长度 8bit|-128 到 127|
|int16|有符号 16 位整型|-32768 到 32767|
|int32|有符号 32 位整型|-2147483648 到 2147483647|
|int64|有符号 64 位整型|-9223372036854775808 到 9223372036854775807|
|uint8|无符号 8 位整型，8 位都用于表示数值|0 到 255|
|uint16|无符号 16 位整型|0 到 65535|
|uint32|无符号 32 位整型|0 到 4294967295|
|uint64|无符号 64 位整型|0 到 18446744073709551615|




#### 浮点型 和复数型

|类型|说明|
|---|---|
|float32|IEEE-754 32位浮点型数|
|float64|IEEE-754 64位浮点型数|
|complex64|32 位实数和 32 位虚数|
|complex128|64 位实数和 64 位虚数|

使用 `read(v`) 和 `imag(v)`可以取出复数的实部和虚部。


### 布尔类型

布尔类型仅可能有两个值：`true` 和 `false`。 你可以使用关键字 `bool` 声明布尔类型。 

> 注意：Go 不同于其他编程语言，在 Go 中，你不能将布尔类型隐式转换为 0 或 1。 你必须显式执行此操作。

```go
var featureFlag bool = true
```

### 字符串类型

在 Go 中，关键字 `string` 用于表示字符串数据类型。 若要初始化字符串变量，你需要在双引号（`"`）中定义值。 单引号（`'`）用于单个字符（以及 runes，正如我们在上一节所述）。

```go
var firstName string = "John"
lastName := "Doe"
println(firstName, lastName)
```

### byte字节型
```golang
byte ： 类似于 uint8
```
字节数据类型用于表示无符号的8位整数。 字节数据类型默认为零。


### rune型
`rune` 只是 `int32` 数据类型的别名。 它用于表示 Unicode 字符（或 Unicode 码位）。

```go
rune := 'G' // 此中为单引号
println(rune)
```

### 范例
```golang
package main
import ("fmt")
func main() {
    // bool 布尔类型  true 或false
    var a bool = true
    fmt.Println(a)

    // int 有符号整型， 32位操作系统为4字节
    var b int = -100
    fmt.Println(b)

    //  int8 有符号 8位整型 (-128 到 127)
    var c int8 = -128
    fmt.Println(c)

    // int16 有符号  16位整型  (-32768 到 32767)
    var d int16 = -32768
    fmt.Println(d)

    // int32 有符号  32位整型   (-2147483648 到 2147483647)
    var e int32 = -214743648
    fmt.Println(e)

    // int64 有符号 64位整型    (-9223372036854775808 到 9223372036854775807)
    var f int64 = -9223372036854775808
    fmt.Println(f)

    // uint8 无符号8位整型   (0 到 255)
    var h uint8 = 255
    fmt.Println(h)

    // uint16 无符号16位整型  (0 到 65535)
    var i uint16 = 65535
    fmt.Println(i)

    // uint32 无符号32位整型   (0 到 4294967295)
    var j uint32 = 4294967295
    fmt.Println(j)

    // uint64 无符号64位整型    (0 到 18446744073709551615)
    var k uint64 = 18446744073709551615
    fmt.Println(k)

    // float32 单精度浮点数
    var l float32 = 3.1459
    fmt.Println(l)

    // float64 双精度浮点数
    var m float64 = 3.1415926535
    fmt.Println(m)

    // complex64 32位实数与虚数    由float32类型的实部和虚部组成
    var n complex64 = 3 + 4i
    fmt.Println(n)

    // comlex128 64位实数和虚数   由float64类型的实部和虚部组成
    var o complex128 = 3 + 4i
    fmt.Println(o)

    // byte unit8的别名，用于表示一个ASCII码字符
    var p byte = 'a'
    fmt.Println(p)

    // rune int32的别名, 用于表示一个Unicode码字符
    var q rune = '国'    
    fmt.Println(q)

    // string 字符串类型   存储任意长度的字符序列
    var r string = "hello world!"
    fmt.Println(r)
}

```

![](attachments/Pasted%20image%2020241017143504.png)

## 复合/派生类型
### 聚合类型
#### 数组
#### 结构体

### 引用类型
#### 指针型pointer
```golang
uintptr: 无符号整数，用于存放一个指针。
```
#### 切片slice
#### map
#### 函数类型function
#### 管道channel

### 接口类型

# 数据类型的特性

Go中的每种类型都定义了两个东西：
1. 变量如何存储（底层数据结构）
2. 可以对变量执行什么操作（类型的的函数：可以用于其中的方法和函数）


## 范例

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
这里 `i` 是 `myInt` 类型，但 `originalInt` 是 `int` 类型。
**只要底层数据结构相同，类型就可以互相转换**。

# 各类型变量的默认值

与在其他编程语言中不同的是，在 Go 中，如果你不对变量初始化，所有数据类型都有默认值。 此功能非常方便，因为在使用之前，你无需检查变量是否已初始化。

- 整型`int` 类型的 `0`（及其所有子类型，如 `int64`）
- 浮点型`float32` 和 `float64` 类型的 `+0.000000e+000`
- 布尔型`bool` 类型的 `false`
- 字符串`string` 类型的空值，即`""`;
- 指针Pointer（new产生）：nil
- 接口interface：nil
- 切片slice、映射map和通道channel：nil


# 操作
## type 定义新类型




## 打印变量的数据类型
使用`%T`打印出变量的数据类型
```go
var i int = 10
fmt.Printf("i数据类型:%T \n" , i)

结果：i数据类型:int
```

## 类型转换
在 Go 中隐式强制转换不起作用。 需要显式强制转换。 
Go 提供了将一种数据类型转换为另一种数据类型的一些本机方法。

```go
var integer16 int16 = 127
var integer32 int32 = 32767
println(int32(integer16) + integer32)
```

### 何时可以使用类型转换

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

# 其他

## 字符串和整数的类型转换：strconv包
若要将 `string` 转换为 `int`，可以使用以下代码，反之亦然：
```go
package main

import "strconv"

func main() {
    i, _ := strconv.Atoi("-42")
    s := strconv.Itoa(-42)
    println(i, s)
}
```


# 参考
```bash

```