```table-of-contents
```
# 技巧
## 链式调用
链式调用技术可以应用于函数（指针）接收器。
为了说明这一点，让我们考虑一个 `Person` 结构，它有两个函数 `AddAge` 和 `Rename`，用于对其进行修改。
```go
type Person struct {
  Name string
  Age  int
}

func (p *Person) AddAge() {
  p.Age++
}

func (p *Person) Rename(name string) {
  p.Name = name
}
```
如果你想给一个人增加年龄然后给他们改名字，常规的方法是：
```go
func main() {
  p := Person{Name: "Aiden", Age: 30}

  p.AddAge()
  p.Rename("Aiden 2")
}
```


或者，我们可以修改 `AddAge` 和 `Rename` 函数接收器，使其返回修改后的对象本身，即使它们通常不返回任何内容。
```go
func (p *Person) AddAge() *Person {
  p.Age++
  return p
}

func (p *Person) Rename(name string) *Person {
  p.Name = name
  return p
}
```
通过返回修改后的对象本身，我们可以轻松地将多个函数接收器链在一起，而无需添加不必要的代码行：
```go
p = p.AddAge().Rename("Aiden 2")

```

## Go 1.20 允许将切片解析为数组或数组指针
当我们需要将切片转换为固定大小的数组时，不能直接赋值，例如：
```go
a := []int{0, 1, 2, 3, 4, 5}
var b [3]int = a[0:3]

// 在变量声明中不能将 a[0:3]（类型为 []int 的值）赋值给 [3]int 类型的变量
// （不兼容的赋值）
```

为了将切片转换为数组，Go 团队在 Go 1.17 中更新了这个特性。随着 Go 1.20 的发布，借助更方便的字面量，转换过程变得更加简单：
```go
// Go 1.20
func Test(t *testing.T) {
   a := []int{0, 1, 2, 3, 4, 5}
   b := [3]int(a[0:3])

  fmt.Println(b) // [0 1 2]
}

// Go 1.17
func TestM2e(t *testing.T) {
  a := []int{0, 1, 2, 3, 4, 5}
  b := *(*[3]int)(a[0:3])

  fmt.Println(b) // [0 1 2]
}
```

## 使用 "import _" 进行包初始化
有时，在库中，你可能会遇到结合下划线 (_) 的导入语句，如下所示：
```go
import (
  _ "google.golang.org/genproto/googleapis/api/annotations"
)
```
这将执行包的初始化代码（init 函数），而无需为其创建名称引用。这允许你在运行代码之前初始化包、注册连接和执行其他任务。
让我们通过一个示例来更好地理解它的工作原理：
```go
// 下划线
package underscore

func init() {
  fmt.Println("init called from underscore package")
}
// main
package main

import (
  _ "lab/underscore"
)

func main() {}
// 输出：init called from underscore package
```
## 使用 "import ." 进行导入
作为开发者，点 (.) 运算符可用于在不必指定包名的情况下使用导入包的导出标识符，这对于懒惰的开发者来说是一个有用的快捷方式。

很酷，对吧？这在处理项目中的长包名时特别有用，比如 `externalmodel` 或 `doingsomethinglonglib`。
为了演示，这里有一个简单的例子：
```go
package main

import (
  "fmt"
  . "math"
)

func main() {
  fmt.Println(Pi) // 3.141592653589793
  fmt.Println(Sin(Pi / 2)) // 1
}
```
## Go 1.20 允许将多个错误合并为单个错误
Go 1.20 引入了对错误包的新功能，包括对多个错误的支持以及对 `errors.Is` 和 `errors.As` 的更改。

在 `errors` 中添加的一个新函数是 `Join`
```go
var (
  err1 = errors.New("Error 1st")
  err2 = errors.New("Error 2nd")
)

func main() {
  err := err1
  err = errors.Join(err, err2)

  fmt.Println(errors.Is(err, err1)) // true
  fmt.Println(errors.Is(err, err2)) // true
}
```
## 检查接口是否为真正的 nil
即使接口持有的值为 `nil`，也不意味着接口本身为 `nil`。这可能导致 Go 程序中的意外错误。因此，重要的是要知道如何检查接口是否为真正的 `nil`。
```go
func main() {
  var x interface{}
  var y *int = nil
  x = y

  if x != nil {
    fmt.Println("x != nil") // <-- 实际输出
  } else {
    fmt.Println("x == nil")
  }

  fmt.Println(x)
}

// 输出：
// x != nil
// <nil>
```

我们如何确定 `interface{}` 值是否为 `nil` 呢？幸运的是，有一个简单的工具可以帮助我们实现这一点：
```go
func IsNil(x interface{}) bool {
  if x == nil {
    return true
  }

  return reflect.ValueOf(x).IsNil()
}
```
## 多参数函数添加参数注释
当处理具有多个参数的函数时，仅通过阅读其用法来理解每个参数的含义可能会令人困惑。考虑以下示例：
```go
printInfo("foo", true, true)
```
如果不检查 `printInfo` 函数，那么第一个 'true' 和第二个 'true' 的含义是什么呢？当你有一个具有多个参数的函数时，仅通过阅读其用法来理解参数的含义可能会令人困惑。

但是，我们可以使用注释使代码更易读。例如：
```go
// func printInfo(name string, isLocal, done bool)

printInfo("foo", true /* isLocal */, true /* done */)
```

# 参考
```c
https://juejin.cn/post/7303390778490028070
```