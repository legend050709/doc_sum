```table-of-contents
```
# 包（package） 
## 介绍
```bash
A package is a source code organization unit: it defines a context for identifier names, determining the visibility of an identifier within the package. Every package is associated with a directory of the same name. All the files in a package’s directory must declare the same package name. – Go doc
```
golang中 包（package） 是组织代码和复用的单元。
==每个包都与一个同名的目录关联。包目录中的所有文件必须声明相同的包名==。
这是一种组织和封装代码的方式。这种方式使得代码结构清晰，易于理解和维护。

**Go 程序组织代码的基本单位是包（package）**，就像说生物体的基本组成单位是细胞一样。

## 包的特点

（1）**golang 的所有go文件都需要指定其所在的包（package）**。
指定方式是，在go文件的第一行中添加包的声明，声明当前`.go`文件归属的包。
```go
package packagename
```

(2) **一个包一般和一个目录是一一对应的**。

目录下的所有的go文件属于同一个包。即：每个包都与一个同名的目录关联。
包目录中的所有文件必须声明相同的包名。包名是文件夹的路径名，而非声明的 `package xxxx`，包名通常与目录名一致。

> 注：一个文件夹下面直接包含的文件只能归属一个包，同一个包的文件不能在多个文件夹下。


（3）**package对应的目录下可以有子目录，只不过子目录下就是另一个package**。

（4）**package 内资源的使用**
==包内==：`package`内可以使用同一个`package`的所有资源，不需要像c一样先声明再使用。
==包外==：`import` 导入的是包名。当`import`一个`package`后，就可以使用其中的大写字母开头的函数、变量、常量、数据类型等。


(5) 一个包中可以有多个 `init()`，会在被导入的时候执行。
同一个文件中的多个 `init()`按顺序执行，不同文件中的多个 `init()` 不保证执行顺序。

（6）go中每个包都会提供一个独立的命名空间。
这意味着**在不同的包中，你可以使用相同的标识符名称**，而不会产生冲突。

## 包为什么是代码组织的基本单位
```go
A package is a source code organization unit: it defines a context for identifier names, determining the visibility of an identifier within the package. Every package is associated with a directory of the same name. All the files in a package’s directory must declare the same package name. – Go doc
```

包是源代码的组织单位：它为标识符名称定义了一个上下文，决定了一个标识符在包内的可见性。每个包都与一个同名的目录关联。包目录中的所有文件必须声明相同的包名。

这是一种组织和封装代码的方式。这种方式使得代码结构清晰，易于理解和维护。

更直白的翻译一下：**Go 程序组织代码的基本单位是包（package）**，就像说生物体的基本组成单位是细胞一样。

**包为什么是代码组织的基本单位**？接下来从命名空间、封装以及代码复用等方面聊聊。

### 命名空间（隔离性）
**命名空间(namespace)** 是一个用于管理标识符名称（比如：变量名、函数名，常量名）的区域范围，要保证同一个命名空间内的标识符名称是唯一的。

Go语言中，每个包都会提供一个独立的命名空间。这意味着**在不同的包中，你可以使用相同的标识符名称**，而不会产生冲突。

而在同一个包内，不同文件之间中定义的变量、方法，都可以直接进行访问。

**避免命名冲突**极大地提高了代码的可维护性。特别是在大型程序中，不同模块或库很可能会使用相同的变量名、函数名。

#### 标识符的可见性

在同一个包内部声明的标识符都位于同一个命名空间下，在不同的包内部声明的标识符就属于不同的命名空间。
想要在包的外部使用包内部的标识符就需要添加包名前缀，例如`fmt.Println("Hello world!")`，就是指调用`fmt`包中的`Println`函数。

##### 设置标识符的可见性
如果想让一个包中的标识符（如变量、常量、类型、函数等）能被外部的包使用，那么标识符必须是对外可见的（public）。
在Go语言中是通过==标识符的首字母大/小写==来控制标识符的对外可见（public）/不可见（private）的。
在一个包内部只有首字母大写的标识符才是对外可见的。

##### 可见性范例
我们定义一个名为`demo`的包，在其中定义了若干标识符。在另外一个包中并不是所有的标识符都能通过`demo.`前缀访问到，因为只有那些首字母是大写的标识符才是对外可见的。

```go
package demo

import "fmt"

// 包级别标识符的可见性

// num 定义一个全局整型变量
// 首字母小写，对外不可见(只能在当前包内使用)
var num = 100

// Mode 定义一个常量
// 首字母大写，对外可见(可在其它包中使用)
const Mode = 1

// person 定义一个代表人的结构体
// 首字母小写，对外不可见(只能在当前包内使用)
type person struct {
	name string
	Age  int
}

// Add 返回两个整数和的函数
// 首字母大写，对外可见(可在其它包中使用)
func Add(x, y int) int {
	return x + y
}

// sayHi 打招呼的函数
// 首字母小写，对外不可见(只能在当前包内使用)
func sayHi() {
	var myName = "七米" // 函数局部变量，只能在当前函数内使用
	fmt.Println(myName)
}
```

同样的规则也适用于结构体，结构体中可导出字段的字段名称必须首字母大写。

```go
type Student struct {
	Name  string // 可在包外访问的方法
	class string // 仅限包内访问的字段
}
```

### 封装（可见性）

**封装**是**将数据和操作这些数据的方法组合**在一起的机制，其主要目标是**隐藏对象的内部状态和实现细节，只暴露必要的接口给外部使用者**。

封装通常用于面向对象编程（OOP）中，可以通过类和对象来实现。而 Go 语言并不是传统意义上的面向对象编程语言，但其**通过使用包也实现了良好的封装**。

#### 包内部访问资源
在同一个包内，不同文件之间中定义的变量、方法，都可以直接进行访问。

#### 外部访问包资源：可导出与不可导出的标识符名称
在一个包内部，**类型、变量、常量和函数** 分为耳熟能详的两类：

- **可导出（Exported）**：大写字母开头，可以被包外的代码访问
- **非可导出（Unexported）**：以小写字母开头，只能在包内访问

这种方式提供了良好的封装性，可以隐藏实现细节，只暴露必要的接口给其他包。而==通过大小写字母来开头的方式来实现可见性==，比一些复杂的类的 `public`，`extern` 之类的方式，更简洁明了。


### 代码复用
代码复用是软件开发中的一个关键实践，通过代码复用，可以提高开发效率、减少错误、提高代码质量和维护性。

Go语言的包也是代码重用的基本单位。通过 `import` 语句可以引入其他包。

- **内置包**：Go语言提供了丰富的标准库（内置包），这些标准库以包的形式组织，涵盖了常见的功能模块，如I/O操作、字符串处理、网络编程等。
- **自定义包**：开发者可以创建自定义包，在一个仓库/模块中，创建多个包，包之间可以引入
- **第三方包**：Go语言生态系统中有大量第三方包，通过 `go get` 命令可以轻松获取和使用这些包。

通过 `go get -u`  + `包的路径`，可以从指定的远程仓库中获取包，并将其安装到本地的Go环境中。
比如：
![](attachments/Pasted%20image%2020241216114208.png)

## 包的命名
### 命名原则

go语言的包的命名，遵循==简洁、小写、和go文件所在目录同名的原则==。
这样就便于我们引用，书写以及快速定位查找。

### 本地的包
比如go自带的http这个包，它这个http目录下的所有go文件都属于这个http包,所以我们使用http包里的函数、接口的时候，导入这个http包就可以了。
```go
package main

import "net/http"

func main() {
	http.ListenAndServe("127.0.0.1:80",handler);
}

```

#### 包的全路径
从这个例子可以看到，我们导入的是`net/http`, 这在go里叫做全路径，因为http包在net里面，net是最顶级的包，所以必须使用全路径导入，go编译程序才能找到http这个包，和我们文件系统的目录路径是一样的。

因为有了全路径，所以命名的包名可以和其他库的一样，只要它们的全路径不同就可以了，使用全路径的导入，也增加了包名命名的灵活性。

### 域名作为顶级包名
对于自己或者公司开发的程序而言，我们一般采用域名作为顶级包名的方式，这样就不用担心和其他开发者包名重复的问题了，比如我的个人域名是`www.flysnow.org`,那么我自己开发的go程序都以`flysnow.org`作为全路径中的最顶层部分，比如导入我开发的一个工具包:
```go
package main

import "flysnow.org/tools"
```

### `github.com/user`作为顶级包名
如果你没有自己的域名，怎么办呢？这时候可以使用`github.com`。干研发这一行的，在`github`都会有个账号，如果没有赶紧申请一个，这时候我们就可以使用`github.com/<username>`作为你的顶级路径了，别人是不会和你重名的。

```go
package main

import "github.com/rujews/tools"
```
这就是换成`github.com`命名的方式。

## 分类

包有两种类型，一种是 `main` 包，使用 `package main` 在代码的最前面声明。
另外一种就是 `非main` 包，使用 `package + 包名` 。

 `main` 包 可以编译为可执行文件。
 
### main包
当把一个go文件的包名声明为`main`时，就等于告诉go编译程序，我这个是一个可执行的程序，那么go编译程序就会尝试把它编译为一个二进制的可执行文件。

一个`main`的包，一定会包含一个`main()`函数，这个函数也是程序的入口。也只有 `main` 包可以编译成可执行的文件。

注：在go语言里，同时要满足`main`包和包含`main()`函数，才会被编译成一个可执行文件。

```go
package main

import "fmt"

func main() {
	fmt.Println("Hello, 世界")
}

```

### 非main包


## 导入包
要想使用一个包，必须先导入它才可以使用。
Go语言提供了`import`关键字来导入一个包，这个关键字告诉Go编译器到磁盘的哪里去找要想导入的包，所以导入的包必须是一个**全路径的包，也就是包所在的位置**。

### 导入语法
`import`语句通常放在`go`文件开头，package声明语句的下方。

```bash
import [packagename] "path/package"
```

- `import`：导入语句，通常放在go文件开头包声明语句的下面；
- `packagename`：自定义包名，可以通过自定义包名为导入的包重新命名，可以防止同包名导入产生歧义。通常都省略，默认值为引入包的包名。
- `path/package`：引入包的路径名称，需要使用双引号裹起来。。

==注：Go语言中禁止循环导入包。==

### 导入包的分类
#### 导入内置包
在Go语言中提供了许多的内置包，例如`fmt`、`os`、`io`等，而`fmt`包则是我们经常使用的一个内置包。

Go 语言的标准库是随着 Go 版本发布的，因此在编写Go代码时，引入标准库的包（例如 `import "fmt"`）不需要指定具体的版本号。

#### 导入自定义包
Go语言提供了丰富的标准库，这些标准库以包的形式组织，涵盖了常见的功能模块，如I/O操作、字符串处理、网络编程等。

#### 导入第三方包
第三方包和模块是指除了**模块自身中的包**与**标准库（内置包）** 之外的包和模块。

Go语言生态系统中有大量第三方包，通过 `go get` 命令可以轻松获取和使用这些包。

如果有的Go包共享在Github上，我们一样有办法使用他们，这就是远程导入包了。如下所示：
```go
import "github.com/spf13/cobra"
```

这种导入，==前提必须是该包托管在一个分布式的版本控制系统上==，比如`Github`、`Bitbucket`等，并且是==`Public`的权限，可以让我们直接访问它们==。

### 导入包的查找
现在我导入了包，那么编译的时候，go编译器去什么位置找他们呢？

这里就要介绍下Go的环境变量了。Go有两个很重要的环境变量`GOROOT`和`GOPATH`。
这是两个定义路径的环境变量：
`GOROOT`是安装Go的路径，比如`/usr/local/go`；
`GOPATH`是我们自己定义的开发者个人的工作空间，比如`/home/flysnow/go`。

编译器会使用我们设置的==这两个路径==，再加上==`import`导入的相对全路径==来查找磁盘上的包，比如我们导入的`fmt`包，编译器最终找到的是`/usr/local/go/fmt`这个位置。

值得了解的是：对于包的查找，是有优先级的，编译器会优先在`GOROOT`里搜索，其次是`GOPATH`,一旦找到，就会马上停止搜索。如果最终都没找到，就报编译异常了。

#### 本地包的导入以及查找
本地内嵌包一般都是在安装go的时候，放在了`$GOROOT/src`下。
因此，import 导入本地包的时候，很容易查找到。

#### 远程包的导入以及查找

如果有的Go包共享在Github上，我们一样有办法使用他们，这就是远程导入包了。如下所示：
```go
import "github.com/spf13/cobra"
```

编译在导入它们的时候，会先在`GOPATH`下搜索这个包，如果没有找到，就会使用`go get`工具从版本控制系统（GitHub）获取，并且会把获取到的源代码存储在`GOPATH`目录下对应URL的目录里，以供编译使用。

注：`go get`工具可以**递归获取依赖包**；如果`github.com/spf13/cobra`也引用了其他的远程包，该工具可以一并下载下来。



### 导入多个包
Go文件中可以导入多个包，例如：
```go
import "fmt"
import "net/http"
import "os"

```

也可以使用括号进行**批量导入**，例如：
```go
import (
    "fmt"
    "net/http"
    "os"
)
```

### 导入包重命名
在使用`import`关键字导入包之后，我们就可以在代码中通过包名使用该包下相应的函数、接口等。如果我们导入的包名正好有重复的怎么办呢？针对这种情况，Go语言可以让我们对导入的包重新命名，这就是命名导入。

比如：
```go
package main

import (
	"fmt"
	myfmt "mylib/fmt"
)

func main() {
	fmt.Println()
	myfmt.Println()
}

```


### 省略导入

```go
import . "fmt"
```
省略导入可以将导入的包直接合并到当前程序，在使用该包时可以不用加包名前缀，直接使用。
```go
import . "fmt"

func main() {
    Println("Hello World！")
}

```

### 匿名导入
#### 背景
Go语言规定，导入的包必须要使用，否则会包编译错误，这是一个非常好的规则，因为这样可以避免我们引用很多无用的代码而导致的代码臃肿和程序的庞大。
因为很多时候，我们都不知道哪些包是否使用，这在C和Java上会经常遇到，有时候我们不得不借助工具来查找我们没有使用的文件、类型、方法和变量等，把它们清理掉。

#### 使用匿名导入
但是有时候，我们需要导入一个包，但是又不使用它，按照规则，这是不行的，为此Go语言给我们提供了一个空白标志符`_`,只需要我们使用`_`重命名我们导入的包就可以了。

如果导入一个包时为，在其导入的路径名称前设置`_`标识作为包名，这种包的导入方式就称为匿名导入。
比如：
```go
import _ "path/package"
```
比如：
```go
import _ "github.com/go-sql-driver/mysql"
```

一个包被匿名导入的目的主要是为了加载这个包，从而使得这个包中的资源得以初始化。被匿名引入的包中的`init`函数**将被执行并且仅执行一遍**。即使包没有 `init` 初始化函数，也不会引发编译器报错。


注：匿名导入的包与其他方式导入的包一样都会被编译到可执行文件中。

### 包的init函数

init函数通常用来做初始化变量、设置包或者其他需要在程序执行前的引导工作。

#### init 的 执行顺序
在Go文件中，可以定义`init`特殊函数。
(1) 当一个包被导入时，其中的每个`init`函数都会按照它们在源文件中的顺序被调用。
(2) 同一个go文件中的 `init`函数按照定义的顺序执行，一个包中的多个go文件都有init函数，则不同文件中的 `init`函数 执行顺序不定。
(3) 这些`init`函数会在包的变量初始化之后、`main`函数执行之前被调用。
(4) 一个包的初始化过程是按照导入顺序来进行初始化的，当前包声明的所有`init`函数会被串行调用并且每个`init`函数仅调用一次。包初始化时先执行依赖的包中声明的`init`函数后，再执行当前包中声明的`init`函数。确保在程序的`main`函数开始执行前，所有的依赖包都已初始化完成。


![](attachments/Pasted%20image%2020241216153825.png)


范例如下所示：
```go
package main

import "fmt"

var number int = 100

const pi = 3.1415926

func init() {
    fmt.Println(number)
    fmt.Println(pi)
    printHello()
}

func printHello() {
    fmt.Println("hello world")
}

func main() {
    fmt.Println("main")
}

// 执行结果
100
3.1415926
hello world
main 

```

#### 不能够主动调用`init`函数
`init`函数不接收任何参数且没有任何返回值，不能够主动调用`init`函数。


## 引用包(使用包)

对于多于一个路径的包名，在代码中引用的时候，使用全路径最后一个包名作为引用的包名，比如`net/http`,我们在代码使用的是`http`，而不是`net`。


# module(模块)

## 背景
在早期编写Go项目时，需要将Go项目代码依赖的所有第三方包放入`GOPATH`目录下，这是方式的缺点在于**不支持版本管理**，同一个依赖包只能存在一个版本的代码。而多个项目下可能会分别使用不同版本的依赖包。

## 介绍
相对于包是组织代码的基本单元，**模块则是管理依赖和版本控制的基本单元**，两者的目标是不同的。

具体来说，`Go module`将依赖**依赖的包版本信息**和**程序代码本身**实现分离管理，每个`Go module`都会有一个`go.mod`文件，该文件包含了`module`的**依赖包列表以及对应的版本信息**，当一个`module`需要引用其他依赖包时，会根据`go.mod`文件中的信息去下载对应的依赖包和对应版本的代码供程序使用。

## 发展历史
在Go1.11版本中，发布了`Go module`的依赖管理方式，可以帮助开发人员更方便地管理项目依赖，并且保证依赖的版本管理和代码的可复用性。Go module在Go1.14 版本开始推荐在生产环境使用，于Go1.16版本默认开启。

## 特点

（1）模块也是文件夹，包含一个或多个包。

（2）模块有个 go.mod 文件。
一个模块对应一个`go.mod`。对于多模块的项目（即单个模块存在多个模块）则是存在多个`go.mod`文件。

（3） 模块路径 (module path)：
模块路径通常是个仓库地址。一个模块可以包含一个或者多个包，它们可以共享同一个模块路径。

（4）每个模块都有一个明确的版本。
模块解决的问题是版本控制和包分发。

一个简单的、典型的 Go 项目的目录结构和文件可能像下边这样：
![](attachments/Pasted%20image%2020241216112138.png)

## 模块的获取
==模块由一系列已发布、版本化、被分发的包构成。可以直接从版本控制仓库或模块代理服务器下载模块==。




## go模块的版本
Go 的**模块版本**，依赖于版本控制仓库，可以使用 Git，也支持 Subversion, Mercurial, Bazaar, Fossil 等其他版本控制的形式。接下来主要介绍基于 Git 的版本发布。

比如我们要发布第一个版本 `v0.1.0`：
```bash
git commit -m "Init: v0.1.0"

git tag v0.1.0

git push origin v0.1.0
```

推送到远端仓库之后，我们可以通过 `go list` 指令查看模块的可用版本：
```bash
✗ go list -m git.woa.com/jasonzxpan/m1 
git.woa.com/jasonzxpan/m1 v0.1.0
```

### 语义化版本号（semantic version）
版本号包括：不兼容的 API 修改的**主版本号**、新增向下兼容特性的**次版本号**、新增向下兼容的补丁的**修订版本号**。

![](attachments/Pasted%20image%2020241216121605.png)

语义化版本的好处，是能让我们一眼就能看出哪些版本更新，哪些版本之间是兼容的。
比如以下版本之间，越往右版本越高（越新）。
```bash
1.0.0-alpha < 1.0.0 < 2.0.0 < 2.1.0 < 2.1.1
```

#### 预发布版本号
如果我们提供的包/模块，不是稳定的版本，但同时也需要打 tag 的话，就可以使用像 `v0.1.0-beta.1` 这样的预发布版本号。这种版本不必保证版本的稳定性。

**预发布版本号**，是通过横线在标准版本`major.minor.patch`的后面连接一个字符串，意思是还没有到达前边的标准版本。

## 使用第三方的包和模块
Go 语言的标准库（内嵌包）是随着 Go 版本发布的，因此在编写Go代码时，引入标准库的包（例如 `import "fmt"`）不需要指定具体的版本号。

第三方包和模块是指除了**模块自身中的包**与**标准库**之外的包和模块。

### 指定版本的依赖
我们上边创建的模块 `git.woa.com/jasonzxpan/m1`，假设其中有一个包 `p1`，其中有个导出函数 `M1P1F1()`。

我们再创建一个模块 `go mod init git.woa.com/jasonzxpan/m2`，要使用这个 `m1/p1` 的包，可以直接在文件中引入。

```go
package main

import (
    m1p1 "git.woa.com/jasonzxpan/m1/p1"
)

func main() {
    m1p1.M1P1F1()
}
```

**go mod tidy分析依赖**
这时候可以直接通过 `go mod tidy` 让其自己分析依赖，并且获得**最新版本**的包。
```bash
# go mod tidy
go: finding module for package git.woa.com/jasonzxpan/m1/p1
go: downloading git.woa.com/jasonzxpan/m1 v0.1.1
go: found git.woa.com/jasonzxpan/m1/p1 in git.woa.com/jasonzxpan/m1 v0.1.1
```

**go get获取指定版本的模块**
`go get` 可以让我们获得指定版本的模块：
```bash
go get git.woa.com/jasonzxpan/m1/p1@v0.1.0
```

**go get -u 更新模块**
如果要更新某个模块，也可以用 `go get -u` 来更新：
- 默认拉取最新的`go get -u example.com/mypackage`，但是要先删掉 go.mod 中对应的 require 之前指定的版本。
- 也可以直接指定 latest 版本，`go get -u example.com/mypackage@latest`


### 导入的多个包依赖同一个模块的版本冲突管理
#### MVS机制
对于版本冲突，Go Modules 使用了一种称为“最小版本选择”（Minimal Version Selection, MVS）的机制。MVS 机制确保在构建过程中使用的每个模块的版本是**所有依赖项中要求的最低版本**。

##### 范例

考虑下图中的示例。主模块需要版本 1.2 或更高版本的模块 A，以及版本 1.2 或更高版本的模块 B。A 1.2 和 B 1.2 分别需要 C 1.3 和 C 1.4。

C 1.3 和 C 1.4 都需要 D 1.2。所以根据 MVS 的机制，加粗黑框中的模块将会被选中。

![](attachments/Pasted%20image%2020241216145815.png)

这里你可能有疑问，C 的两个冲突版本 1.3 和 1.4 中最小的不是1.3吗？为什么选择1.4。MVS 不是要找依赖的最低版本，而是找**满足所有模块依赖的最低版本**，比如 B 1.2 如果使用到了 C 1.4 中的新增接口，那么 C 1.3 是不能满足 B 1.2 的需求的。所以，所有依赖项对 C 模块依赖的最低版本是 1.4。

B 和 D 的更高版本可用，但 MVS 不会选择它们，因为没有任何内容需要它们。

#### 冲突解决
如果发生依赖的冲突无法解决，可能会涉及到**升级依赖、降级依赖、替代、排除**等方法来解决。
![](attachments/Pasted%20image%2020241216150055.png)
![](attachments/Pasted%20image%2020241216150017.png)

![](attachments/Pasted%20image%2020241216150105.png)
![](attachments/Pasted%20image%2020241216150116.png)


### 第三方依赖的下载和存放
`Go Modules` 会将依赖下载到本地的模块缓存目录，默认情况下，这个目录是 `$GOPATH/pkg/mod`。每个依赖包都会带有版本号进行区分，这样就允许在本地存在同一个包的多个不同版本。
如果没有设置 `GOPATH` 环境变量，默认的 `GOPATH` 是用户主目录下的 `$HOME/go` 目录。
```go
mod
├── cache
├── cloud.google.com
├── github.com
    	└──google
          ├── uuid@v1.1.2
          ├── uuid@v1.3.0
          └── uuid@v1.3.1
...
```

如下所示，grpc 模块的存放位置。
![](attachments/Pasted%20image%2020241216144142.png)


### 第三方依赖的清理

如果缓存的模块太多导致磁盘占用太大，或者缓存的模块出现问题导致构建失败，这些情形下，我们可以调用 `go clean -modcache` 来**清理模块缓存**，这会删除 `$GOPATH/pkg/mod` 目录中的所有内容。

![](attachments/Pasted%20image%2020241216144258.png)

对比之下还有 `go clean` 指令，他只是用来清理项目自身构建的二进制文件、测试缓存文件、临时文件等。

## 模块路径（module path）
每个模块都有一个唯一的模块路径作为标识符。该路径在 `go.mod` 文件中通过`module` 指令 所声明。 声明，并与模块依赖信息放在一起。

如下所示：
![](attachments/Pasted%20image%2020241216143307.png)

### 模块路径的作用
Go 模块路径，也就是我们在调用 `go mod init` 的时候后边加的路径，通常是一个**仓库名**。

**模块路径的两个作用**：
- 模块路径可以唯一标识一个模块
- 模块路径可以指示下载地址


### 模块根目录
模块根目录是指包含 `go.mod` 文件的目录。

### 模块下包路径

![](attachments/Pasted%20image%2020241216171025.png)

模块下的每个包都是一系列同目录下、将被编译到一起的文件集合。
包路径是模块路径和包含包的子目录（相对于模块根目录的路径）拼起来的结果。比如，模块 `golang.org/x/net` 包含了目录 `html` 下的包。则这个包路径就是 `golang.org/x/net/html`。

### 模块的导入路径
模块的导入路径是一个全局唯一的标识符，它具有模块路径和版本两部分，比如 `github.com/my/project@v1.2.3`。
这里的@后面的部分就是版本号，这样设定的好处是可以在依赖管理中方便地指定需要的版本，这也就避免了不同版本间的冲突。

### 第三次模块的模块路径的组成
模块路径应该描述模块做什么以及在哪里找到它。
通常，模块路径由存储库根路径、存储库中的目录（通常为空）和主要版本后缀（仅适用于主要版本为 2 或更高）组成。

**(1) 存储库根路径**
存储库根路径是模块路径的一部分，对应开发模块的版本控制仓库的根目录。大多数模块都被定义在存储库根目录，因此这通常就是整个模块路径了。比如，`golang.org/x/net` 就是同名模块的存储库根路径。
如欲了解更多 go 命令如何通过模块路径并使用 HTTP 请求定位仓库的信息，请参阅 寻找模块存储库。

**(2) 存储库中的目录**
如果模块并没有定义于仓库根目录，则模块子目录是命名目录的模块路径的一部分，且不包括主版本后缀。这一规则也作用于语义化版本标签的前缀。比如，`golang.org/x/tools/gopls` 表示模块在存储库根路径` golang.org/x/tools` 的子目录`gopls`下，因此它具有模块子目录 `gopls`。参见 版本映射至提交 以及 同一仓库下的模块名称。

**(3) 主要版本后缀**
假设模块发布在版本 2 或更高，模块路径必须有像 /v2 这样的 主版本后缀。例如，路径为 `golang.org/x/repo/sub/v2` 的模块可以位于存储库 `golang.org/x/repo` 的 `/sub` 或 `/sub/v2` 子目录中。

如果某个模块需要被其他模块所依赖，则必须遵循这些规则，以便 go 命令可以查找和下载该模块。

### 仓库地址和模块路径不一样时
**如果仓库地址和模块路径不一样怎么办？**

#### 场景
比如我们之前的仓库地址是 `git.code.oa.com`, 之后迁移到了 `git.woa.com`，那很多之前很多代码仓库的 `module path` 都是以 `git.code.oa.com` 开头的。这时候如果直接 `go get git.woa.com/trpc/trpc-go`，可能会遇到跟下面的错误：
```text
# go get git.woa.com/trpc-go/trpc-go
go: git.woa.com/trpc-go/trpc-go@upgrade (v0.18.1) requires git.woa.com/trpc-go/trpc-go@v0.18.1: parsing go.mod:
  module declares its path as: git.code.oa.com/trpc-go/trpc-go
          but was required as: git.woa.com/trpc-go/trpc-go
```

如果直接使用老仓库地址，`go get git.code.oa.com/trpc-go/trpc-go` 可能老的服务已经停止了，无法获得对应的包。

#### go.mod中的replace解决
虽然仓库地址是 `github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc`，但是他模块路径自己写的是 `google.golang.org/grpc/cmd/protoc-gen-go-grpc`，这两者是不一致的，到时候会报错：

```text
go get github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc
go: downloading github.com/grpc/grpc-go v1.65.0
go: downloading github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc v1.4.0
go: github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc@upgrade (v1.4.0) requires github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc@v1.4.0: parsing go.mod:
  module declares its path as: google.golang.org/grpc/cmd/protoc-gen-go-grpc
          but was required as: github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc
```

你需要在你的`go.mod`文件中添加一个 `replace` 指令，将 `github.com` 模块名替换为`google.golang.org` 开头的模块名。例如：
```go
replace github.com/grpc/grpc-go/cmd/protoc-gen-go-grpc => google.golang.org/grpc/cmd/protoc-gen-go-grpc v1.4.0
```

#### GOPROXY解决
问：为什么 `goproxy.woa.com` 能去 git 下载到 `import path` 为 `git.code.oa.com` 的库？

答：是因为 `goproxy.woa.com` 服务 `git clone` 时会自动将 `git.code.oa.com` 替换为 `git.woa.com`

## 将引入包解析为模块
当 `go` 命令使用包路径加载包时(即：解析go文件的 import 命令时)，它需要确定哪个模块提供包。
go 命令从搜索[构建列表](https://go.dev/ref/mod#glos-build-list) (build list: 即 go.mod 中模块的版本列表）开始，搜索具有包路径前缀的模块。
如果构建列表中只有一个模块提供包，则使用该模块。如果没有模块提供包，或者有多个模块提供包，则 `go` 命令会报错。
例如，导入了包 `example.com/a/b`，并且模块 `example.com/a` 在构建列表中，那么 go 命令将检查 `example.com/a` 目录中是否包含需要导入的`包b`。在 `b` 目录中必须至少存在一个扩展名为 `.go` 的文件，才能将其视为包。

`-mod=mod` 标志表示 `go` 命令尝试查找缺失的包模块，并更新 `go.mod` 和 `go.sum`。`go get` 命令和 `go mod tidy`命令会自动触发此操作。



## 项目中使用go module的流程
### 流程
**（1）设置`GO111MODULE`环境变量**：
在创建项目后，如果需要使用`go module`，则需要将`go env`中的`GO111MODULE`环境变量设置成`on`，打开`go module`模式。

**（2）生成go.mod文件**：
到项目目录下，执行`go mod init`初始化生成 `go.mod` 文件。

**（3）go get下载第三方依赖**
如果需要导入指定版本的第三方依赖包，则可以使用`go get`命令进行下载。
执行`go get` 命令，在下载依赖包的同时还可以指定依赖包的版本。
- `go get -u`命令会将项目中的依赖包升级到最新的次要版本或者修订版本；
- `go get -u=patch`命令会将项目中的依赖包包升级到最新的修订版本；
- `go get [包名]@[版本号]`命令会下载对应依赖包的指定版本或者将对应包升级到指定的版本，例如`go get foo@v1.2.3`。

注：`go module` 安装依赖包时先拉取最新的 `release tag`，若无 `tag` 则拉取最新的 `commit`，且go项目会自动生成一个 `go.sum` 文件来记录 `dependency tree`。`go module` 引入了`go.sum`机制来对依赖包进行校验。


### 范例
在本地新建一个名为`testProject`的项目目录，并切换到该目录下：
```go
$ mkdir testProject
$ cd testProject
```

在`testProject`项目目录下，初始化创建一个`go.mod`文件。
```go
$ go mod init testProject
go: creating new go.mod: module testProject
```

使用该命令后自动会在项目目录下创建一个`go.mod`文件，内容如下：
```go
module testProject

go 1.19
```

- `module testProject`：表示当前项目的导入路径；
- `go 1.19`：表示当前项目使用的Go版本；


`go.mod`文件会记录项目中使用的第三方依赖包信息，包括包名和版本，但由于目前项目并未使用到任何第三方包，因此`go.mod`文件暂时还没有记录任何依赖包信息。

可以在项目的目录下创建一个`main.go`文件，然后使用`go get`命令手动下载一个依赖包来测试一下：
```go
$ go get -u github.com/labstack/echo
go: added github.com/labstack/echo v3.3.10+incompatible
go: added github.com/labstack/gommon v0.4.1
go: added github.com/mattn/go-colorable v0.1.13
go: added github.com/mattn/go-isatty v0.0.20
go: added github.com/valyala/bytebufferpool v1.0.0
go: added github.com/valyala/fasttemplate v1.2.2
go: added golang.org/x/crypto v0.16.0
go: added golang.org/x/net v0.19.0
go: added golang.org/x/sys v0.15.0
go: added golang.org/x/text v0.14.0

```

可以看到调用`go get`命令后，将依赖添加到项目中，前缀 `go: added` 表示成功添加了一个新的依赖包。此时go.mod中的内容：
```go
module testProject

go 1.19

require (
    github.com/labstack/echo v3.3.10+incompatible // indirect
    github.com/labstack/gommon v0.4.1 // indirect
    github.com/mattn/go-colorable v0.1.13 // indirect
    github.com/mattn/go-isatty v0.0.20 // indirect
    github.com/valyala/bytebufferpool v1.0.0 // indirect
    github.com/valyala/fasttemplate v1.2.2 // indirect
    golang.org/x/crypto v0.16.0 // indirect
    golang.org/x/net v0.19.0 // indirect
    golang.org/x/sys v0.15.0 // indirect
    golang.org/x/text v0.14.0 // indirect
)

```

在导入了指定的依赖包后，此时可以在main.go文件中使用导入的依赖包，例如：

```go
package main

import (
    "github.com/labstack/echo"
    "net/http"
)

func main() {
    e := echo.New()
    e.GET("/", func(c echo.Context) error {
       return c.String(http.StatusOK, "Hello, World!")
    })
    e.Logger.Fatal(e.Start(":8080"))
}

```


## 多模块的项目
比如  `grpc-go` 的仓库中都有多个 `go.mod ` 文件，这就意味着这些仓库中有多个模块。

![](attachments/Pasted%20image%2020241216115304.png)

我们以 grpc-go 仓库为例：

- grpc-go 是 gRPC Go 项目的根模块，Go版本的gRPC 实现都在这模块中
- grpc-go/cmd/protoc-gen-go-grpc 是使用 gRPC Go 时，要通过 `protoc` 将 .proto文件转换成Go语言库，其具体使用的工具就是这个 `protoc-gen-go-grpc`。该模块本身只有根目录那一个包。
- grpc-go/examples 是 gRPC Go 的示例模块，其中有三个子包

### 为什么要多模块
#### 优点
单项目，多模块这样划分，客观上还带来了一些好处：
##### 依赖管理是独立的
这里的 examples 依赖等不会影响到外层 grpc-go 模块的依赖；而examples 中的依赖，也可以依赖 grpc-go 中没有的或者版本不一样的模块

##### 版本管理是独立的
这里examples直接没有打版本的需要，我们看另外一个 protoc-gen-go-grpc 的版本情况。
我们可以通过 `go list -m -versions` 来看两个模块的版本信息，可以看到这个工具只有到6个版本，而根模块有到 v1.66.0-dev 等几十个版本：

![](attachments/Pasted%20image%2020241216120519.png)

#### 多模块的适用场景
**多个模块有关联性，但是又不那么强烈的耦合**的场景，就适合拆分成多个模块。
比如 `grpc-go/examples` 模块，只是 `grpc-go` 的示例，单独放个仓库过于简单，没有必要。而如果直接放在 `grpc-go` 模块中作为一个包，又会对 `grpc-go` 包的纯洁性有影响——用户不会依赖 `examples` 包。

### 多模块中如何对不同的模块打不同的版本
在官方文档 [Managing module source](https://go.dev/doc/modules/managing-source) 中有介绍。==根目录直接打 `v1.2.1`，子目录打标签时候带上目录即可==。
比如我们看到 `protoc-gen-go-grpc` 对应的版本在仓库中有打对应的 tag 如 `cmd/protoc-gen-go-grpc/v1.1.0`：

![](attachments/Pasted%20image%2020241216120814.png)

可以上下层次，也可以两个目录并行，比如官方介绍的：

![](attachments/Pasted%20image%2020241216120839.png)


## go.sum文件
在官方文档 [管理依赖](https://go.dev/doc/modules/managing-dependencies#adding_dependency) 一文中，有这样一段描述：
```text
When you add dependencies, Go tools also create a go.sum file that contains checksums of modules you depend on. Go uses this to verify the integrity of downloaded module files, especially for other developers working on your project.

Include the go.mod and go.sum files in your repository with your code.
```
当添加依赖（创建 `go.mod`）的同时，也会创建 `go.sum` 文件。`go.sum` 记录了项目中使用的每个依赖模块的模块路径（`module path`）以及对应的校验和，用于验证依赖模块在下载时的完整性。`go get` 和 `go mod tidy` 的时候，在更新` go.mod` 的同时，也会更新 `go.sum`。

`go.mod` 文件定义了项目的模块依赖关系和版本约束；
而 `go.sum` 文件则提供了依赖模块的安全性保证，确保每次构建时都能使用正确的依赖版本，可以防止恶意或不良网络条件下的**中间人攻击**。

**go.sum 和 go.mod 一样，都要加入到版本控制中。**


# project(项目工程)和module(模块)和package(包)的关系
## 背景
软件是由代码组成的。为了复用代码，代码的组织出现了不同层次的抽象和实现，如 Module（模块），包（Package），Lib（库），Framwork（框架）等。

## module(模块)和package(包)的关系
**在Go语言中，包（package）是代码组织和重用的基本单位，而模块（module）是控制版本和管理依赖的单元**。

```bash
A package is a collection of source files in the same directory that are compiled together.

A module is a collection of related Go packages that are released together.
```

**包**（Package）是Go语言中的基础组织单位，它用于组成代码结构和解决命名隔离问题。这是由一系列相关的go代码文件构成的，这些文件在文件系统上存在同一个目录下。

而**模块**（Module）则是一个集合了许多包的更大单位，模块不仅是==版本控制的管理单位==，而且还是==包的分发单位==。
模块中可以包含一个或者多个包，它们可以共享同一个模块路径。并且每个模块都有一个明确的版本。


### 包是代码组织的基础单元

包是代码组织的基础单元，用于解决命名隔离的问题，确定代码的边界，以及复用代码等问题。包的目标是内聚，对于外部隐藏细节，对外只提供接口。

在Go语言中，一个包就是一个目录，里面包含了一系列`.go`源代码文件。 Go语言的编译器会从包的构造点开始，追踪分析程序的所有依赖，并一次性完成编译任务。这样可以大大提高编译效率。

对于Go语言的包来说，它们包含了导入路径、包声明以及包别名等内容。其中，导入路径是一个独特很重要的部分，它告诉了Go编译器在哪里可以找到这个包。程序员可以通过导入包的路径来使用包中的公开函数、类型或变量。


### Go语言中的模块是包的更大的组织及管理单位

模块是一组相关的包的集合，一个模块可以包含一个或者多个包，并且每个模块都有一个明确的版本。模块的设计目标是让开发者能够更好地追踪版本、处理依赖关系、分发他们的工作。

Go模块通过`go.mod`文件来定义模块的依赖关系，并且每个模块都有一个唯一的模块路径作为标识符。通过模块，我们可以更好地管理项目中的依赖关系，确保版本的兼容性，以及方便地共享和重用模块。

模块的导入路径是一个全局唯一的标识符，它具有模块路径和版本两部分，比如 `github.com/my/project@v1.2.3`。这里的@后面的部分就是版本号，这样设定的好处是可以在依赖管理中方便地指定需要的版本，这也就避免了不同版本间的冲突。

## 范例

### CASE1：单个本地module，包含子package
下面是我`go mod init HAHA`自己生成的内容：
```go
$ cat go.mod
module HAHA  //这里HAHA是我的module的名字

go 1.22.3
```

```go
$ tree
.
├── go.mod
├── main.go
└── mymod1
    └── test1.go

1 directory, 3 files

$ cat main.go
package main

import (
        "HAHA/mymod1"
        "fmt"
)

func main() {
        fmt.Printf("Hello, world.\n")
        mymod1.Do_mymod1()
}

$ cat mymod1/test1.go
package mymod1

import (
        "fmt"
)

func Do_mymod1() {
        fmt.Printf("I'm mymod1\n")
}

$ go build main.go //此时会生成二进制可执行文件main
$ ./main //执行该文件
Hello, world.
I'm mymod1

```

**注意**：我的环境是go1.16，**我用相对目录"./mymod1"就提示找不到包**。
```bash
[root@7a6603fc4c13 go_test]# go build main.go
main.go:5:9: "./mymod1" is relative, but relative import paths are not supported in module mode
```


### CASE2：使用网络上的module
上面的情况是在同一个module内使用另一个package, 我们也可以从网上导入一个module来使用。
 go mod tidy 会自动查找依赖，自动下载。这些内容会被安装在GOPATH/pkg/mod目录下，并且go.mod也会记录下它们的版本号。
 
```go
package main

import (
        "HAHA/mymod1"
        "fmt"
	"github.com/labstack/echo"
)

func main() {
        fmt.Printf("Hello, world.\n")
        mymod1.Do_mymod1()
	e := echo.New()
	e.GET("/", func(c echo.Context) error {
		return c.String(http.StatusOK, "Hello, World!")
	})
	e.Logger.Fatal(e.Start(":1323"))
}
```

```go
(1) go mod tidy 之前的 go.mod
# cat go.mod
module HAHA

go 1.22.3

(2) 执行 go mod tidy
# go mod tidy
go: finding module for package github.com/labstack/echo
go: downloading github.com/labstack/echo v3.3.10+incompatible
go: found github.com/labstack/echo in github.com/labstack/echo v3.3.10+incompatible
go: finding module for package github.com/labstack/gommon/color
go: finding module for package github.com/labstack/gommon/log
go: finding module for package golang.org/x/crypto/acme/autocert
go: finding module for package github.com/stretchr/testify/assert
go: downloading golang.org/x/crypto v0.31.0
go: downloading github.com/stretchr/testify v1.10.0
go: downloading github.com/labstack/gommon v0.4.2
go: found github.com/labstack/gommon/color in github.com/labstack/gommon v0.4.2
go: found github.com/labstack/gommon/log in github.com/labstack/gommon v0.4.2
go: found golang.org/x/crypto/acme/autocert in golang.org/x/crypto v0.31.0
go: found github.com/stretchr/testify/assert in github.com/stretchr/testify v1.10.0
go: downloading github.com/mattn/go-colorable v0.1.13
go: downloading github.com/mattn/go-isatty v0.0.20
go: downloading github.com/valyala/fasttemplate v1.2.2
go: downloading github.com/davecgh/go-spew v1.1.1
go: downloading github.com/pmezard/go-difflib v1.0.0
go: downloading golang.org/x/net v0.21.0
go: downloading gopkg.in/yaml.v3 v3.0.1
go: downloading golang.org/x/sys v0.28.0
go: downloading github.com/valyala/bytebufferpool v1.0.0
go: downloading golang.org/x/text v0.21.0


(3) go mod tidy 之后的 go.mod
# cat go.mod
module HAHA

go 1.22.3

require github.com/labstack/echo v3.3.10+incompatible

require (
	github.com/labstack/gommon v0.4.2 // indirect
	github.com/mattn/go-colorable v0.1.13 // indirect
	github.com/mattn/go-isatty v0.0.20 // indirect
	github.com/stretchr/testify v1.10.0 // indirect
	github.com/valyala/bytebufferpool v1.0.0 // indirect
	github.com/valyala/fasttemplate v1.2.2 // indirect
	golang.org/x/crypto v0.31.0 // indirect
	golang.org/x/net v0.21.0 // indirect
	golang.org/x/sys v0.28.0 // indirect
	golang.org/x/text v0.21.0 // indirect
)



(4) 查看 /GOROOT/pkg/mod目录
# ll /root/go/pkg/mod/
total 0
drwxr-xr-x 3 root root 34 Dec 12 19:37 cache
drwxr-xr-x 8 root root 96 Dec 12 19:37 github.com
drwxr-xr-x 3 root root 15 Dec 12 19:37 golang.org
drwxr-xr-x 3 root root 28 Dec 12 19:37 gopkg.in

# ll /root/go/pkg/mod/github.com/labstack/
total 4
dr-xr-xr-x 5 root root 4096 Dec 12 19:37 echo@v3.3.10+incompatible
dr-xr-xr-x 8 root root  196 Dec 12 19:37 gommon@v0.4.2

(5) go mod tidy 后自动生成/更新的 go.sum 
# cat go.sum
github.com/davecgh/go-spew v1.1.1 h1:vj9j/u1bqnvCEfJOwUhtlOARqs3+rkHYY13jYWTU97c=
github.com/davecgh/go-spew v1.1.1/go.mod h1:J7Y8YcW2NihsgmVo/mv3lAwl/skON4iLHjSsI+c5H38=
github.com/labstack/echo v3.3.10+incompatible h1:pGRcYk231ExFAyoAjAfD85kQzRJCRI8bbnE7CX5OEgg=
github.com/labstack/echo v3.3.10+incompatible/go.mod h1:0INS7j/VjnFxD4E2wkz67b8cVwCLbBmJyDaka6Cmk1s=
github.com/labstack/gommon v0.4.2 h1:F8qTUNXgG1+6WQmqoUWnz8WiEU60mXVVw0P4ht1WRA0=
github.com/labstack/gommon v0.4.2/go.mod h1:QlUFxVM+SNXhDL/Z7YhocGIBYOiwB0mXm1+1bAPHPyU=
github.com/mattn/go-colorable v0.1.13 h1:fFA4WZxdEF4tXPZVKMLwD8oUnCTTo08duU7wxecdEvA=
github.com/mattn/go-colorable v0.1.13/go.mod h1:7S9/ev0klgBDR4GtXTXX8a3vIGJpMovkB8vQcUbaXHg=
github.com/mattn/go-isatty v0.0.16/go.mod h1:kYGgaQfpe5nmfYZH+SKPsOc2e4SrIfOl2e/yFXSvRLM=
github.com/mattn/go-isatty v0.0.20 h1:xfD0iDuEKnDkl03q4limB+vH+GxLEtL/jb4xVJSWWEY=
github.com/mattn/go-isatty v0.0.20/go.mod h1:W+V8PltTTMOvKvAeJH7IuucS94S2C6jfK/D7dTCTo3Y=
github.com/pmezard/go-difflib v1.0.0 h1:4DBwDE0NGyQoBHbLQYPwSUPoCMWR5BEzIk/f1lZbAQM=
github.com/pmezard/go-difflib v1.0.0/go.mod h1:iKH77koFhYxTK1pcRnkKkqfTogsbg7gZNVY4sRDYZ/4=
github.com/stretchr/testify v1.10.0 h1:Xv5erBjTwe/5IxqUQTdXv5kgmIvbHo3QQyRwhJsOfJA=
github.com/stretchr/testify v1.10.0/go.mod h1:r2ic/lqez/lEtzL7wO/rwa5dbSLXVDPFyf8C91i36aY=
github.com/valyala/bytebufferpool v1.0.0 h1:GqA5TC/0021Y/b9FG4Oi9Mr3q7XYx6KllzawFIhcdPw=
github.com/valyala/bytebufferpool v1.0.0/go.mod h1:6bBcMArwyJ5K/AmCkWv1jt77kVWyCJ6HpOuEn7z0Csc=
github.com/valyala/fasttemplate v1.2.2 h1:lxLXG0uE3Qnshl9QyaK6XJxMXlQZELvChBOCmQD0Loo=
github.com/valyala/fasttemplate v1.2.2/go.mod h1:KHLXt3tVN2HBp8eijSv/kGJopbvo7S+qRAEEKiv+SiQ=
golang.org/x/crypto v0.31.0 h1:ihbySMvVjLAeSH1IbfcRTkD/iNscyz8rGzjF/E5hV6U=
golang.org/x/crypto v0.31.0/go.mod h1:kDsLvtWBEx7MV9tJOj9bnXsPbxwJQ6csT/x4KIN4Ssk=
golang.org/x/net v0.21.0 h1:AQyQV4dYCvJ7vGmJyKki9+PBdyvhkSd8EIx/qb0AYv4=
golang.org/x/net v0.21.0/go.mod h1:bIjVDfnllIU7BJ2DNgfnXvpSvtn8VRwhlsaeUTyUS44=
golang.org/x/sys v0.0.0-20220811171246-fbc7d0a398ab/go.mod h1:oPkhp1MJrh7nUepCBck5+mAzfO9JrbApNNgaTdGDITg=
golang.org/x/sys v0.6.0/go.mod h1:oPkhp1MJrh7nUepCBck5+mAzfO9JrbApNNgaTdGDITg=
golang.org/x/sys v0.28.0 h1:Fksou7UEQUWlKvIdsqzJmUmCX3cZuD2+P3XyyzwMhlA=
golang.org/x/sys v0.28.0/go.mod h1:/VUhepiaJMQUp4+oa/7Zr1D23ma6VTLIYjOOTFZPUcA=
golang.org/x/text v0.21.0 h1:zyQAAkrwaneQ066sspRyJaG9VNi/YJ1NfzcGB3hZ/qo=
golang.org/x/text v0.21.0/go.mod h1:4IBbMaMmOPCJ8SecivzSH54+73PCFmPWxNTLm+vZkEQ=
gopkg.in/yaml.v3 v3.0.1 h1:fxVm/GzAzEWqLHuvctI91KS9hhNmmWOoWu0XTYJS7CA=
gopkg.in/yaml.v3 v3.0.1/go.mod h1:K4uyk7z7BCEPqu6E+C64Yfv1cQ7kz7rIZviUmN+EgEM=
```

### CASE3：使用本地的module
如果我本地的网络不好，或者某些公司内部无法联网，那怎么引用别的module呢。其实也可以预先下载到本地然后引用的。

代码的，目录结构如下所示：
```go
$ tree
.
├── README.md
├── go.mod
├── hello
│   ├── go.mod
│   └── hello.go
├── main.go
└── utils
    ├── addAndGreet.go
    └── go.mod
```
可以看到这里有三个go.mod, 表示三个module。最外层的module会引用后面两个。

main.go 的内容如下：
```go
package main

import (
	"example.org/hello"
	"example.org/utils"
	"fmt"
)

func main() {
	fmt.Println(hello.Hello("martin"))
	fmt.Println(utils.AddAndGreet("martin", 2, 3))
}
```

如何才能让`go`在本地查找这两个模块呢，关键是在`go.mod`中使用`replace`关键字告诉 `go`编译器从本地查找这两个模块。
`go mod init example.com/localmodexample` 先生成一个`go.mod`文件，然后更改`go.mod`，更改之后的最外层的`go.mod`内容如下：
```go
module example.com/localmodexample

go 1.22.3

require (
	example.org/hello v0.0.0
	example.org/utils v0.0.0

)

replace (
	example.org/hello => ./hello
	example.org/utils => ./utils
)
```


```go
# utils下的 addAndGreet.go的内容
package utils

import (
	"example.org/hello"
	"strconv"
)

func AddAndGreet(name string, a, b int) string {
	return hello.Hello(name) + " " + strconv.Itoa(a + b)
}

# utils 的 go.mod的内容
module example.org/utils

==============

# hello下的 hello.go的内容
package hello

func Hello(name string) string {
	return "hello " + name
}

# hello下的go.mod的内容
module example.org/hello
```

### CASE4-失败的例子：把本地的子package改成module

有了上面的`go.mod`中的 `replace`的使用例子，我想着把上面“CASE1”里mymod1改成module岂不是很简单，我直接在import的时候用相对路径引入，然后在go.mod里添加对mymod1的require不就行了吗，replace操作我手动处理好了， 应该不用replace这个字段了。结果很打脸，就是不行的。

```go
$ tree
.
├── go.mod
├── main.go
└── mymod1
    ├── go.mod
    └── test1.go

1 directory, 4 files

$ cat go.mod
module HAHA

go 1.22.3

require (
        mymod1 v0.0.0
)

$ cat main.go
package main

import (
        //"HAHA/mymod1"
        "./mymod1"
        "fmt"
)

func main() {
        fmt.Printf("Hello, world.\n")
        mymod1.Do_mymod1()
}

$ go build main.go  ######啊啊啊啊报错了#####
go: mymod1@v0.0.0: missing go.sum entry; to add it:
        go mod download mymod1
```
我把`replace`字段理解成C语言里的宏预处理了，以为它会对`import`关键字后面的内容进行简单的字符替换，实际上不是这样的。
==把`go.mod`中的`replace`字段理解成GCC里的编译选项`-L`更合适，也就是指明库的位置==。 
当然这个位置可以是当前目录下，也可以是别的任意位置。

### CASE5-成功的例子：把本地的子package改成module

按照`CASE3`里的思路重新改写`CASE1`的代码：
```go
$ tree
.
├── go.mod
├── main.go
└── mymod1
    ├── go.mod
    └── mymod1.go

1 directory, 4 files

$ cat go.mod
module HAHA

go 1.22.3

require (
        mymod1 v0.0.0
)

replace (
        mymod1 => ./mymod1
)

$ cat main.go
package main

import (
        //"HAHA/mymod1"
        "mymod1"
        "fmt"
)

func main() {
        fmt.Printf("Hello, world.\n")
        mymod1.Do_mymod1()
}


# cat mymod1/go.mod
module mymod1

go 1.22.3


# cat mymod1/test1.go
package mymod1

import (
        "fmt"
)

func Do_mymod1() {
        fmt.Printf("I'm mymod1\n")
}


# 测试如下：
$ ./main
Hello, world.
I'm mymod1

```

# golang包管理的发展历程
## GOPATH
## govendor
## go mod
## go list
```bash
go list -m all 
查看当前项目正在使用的 package 版本
然后执行 `go get xxx/xxx` 来更新指定的 package
```
# go module 详解

## 相关env配置
### `GO111MODULE` 环境变量
在环境变量中，`GO111MODULE`变量作为`Go modules`的开关，其允许设置如下参数：

- `auto`：项目若包含`go.mod`文件，则启用`Go modules`；
- `on`：启用`Go modules`，go命令行会使用`modules`，而不会去`GOPATH`目录下查找依赖包，推荐设置；
- `off`：禁用`Go modules`，寻找依赖包的方式沿用旧版本通过`vendor`目录或者`GOPATH`模式来查找，不推荐设置；



  如果需要对`GO111MODULE`环境变量的值进行变更，则可以使用如下命令：
```go
go env -w GO111MODULE=on
```

### `GOPROXY`
`GOPROXY`环境变量主要用于设置Go模块代理（`Go module proxy`），使Go在后续拉取依赖时能够**脱离传统的`VCS`(版本控制系统, 比如 git)方式，直接通过镜像站点快速拉取**。

#### 国内的代理点
`GOPROXY` 的默认值是：`https://proxy.golang.org,direct`，该站点在国内无法正常访问，因此在开启`Go modules`时，需要设置国内的Go模块代理。目前社区使用比较多的有两个`https://goproxy.cn`和`https://goproxy.io`。

执行命令如下：
```go
go env -w GOPROXY=https://goproxy.cn,direct
```

#### 多个代理地址
`GOPROXY`允许设置多个模块代理地址，多个地址之间以英文逗号“,”分隔开，当配置多个代理地址时，值代理地址列表中上一个Go模块代理地址返回404 或 410 错误时，Go 会自动尝试下一个模块代理地址，当遇到 `direct`时会回到源地址去抓取，当遇到`EOF`时中止并抛出错误。

若不想使用，则可以设置为“off”，这将会禁止Go在后续操作中使用任何的Go模块代理。

#### direct 
`direct`是一个特殊指示符，指示 `go` 命令从开发模块的版本控制存储库下载模块，而不是使用代理。

### `GOSUMDB`
`GOSUMDB`环境变量（Go checksum database）在拉取依赖版本时（无论是在源站点还是Go module proxy）保证拉取的模块版本数据未经过篡改，若发现不一致，即可能存在篡改，则会立即中止拉取。

`GOSUMDB`的默认值为`sum.golang.org`，该站点在国内无法访问，不过可以通过之前设置的`GOPROXY`解决，`GOPROXY`设置的代理`goproxy.cn`同样支持代理`sum.golang.org`。

若该值设置为`off`，则禁止GO在后续操作中校验模块版本。

### `GOMODCACHE` 环境变量
在 Go 1.15，可以通过 GOMODCACHE 环境变量设置模块缓存的位置。
即存储go下载的外部依赖模块文件的目录，默认值为`$GOPATH/pkg/mod`。
因此我们一般只需要更改`GOPATH`的值即可，此环境变量的值就会自动做出相应的变动。当然你也可以设置为其他值。

## go mod相关的命令

![](attachments/Pasted%20image%2020241212164211.png)

```text
download    download modules to local cache
edit        edit go.mod from tools or scripts
graph       print module requirement graph
init        initialize new module in current directory
tidy        add missing and remove unused modules
vendor      make vendored copy of dependencies
verify      verify dependencies have expected content
why         explain why packages or modules are needed
```

### go mod init
一般是在首次创建项目时使用，初始化项目依赖，生成`go.mod`文件.

```go
# go help mod init
usage: go mod init [module-path]
```

### go mod tidy
检查项目中的依赖关系：将项目文件中引入的依赖与`go.mod`进行比对。
添加缺少的模块并删除未使用的模块，一般用来更新 go.mod 和 go.sum 文件。

```go
# go help mod init
usage: go mod init [module-path]
```


### go mod edit
编辑`go.mod`文件

```go
# go help mod edit
usage: go mod edit [editing flags] [-fmt|-print|-json] [go.mod]
```

### go mod download
`go mod download`下载依赖的模块到本地 cache。
```go
# go help mod download
usage: go mod download [-x] [-json] [-reuse=old.json] [modules]
```

### go get 
通过 go get 命令可以添加依赖：将依赖项添加到 go.mod 文件，并将依赖项的版本信息记录在 go.sum 文件中。

```go
# go help get
usage: go get [-t] [-u] [-v] [build flags] [packages]
```

### go list 命令
#### 列出依赖的模块
通过 go list 命令可以查看项目的依赖，其中 -m 选项表示列出模块而不是包。
```go
go list -m all
```

#### 查看可升级的依赖
go list 的 -u 选项将在依赖的模块后面通过中括号显示可用的最新版本（如果有的话）。
```go
go list -m -u all

my/main/module
golang.org/x/text v0.3.0 [v0.4.0] => /tmp/text
rsc.io/pdf v0.1.1 (retracted) [v0.1.2]
```

### 清理模块缓存
理模块缓存表示删除存储在本地已下载的模块文件。模块缓存文件存放在 `GOPATH/pkg/mod` 目录。
```go
go clean -modcache
```

# go.mod文件
`go.mod`文件是Go语言工具链用于管理Go语言项目的一个配置文件，我们不用手动修改它，Go语言的工具链会帮我们自动更新。

## module指令
模块指令定义模块的路径。一个 `go.mod` 文件必须只包含一个`模块`指令。

```go
module golang.org/x/net
```

## go指令

`go` 指令表示一个模块是按照给定的 Go 版本的语义来编写的。
一个 `go.mod` 文件最多可以包含一个 `go` 指令。如果没有，大多数命令会添加一个当前 Go 版本的 `go` 指令。
版本必须是有效的 Go 发布版本：一个正整数，后面跟着一个点和一个非负整数（例如，`1.9`，`1.14`）。

### 作用
go 指令最初是为了支持 Go 语言的向后不兼容的变化（详情见 Go 2 transition）。
自从引入模块以来，没有任何不兼容的语言变化，但 go 指令仍然影响到新语言特性的使用：
对于模块内的包，编译器会拒绝使用 go 指令指定的版本之后引入的语言特性。例如，如果一个模块的指令是 go 1.12，它的包就不能使用 1_000_000 这样的数字字面，这是在 Go 1.13 中引入的。


## require指令
### 介绍
一个 `require` 指令声明了一个特定模块依赖的最低版本要求。

### 格式
```go
module testProject

go 1.19

require (
    github.com/labstack/echo v3.3.10+incompatible // indirect
    github.com/labstack/gommon v0.4.1 // indirect
    github.com/mattn/go-colorable v0.1.13 // indirect
    github.com/mattn/go-isatty v0.0.20 // indirect
    github.com/valyala/bytebufferpool v1.0.0 // indirect
    github.com/valyala/fasttemplate v1.2.2 // indirect
    golang.org/x/crypto v0.16.0 // indirect
    golang.org/x/net v0.19.0 // indirect
    golang.org/x/sys v0.15.0 // indirect
    golang.org/x/text v0.14.0 // indirect
)
```
从上述`go.mod`文件中记录的当前项目中所有依赖包的相关信息可知，声明依赖的格式如下：
```go
require module/path v1.2.3
```
- `require`：声明依赖包的关键字；
- `module/path`：依赖包的导入路径；
- `v1.2.3`：依赖包的版本号，支持`latest`最新版本、详细版本`v1.2.3`、指定某次`commit hash`这几种格式。

### 间接依赖

如果我们项目复杂点，打开 go.mod 文件，会看到一些信息。 除了 module_path、Go 版本以及直接依赖的模块和版本信息之外，还有另外一个**间接依赖**。与直接依赖类似，也是模块路径和版本号，区别在于最后有个 `// indirect` 的注释。

![](attachments/Pasted%20image%2020241216143413.png)

#### 间接依赖是什么
间接依赖是指：如果我们的模块直接使用 `moduleA` 中的包，而 `moduleA` 依赖 `moduleB`，那么 `moduleB` 就是我们模块的**间接依赖**。

#### go.mod什么时候出现间接依赖
是不是 go.mod 会将所有间接依赖都列出来？答案是**否定的**。以上边的依赖关系来说，只有 moduleA 的 go.mod 文件中，没有指定 moduleB 的版本，才会在我们的自己的模块 go.mod 文件中指定间接依赖的版本。如果 moduleA 已经指定了 moduleB 的版本，就不需要指定了。

#### 更新go.mod中的间接依赖
运行 `go mod tidy` 命令来更新你的`go.mod`文件，这个命令会添**加缺失的模块，删除无用的模块，以及更新间接依赖的版本**。

#### 为什么需要指定间接依赖的版本
**记录间接依赖的版本信息是为了确保项目的构建稳定性和可重现性**。
因为间接依赖的版本可能会影响**直接依赖模块**的行为和版本兼容性。
只有记录清楚所有依赖的版本，才能够重现完全相同的依赖版本树，从而避免由于间接依赖的不一致而导致的问题。

## replace指令
### 场景
有些场景，我们需要替换掉依赖远端的第三方库。
比如：第三方库需要增加一些调试信息，或者增加一些特性。

### 操作流程
可以从源仓库 fork第三发模块，然后在本地开发（也可以开发完成后push到自己的仓库）

大概的操作流程都类似：
- 将第三方模块，clone 到本地
- 开发过程、修改第三方模块
- 通过 replace 的方式，修改 go.mod 中的依赖，替换成本地目录


### replace 使用本地的 模块

以上边 m2 依赖 m1 模块为例，我们可以直接通过 `replace`，在开发过程中使用本地的 `m1`：
```go
go mod edit -replace=git.woa.com/jasonzxpan/m1@v0.1.1=../m1
```

### replace 替换为远程的模块
```go
go mod edit -replace=git.code.oa.com/trpc-go/trpc-go=git.woa.com/trpc-go/trpc-go@v0.15.0
```

## exclude指令
一个 `exclude` 指令可以阻止一个模块的版本被 `go` 命令加载。

从 Go 1.16 开始，如果任何 `go.mod` 文件中的 `require` 指令所引用的版本被主模块的 `go.mod` 文件中的 `exclude` 指令所排除，则该需求被忽略。

## retract 指令
`retract` 指令表示由 `go.mod` 定义的模块的某个版本或一系列版本不应该被依赖。当一个版本过早发布或在发布后发现严重问题时，`retract` 指令很有用。
被撤回的版本应该在版本控制库和模块代理中保持可用，以确保依赖它们的构建不会被破坏。`retract` 这个词是从学术文献中借用的：被撤回的研究论文仍然可以使用，但它有问题，不应该成为未来工作的基础。

当一个模块的版本被撤回时，用户将不会使用 `go get`, `go mod tidy` 或其他命令自动升级到该版本。依赖于撤回版本的构建应该继续工作，但用户在使用 `go list -m -u` 检查更新或使用 `go get` 更新相关模块时，将被告知撤回的情况。

# 参考
```bash
# Golang 工程管理与业务实践
https://wenzhiquan.github.io/2021/05/16/2021-05-16-golang-dependency/

# Go语言实战笔记（一）| Go包管理
https://www.flysnow.org/2017/03/04/go-in-action-go-package
```