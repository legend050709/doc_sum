```table-of-contents
```
# 总结
![](attachments/Pasted%20image%2020231217180554.png)
# 基础
## 指针概念
在 Go 中，指针是**一种存储变量内存地址**的变量。在指针类型变量中存储的是另一个变量的地址，而不是该变量的值本身。

Go指针类型允许对这个指针类型指向的数据进行修改，传递数据可以通过传递指针来实现，无须拷贝数据。
指针类型不能够进行偏移和运算。
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
使用`&`操作符对变量进行取地址操作后会得到该变量的指针，
对指针使用`*`操作，可以获得该指针所指向的值，即`指针取内容`。
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

# go中的指针和C中的指针的对比
在Go语言中，直接砍掉了 C 语言指针最复杂的指针运算部分（即**go中指针不能够进行偏移和运算**），只留下了获取指针（`&`运算符）和获取对象（`*`运算符）的运算，用法和C语言很类似。

另外，Go语言中没有`->`操作符来调用指针所属的成员，而与一般对象一样，都是使用`.`来调用。

Go 语言中一个指针被定义后没有分配到任何变量时，它的值为`nil`。

# 指针隐式间接引用
## golang 指针Dereferencing（解引用）
在 Go 语言中，指针解引用（dereferencing）是指通过指针访问指针所指向的内存地址上存储的值。在指针变量前加上 * 符号可以进行指针解引用操作。指针解引用会返回指针所指向的内存地址上存储的值。例如，假设有一个指向int类型变量的指针：
```go
var x int = 42
var p *int = &x
```
要访问p指针指向的值，可以使用指针解引用：
```go
fmt.Println(*p) // 输出：42
```

## golang的 指针隐式解引用
Go 语言自带指针隐式解引用 ：对于一些**复杂类型**的指针， 如果要访问成员变量时候需要写成类似`*p.field`的形式时，只需要`p.field`即可访问相应的成员。
以下复杂类型自带指针隐式解引用：

### 结构体类型指针隐式间接引用
结构体字段可以通过结构体指针来访问。如果我们有一个指向结构体的指针 `p`，那么可以通过 `(*p).X` 来访问其字段 `X`。不过这么写太啰嗦了，所以语言也允许我们使用隐式间接引用，直接写 `p.X` 就可以。

示例代码如下：
```go
package main
 
import (
    "fmt"
)
 
type Student struct {
    name   string
    age    int
    weight float32
    score  []int
}
 
func main(){
   pp := new(Student) //使用 new 关键字创建一个指针
   *pp = Student{"qishuangming", 23, 65.0, []int{2, 3, 6}}
   fmt.Printf("stu pp have %d subjects\n", len((*pp).score)) //按照我们对指针的了解，对Student结构体对象pp显示赋值的话需要使用解引用语法进行赋值，但是实际编码时都会省去*，写法如下行所示。
   fmt.Printf("stu pp have %d subjects\n", len(pp.score)) //编译器会自动将指针解引用，并访问结构体中的对应字段，这个过程被称为隐式间接引用。
}

```


### 数组类型指针隐式间接引用

指向数组的指针可以隐式解引用数组中的元素。
```go
var arr [3]int
p := &arr
p[0] = 1 // 等价于 (*p)[0] = 1
```
### 切片类型指针隐式间接引用

切片实际上是对底层数组的封装，因此指向切片的指针可以隐式解引用切片中的元素

```go
s := []int{1, 2, 3}
p := &s
p[0] = 4 // 等价于 (*p)[0] = 4
```

### 字典类型隐式间接引用
map 是引用类型，当我们使用 `map` 类型的变量访问元素时，也不需要使用 `*` 运算符进行解引用，Golang 会自动帮我们解引用。

```go
m := map[string]int{"a": 1, "b": 2}
fmt.Println(m["a"]) // 隐式解引用
```

### func 类型隐式间接引用

在 Golang 中，函数类型也是一种类型，它可以使用指针类型来表示函数的地址。如果我们定义了一个函数类型的变量，并将一个函数的地址赋值给它，那么我们可以直接调用该变量，并且不需要使用 `*` 运算符进行解引用。

例如，以下代码演示了函数类型指针的隐式解引用：
```go
type Add func(a, b int) int
 
func main() {
    var add Add
    add = func(a, b int) int {
        return a + b
    }
    println(add) //0x10cf168
 
    sum := add(1, 2) // 隐式解引用
    fmt.Println(sum)
}
```

在上面的代码中，我们定义了一个函数类型 `Add`，它接受两个 `int` 类型的参数并返回一个 `int` 类型的值。我们定义了一个变量 `add`，它的类型是 `Add`。我们将一个函数的地址赋值给了 `add` 变量，然后直接调用了 `add` 变量，不需要使用 `*` 运算符进行解引用。

### 基本数据类型的指针解引用

在 Go 中，基本类型（如 int、float、bool 等）以及字符串类型等非引用类型都没有指针隐式解引用的行为。这意味着，如果需要访问基本类型的指针指向的值，必须显式地使用 * 运算符来解引用指针。
下面是一个示例：
```go
var i int
p := &i
*p = 1 // 显式解引用指针来修改指针所指向的值
fmt.Println(i) // 输出 1
```

另外，对于基本类型而言，使用指针可能会导致性能下降。
因此，在使用指针时应该谨慎，并且只在必要的情况下使用指针来传递数据。

### 小结

在 Go 中，指针隐式解引用是指通过指针直接访问指针所指向的值，而不需要显式地使用 * 运算符来解引用指针，编译器会自动将指针解引用。对于一些**复杂类型**的指针（结构体类型指针、数组/切片类型指针、字典类型、func类型）， 如果要访问成员变量时候需要写成类似`*p.field`的形式时，只需要`p.field`即可访问相应的成员。

# 指针使用
使用指针在函数或者方法传参时：

- 使用指针，可以**在函数内部修改实参的值**；
- 使用指针，可以**避免参数副本的内存消耗**；

在 Go 函数中，函数参数的传递方式是`值传递`（Pass by Value），会将参数值（变量值）进行拷贝并传入到函数中，即传递的是参数值的副本。


# 参考
```go
# Go指针剖析
https://juejin.cn/post/7303789777789419561

# Golang指针隐式间接引用
https://www.cnblogs.com/zhangmingcheng/p/17403603.html

```