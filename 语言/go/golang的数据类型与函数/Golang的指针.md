```table-of-contents
```
# 总结
![](attachments/Pasted%20image%2020231217180554.png)
# 基础
## 指针概念
在 Go 中，指针是**一种存储变量内存地址**的变量。在指针类型变量中存储的是另一个变量的地址，而不是该变量的值本身。

Go指针类型允许对这个指针类型指向的数据进行修改，传递数据可以通过传递指针来实现，无须拷贝数据。指针类型不能够进行偏移和运算。
因此Go中的指针类型变量拥有指针高效访问的特点，且不会发生指针偏移，避免了非法修改指针从而导致非法内存访问和指针溢出等问题，提高代码安全性。同时，`垃圾回收`也比较容易对不会发生偏移的指针进行检索和回收。

当声明并初始化一个变量时：
```go
var num int = 10
```
上述声明并初始化时，在内存中开辟了一块空间，该空间存放着数值10，此时该空间有一个唯一的地址来进行标识，此时指向这个地址的变量称为`指针变量`（指针）。
![](attachments/Pasted%20image%2020231217180942.png)

当一个指针变量被定义后`没有指向到任何变量`时，它的默认值为 `nil`。
```go
func main() {
    var num *int
    fmt.Println(num)
}

// 执行结果
<nil>
```
## 取地址
Go程序中，每个变量在运行时都拥有一个在内存分配的地址来表示变量在内存中的位置。
Go语言中使用`&`字符放在变量前面对变量进行“取地址”操作。

取变量指针的语法如下：
```go
ptr := &v    // v的类型为T
```
- `v`: 代表被取地址的变量，类型为`T`
- `ptr`: 用于接收地址的变量，ptr的类型就为`*T`，其中T为指针的类型。`*`代表指针。
```go
func main() {
    num := 10
    address := &num
    fmt.Printf("nums: %d ptr: %p\n", num, &num)
    fmt.Printf("address: %p type: %T\n", address, address)
    fmt.Println(&address)
}

// 执行结果
nums: 10 ptr: 0xc0000aa058
address: 0xc0000aa058 type: *int
0xc0000ce018
```

## 指针类型
Go语言中的值类型（int、float、bool、string、array、struct）都有对应的指针类型，如：`*int`、`*int64`、`*string`等。
```go
package main

import "fmt"

type Person struct {
    Name string
    Age  int
}

func main() {
    var x int = 255
    var ptrInt *int = &x
    fmt.Printf("int pointer: %T\n", ptrInt)

    var str string = "hello"
    var ptrStr *string = &str
    fmt.Printf("string pointer: %T\n", ptrStr)

    var b bool = true
    var ptrBool *bool = &b
    fmt.Printf("bool pointer: %T\n", ptrBool)

    var f float64 = 3.14
    var ptrFloat64 *float64 = &f
    fmt.Printf("float64 pointer: %T\n", ptrFloat64)

    var arr [3]int = [3]int{1, 2, 3}
    var ptrArr *[3]int = &arr
    fmt.Printf("[3]int pointer: %T\n", ptrArr)

    var p Person = Person{"Tom", 20}
    var ptrP *Person = &p
    fmt.Printf("Person pointer: %T\n", ptrP)
}

// 执行结果
int pointer: *int
string pointer: *string
bool pointer: *bool
float64 pointer: *float64
[3]int pointer: *[3]int
Person pointer: *main.Person

```

## 取内容
使用`&`操作符对变量进行取地址操作后会得到该变量的指针，对指针使用`*`操作，可以获得该指针所指向的值，即`指针取内容`。
```go
func main() {
    num := 256
    addr := &num
    fmt.Printf("type of addr: %T\n", addr)
    value := *addr
    fmt.Printf("type of value: %T\n", value)
    fmt.Printf("value: %v\n", value)
}

// 执行结果
type of addr: *int
type of value: int
value: 256
```

通过取地址操作符`&`获取到变量的地址，通过取值操作符`*`可以获取到地址指向的值。

# 指针使用
使用指针在函数或者方法传参时：

- 使用指针，可以**在函数内部修改实参的值**；
- 使用指针，可以**避免参数副本的内存消耗**；

在 Go 函数中，函数参数的传递方式是`值传递`（Pass by Value），会将参数值（变量值）进行拷贝并传入到函数中，即传递的是参数值的副本。

# 参考
```go
# Go指针剖析
https://juejin.cn/post/7303789777789419561
```