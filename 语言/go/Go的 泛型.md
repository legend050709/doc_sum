```table-of-contents
```
# 背景
在函数定义中，定义要求的参数叫形参，实际传入的参数叫实参，在强类型编程语言中要求形参和实参的类型一致。

这样就会有一个问题，导致我们的函数限制非常强，特别是在 Golang 的数字类型有很多种的情况下。
```
package main

import "fmt"

func MinInt(a, b int) int {
    if a <= b {
        return a
    } else {
        return b
    }
}

func main() {
    fmt.Println(MinInt(1, 2))
}
```
像这样比较大小的函数，(a,b) 就是形参，(1,2) 就是实参，我们可能对于不同的类型都要分别再写一个函数。
如果能够不限制形参类型，在函数调用的时候再指定具体类型，这个问题是不是就可以解决了呢？

# 概念
**形参**
**实参**
# 参考
```c
# Golang 泛型实践
https://windard.com/blog/2022/05/17/Golang-Generic

https://juejin.cn/post/7183613424230727740
```