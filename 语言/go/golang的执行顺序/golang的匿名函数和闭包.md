```table-of-contents
```
# 匿名函数
## 定义
匿名函数没有函数名，只有函数体，它只有在被调用的时候才会被初始化。
（与之相对的，有名字的函数被称为具名函数）
## 匿名函数的声明和调用
### 申明
Golang 的匿名函数的声明样式如下所示：
```go
func(params)(return params){
    function body
}
```
匿名函数的声明与普通函数的定义基本一致，除了没有名字之外。
### 调用
我们可以在匿名函数声明之后直接调用它。
如下例子所示：
```go
func (name string){
    fmt.Println("My name is ", name)
}("王小二")
```

在声明匿名函数之后，在其后加上调用的参数列表，即可立即对匿名函数进行调用。
除此之外，我们还可以将匿名函数赋值给函数类型的变量，用于多次调用或者求值，如下例子所示：
```go
currentTime := func() {
    fmt.Println(time.Now())
}
// 调用匿名函数
currentTime()
```
## 使用场景
### 赋值给函数变量
```go
 func main() {
     sumFun := func(num1, num2 int) int {
         return num1 + num2
     }
     sum := sumFun(10, 20)
     fmt.Println(sum)
 ​
     return
 }
 ​
 // 输出
 30
```
### 直接执行
```go
 func main() {
     func(name string) {
         fmt.Println("Hello", name)
     }("TOMOCAT")
 ​
     return
 }
 ​
 // 输出
 Hello TOMOCAT
```
###  作为函数参数(回调函数)
```go
 /*
 求和并调用callback函数对结果进行特殊处理
  */
 func sumWorker(data []int, callback func(int)) {
     sum := 0
     for _, num := range data {
         sum += num
     }
 ​
     callback(sum)
 }
 ​
 func main() {
     // 打印出求和结果
     sumWorker([]int{1, 2, 3, 4}, func(a int) {
         fmt.Println("sum:", a)
     })
 ​
     // 判断求和结果是否大于100
     sumWorker([]int{1, 2, 3, 4}, func(a int) {
         if a > 100 {
             fmt.Println("sum > 100")
         } else {
             fmt.Println("sum <= 100")
         }
     })
 }
 ​
 // 输出
 sum: 10
 sum <= 100
```
## 范例
# 闭包
## 定义
![](attachments/Pasted%20image%2020240103174002.png)
闭包是由函数及其相关的引用环境（函数体之外的变量）组合而成的实体。
即：**闭包=函数+引用环境**。可以理解为一个函数“捕获”了和它处于同一作用域的其他变量。

- 函数
> 在闭包实际实现的时候，往往通过调用一个外部函数返回其内部函数来实现的。用户得到一个闭包，也等同于得到了这个内部函数，每次执行这个闭包就等同于执行内部函数。
>注：返回闭包的时候，并不是单纯的返回一个函数，返回的是一个结构体。其中记录了函数返回地址和引用的环境变量的地址。

- 引用环境
>可以简单理解为，在函数外部定义但在函数内被引用的变量。



## 作用
闭包是携带状态的函数，它是将函数内部和函数外部连接起来的桥梁。
通过闭包，我们可以调用定义在函数外的变量，闭包可以使当前函数捕捉到一个外部状态。
## 闭包和匿名函数
闭包只能通过匿名函数实现，我们可以把闭包看作是**有状态的匿名函数**，反过来，如果匿名函数引用了外部变量，就形成了一个闭包（Closure）。

简单来说，「闭」的意思是「封闭外部状态」，即使外部状态已经失效，闭包内部依然保留了一份从外部引用的变量。
## 闭包和全局变量
使用普通函数和全局变量完全可以实现和闭包相同的功能。

​**为什么不用全局变量？**
缩小变量作用域，减少对全局变量的污染。
同时，如果我要实现n个闭包，如果我使用全局变量，就需要维护n个函数对应的全局变量。

## 范例
### 捕获自由变量
当一个函数引用了外部函数的一些变量时，在外部函数调用结束时，被引用的变量不会被释放。也就是说闭包捕获了这些变量，它不关心这些捕获了的变量是否已经超出了作用域，只要闭包还在使用它，这些变量就还会存在。

**捕获的本质就是引用传递而非值传递。**
```go
 func main() {
     i := 0
 ​
     // 闭包: i是引用传递
     defer func() {
         fmt.Println("defer closure i:", i)
     }()
 ​
     // 非闭包: i是值传递
     defer fmt.Println("defer i:", i)
 ​
     // 修改i的值
     i = 100
     
     return
 }
 ​
 // 输出
 defer i: 0
 defer closure i: 100
```

闭包对它作用域上部的变量可以进行修改，修改引用的变量会对变量进行实际修改。
```go
package main
import "fmt"

func main() {
   x := 1
   y := func() int {
      x += 1
      return x
   }()
   fmt.Println("main:", x, y)
}
# 结果
main: 2 2
```

### 闭包的隔离性
在函数式语言中，当内嵌函数体内引用到体外的变量时，将会把定义时涉及到的引用环境和函数体打包成一个整体（闭包）返回。

由于闭包把函数和运行时的引用环境打包成为一个新的整体，所以就解决了函数编程中的嵌套所引发的问题。
当每次调用包含闭包的函数时都将返回一个新的闭包实例，这些实例之间是隔离的，分别包含调用时不同的引用环境现场。
不同于函数，闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例。

```go
 /*
 返回匿名函数的函数
 1) x是自由变量
 2) 匿名函数和x自由变量共同组成闭包
  */
 func Adder(x int) func(int) int{
     return func(y int) int{
         x += y
         fmt.Printf("x addr %p, x value %d\n", &x, x)
         return x
     }
 }
 ​
 func main() {
     fmt.Println("----------------Adder()返回的匿名函数实例1----------------")
     closure := Adder(1)
     closure(100)
     closure(1000)
     closure(10000)
 ​
     fmt.Println("----------------Adder()返回的匿名函数实例2----------------")
     closure2 := Adder(10)
     closure2(1)
     closure2(1)
     closure2(1)
     closure2(1)
 ​
     return
 }
 ​
 // 输出
 ----------------Adder()返回的匿名函数实例1----------------
 x addr 0xc00002a958, x value 101
 x addr 0xc00002a958, x value 1101
 x addr 0xc00002a958, x value 11101
 ----------------Adder()返回的匿名函数实例2----------------
 x addr 0xc00002a978, x value 11
 x addr 0xc00002a978, x value 12
 x addr 0xc00002a978, x value 13
 x addr 0xc00002a978, x value 14
```
**分析**
通过`Adder()`返回一个匿名函数，这个匿名函数和自由变量`x`（函数外变量）组成闭包，只要匿名函数的实例`closure`没有消亡，那么`x`都是引用传递。



```go
package main

import (
   "fmt"
)

func Fibonacci() func() int {
   a, b := 0, 1
   return func() int {
      a, b = b, a+b
      return a
   }
}
func main() {
   f1, f2 := Fibonacci(), Fibonacci()
   for i := 0; i < 10; i++ {
      fmt.Printf("Fibonacci: %d %d\n", f1(), f2())
   }
   fmt.Println(&f1, &f2)
}
/*
Fibonacci: 1 1
Fibonacci: 1 1
Fibonacci: 2 2
Fibonacci: 3 3
Fibonacci: 5 5
Fibonacci: 8 8
Fibonacci: 13 13
Fibonacci: 21 21
Fibonacci: 34 34
Fibonacci: 55 55
0xc0000ae018 0xc0000ae020
*/
```
**分析**
![](attachments/Pasted%20image%2020240103174951.png)

### 闭包中使用值传递
由于闭包的存在，Golang中使用匿名函数的时候要特别注意区分清楚引用传递和值传递。根据实际需要，我们在不需要引用传递的地方通过匿名函数参数赋值的方式实现值传递。

```go
 func main() {
     fmt.Println("----------------引用传递----------------")
     for i := 0; i < 10; i++ {
         go func() {
             fmt.Println(i)
         }()
     }
     time.Sleep(10 * time.Millisecond)
 ​
     fmt.Println("----------------值传递----------------")
     for i := 0; i < 10; i++ {
         go func(x int) {
             fmt.Println(x)
         }(i)
     }
     time.Sleep(10 * time.Millisecond)
 ​
     return
 }
 ​
 // 输出
 ----------------引用传递----------------
 1
 10
 10
 10
 10
 10
 10
 10
 10
 10
 ----------------值传递----------------
 4
 9
 1
 6
 5
 7
 2
 3
 8
 0
```

## 闭包常见的坑
### for range 中使用闭包
```go
package main

import (
   "fmt"
   "sync"
)

func main() {
   s := []string{"a", "b", "c"}
   var wg sync.WaitGroup
   for i, v := range s {
      wg.Add(1)
      go func() {
         fmt.Println(i, v)
         wg.Done()
      }()
   }
   wg.Wait()
}

/*
2 c
2 c
2 c
*/
```

### 函数列表使用不当
**现象**
```go
package main

func main() {
   var funcSlice []func()
   for i := 0; i < 3; i++ {
      funcSlice = append(funcSlice, func() {
         println(i)
      })

   }
   for j := 0; j < 3; j++ {
      funcSlice[j]()
   }
}

# 结果
3
3
3
```

**分析**
每次 append 操作仅将匿名函数放入到列表中，但并未执行，并且引用的变量都是 i，随着 i 的改变匿名函数中的 i 也在改变，所以当执行这些函数时，他们读取的都是环境变量 i 最后一次的值。


**解决方法**
声明新变量：
> 这相当于为这三个函数各声明一个变量，一共三个，这三个变量初始值分别对应循环中的 i 并且之后不会再改变。

```go
package main

func main() {
   var funcSlice []func()
   for i := 0; i < 3; i++ {
      i := i
      funcSlice = append(funcSlice, func() {
         println(i)
      })

   }
   for j := 0; j < 3; j++ {
      funcSlice[j]()
   }
}
# 结果
0
1
2
```

声明新匿名函数并传参：
> 现在 println(i)使用的 i是通过函数参数传递进来的，并且 Go 语言的函数参数是按值传递的。  现在相当于在这个新的匿名函数内声明了三个变量，被三个闭包函数独立引用。原理跟第一种方法是一样的。

```go
package main

func main() {
   var funcSlice []func()
   for i := 0; i < 3; i++ {
      func(i int) {
         funcSlice = append(funcSlice, func() {
            println(i)
         })
      }(i)

   }
   for j := 0; j < 3; j++ {
      funcSlice[j]()
   }
}
# 结果
0
1
2
```

### defer与闭包
```go
package main

import "fmt"

func main() {
    x, y := 1, 2

    defer func(a int) { 
        fmt.Printf("x:%d,y:%d\n", a, y)  // y 为闭包引用
    }(x)      // 复制 x 的值

    x += 100
    y += 100
    fmt.Println(x, y)
}

# 结果
101 102
x:1,y:102
```
在defer定义时 已经将x的拷贝 1 复制给了defer, defer执行时使用的是当时defer定义时x的拷贝，而不是当前环境中x的值。


```go
package main

import (
   "fmt"
)

func f1() (result int) {
   defer func() {
      result++
   }()
   return 0
}

func f2() (r int) {
   t := 5
   defer func() {
      t = t + 5
   }()
   return t
}

func f3() (r int) {
   defer func(r int) {
      r = r + 5
   }(r)
   return 1
}

func main() {
   fmt.Println(f1(), f2(), f3())
}

# 结果：1 5 1
```

## 小结
- 什么是闭包
>匿名函数+其“捕获”的自由变量被称为闭包。

- 什么是自由变量，自由变量的特性，生命周期
>被闭包捕获的变量称为“自由变量”，在匿名函数实例（匿名函数变量）未消亡时共享内存地址（引用传递）。

- 闭包的隔离性
>同一个匿名函数可以构造多个实例，每个实例内的自由变量地址不同。

- 闭包内部的局部变量
>匿名函数内部的局部变量在每次执行匿名函数时地址都是变换的。

- 闭包的值传递
>通过匿名函数参数赋值的方式可以实现值传递。

# 参考
```c
# 通过实例理解Go匿名函数及闭包
https://zhuanlan.zhihu.com/p/351428978

# 探究Golang中的闭包
https://llmxby.com/2022/08/27/%E6%8E%A2%E7%A9%B6Golang%E4%B8%AD%E7%9A%84%E9%97%AD%E5%8C%85/
```