```table-of-contents
```
# 静态代码检查
静态代码检查是一个老生常谈的问题，它通过对源代码进行分析，找出其中的潜在问题和错误，以提高代码质量。

# Go 语言的静态代码检查
##  检查工具linter

在计算机科学中，lint 是一种工具程序的名称，它用来标记源代码中，某些可疑的、不具结构性（可能造成 bug）的段落。它是一种静态程序分析工具，最早适用于 C 语言，在 UNIX 平台上开发出来。后来它成为通用术语，可用于描述在任何一种计算机程序语言中，用来标记源代码中有疑义段落的工具。

Go 语言的静态代码检查工具，常见的有：`gofmt`, `go vet`, `golint`, `golangcli-lint`等等。

### gofmt
Go SDK自带的**代码格式检查工具**，用于检查缩进、文件尾是否有空行等。`go fmt`命令（即`gofmt -l -w`）可使用该工具格式化代码。

例如下面的范例：
```go
package main
import "fmt"
func main() {
  fmt.Println("Hello, "+"World")
}
```
如果使用gofmt将其格式化，会变成下面这样：
```go
package main

import "fmt"

func main() {
    fmt.Println("Hello, " + "World")
}
```
### goimports 
`Goimports`会完成`gofmt`做的所有工作，并且添加缺失的导入并删除未引用的导入。

### go vet
Go SDK自带的**可疑代码检查工具**，检查unsafe.Pointer的错误使用、unreacheable code等可能有问题的代码。

### golint
golang 官方提供的代码风格检查工具，可以检查代码是否符合官方规范。

### golangci-lint
社区提供的**linters聚合运行工具**，比如集成了 go vet 和 golint 等，内置了多个linter（代码检查工具），支持配置化&并行运行。

# golangci-lint
## 介绍
## 特性 
## 集成
### goland中集成golangci-lint
### vscode中集成golangci-lint
## 安装
### mac中安装
### linux中安装

# 参考
```bash
# 静态代码检查利器：golangci-lint
https://juejin.cn/post/7231921877466775589

# golangci-lint配置与使用
https://blog.ivansli.com/2022/06/21/golang-lint/
```