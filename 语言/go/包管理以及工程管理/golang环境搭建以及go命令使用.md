```table-of-contents
```
# go环境搭建

## 开发工具包下载和安装
要搭建Go语言开发环境，我们第一步要下载go的开发工具包。

Go为我们所熟知的所有平台架构提供了开发工具包，比如我们熟知的Linux、Mac和Windows，其他的还有FreeBSD等。

### 基于机器选择开发工具包
可以根据自己的机器操作系统选择相应的开发工具包；
比如你的是Windows 64位的，就选择windows-amd64的工具包；
是Linux 32位的就选择linux-386的工具包。
可以自己查看下自己的操作系统，然后选择，Mac的现在都是64位的，直接选择就可以了。


根据自己的操作系统选择后，就可以下载开发工具包了，Go语言的官方下载地址是 [https://golang.org/dl/](https://golang.org/dl/) 可以打开选择版本下载，如果该页面打不开，或者打开了下载不了，可以通过Golang的国内网站 [https://golang.google.cn/dl/](https://golang.google.cn/dl/) 下载。

### 开发工具包分类
开发工具包又分为安装版和压缩版。
#### 安装版
安装版是Mac和Windows特有的，他们的名字类似于：
- go1.16.4.darwin-amd64.pkg
- go1.16.4.windows-386.msi
- go1.16.4.windows-amd64.msi

安装版，顾名思义，双击打开会出现安装向导，让你选择安装的路径，帮你设置好环境变量等信息，比较省事方便一些。

#### 压缩版
压缩版的就是一个压缩文件，可以解压得到里面的内容，他们的名字类似于：

- go1.16.4.darwin-amd64.tar.gz
- go1.16.4.linux-386.tar.gz
- go1.16.4.linux-amd64.tar.gz
- go1.16.4.windows-386.zip
- go1.16.4.windows-amd64.zip

压缩版我们下载后需要解压，然后自己移动到要存放的路径下，并且配置环境变量等信息。

### linux下安装
```bash
wget https://go.dev/dl/go1.22.0.linux-amd64.tar.gz
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.22.0.linux-amd64.tar.gz
```

解压后的 `/usr/local/go`目录下的结构，`go`文件夹下是`bin、src、lib、pkg、doc`等目录。如下所示：
![](attachments/Pasted%20image%2020241212105125.png)

如果是自己用软件解压的，可以拷贝到`/usr/local/go`下，但是要保证你的go文件夹下是`bin、src、doc`等目录，不要go文件夹下又是一个go文件夹，这样就双重嵌套了。

#### 配置环境变量
配置环境变量，Linux下又两个文件可以配置，其中`/etc/profile`是针对所有用户都有效的； `$HOME/.profile`是针对当前用户有效的，可以根据自己的情况选择。

在`/etc/profile`文件的末尾添加如下配置保存即可：
```bash
echo "export GOROOT=/usr/local/go" >> $HOME/.profile
echo "export PATH=$PATH:$GOROOT/bin" >> $HOME/.profile
source $HOME/.profile
```
![](attachments/Pasted%20image%2020241212110518.png)

### mac下安装
Mac分为压缩版和安装版，他们都是64位的。压缩版和Linux的大同小异，因为Mac和Linux都是基于Unix，终端这一块基本上是相同的。

压缩版解压后，就可以和Linux一样放到一个目录下，这里也以`/usr/local/go/`为例。

在配置环境变量的时候，针对所有用户和Linux是一样的，都是`/etc/profile`这个文件；针对当前用户，Mac下是`$HOME/.bash_profile`，其他配置都一样。

# go env环境变量
## 常见的环境变量 go env

![](attachments/Pasted%20image%2020241212111823.png)

### 每个环境变量的含义
要搞清楚每个字段什么意思，可以参考官方解释。
```bash
go help environment
```

![](attachments/Pasted%20image%2020241212142618.png)

## GOROOT
其中`GOROOT`环境变量表示我们GO的安装目录，
### 作用
`GOROOT`环境变量的作用如下：
**（1）便于GO开发程序查找到 Go-SDK**
Go开发IDE就可以基于`GOROOT`环境变量自动的找到Go安装目录，达到自动配置Go SDK的目的。

**（2）设置PATH，方便使用go、gofmt等命令**
把`/usr/local/go/bin`这个目录加入到环境变量PATH里，这样我可以在终端里直接输入go等常用命令使用了，而不用再加上`/usr/local/go/bin`这一串绝对路径，更简洁方便。

## GOPATH
在 go1.12 之前，安装 golang 之后，需要配置两个环境变量----GOROOT 和GOPATH。GOPATH 环境变量定义了 Go 项目工作区的路径。
==默认情况下 GOPATH 为 `$HOME/go` 目录==；

go1.12 之后，淡化了 GOPATH，因此也可以忽略这部分内容。默认情况下 GOPATH 为 `$HOME/go` 目录。

### GOPATH的工作区目录
在 `GOPATH` 定义的工作区下，有三个目录：`bin`，`pkg`，`src`。
```bash
.
├── bin
├── pkg
└── src
  └── github.com/foo/bar
    └── bar.go
```
**（1）`$GOPATH/bin` **
`$GOPATH/bin` 目录是 GO 用来存放通过命令 go install 编译的二进制文件的位置。
操作系统会根据全局环境变量 `$PATH` 来查找和执行编译后的二进制应用程序。所以最好把`$GOPATH/bin`目录添加到全局环境变量`$PATH` 中。

**(2) `$GOPATH/pkg`**
`$GOPATH/pkg` 目录是 Go 存储预编译目标文件的地方，以加速程序的后续编译。
通常，大多数开发人员不需要访问这个目录。 如果在编译时遇到问题，可以安全地删除这个目录，然后 Go 将重新编译它。

**(3) `$GOPATH/src`**
`src` 目录是所有 .go 文件或源代码的位置。这不应该与 Go 工具使用的源代码相混淆，后者位于 $GOROOT。
可以在src里创建多个项目。每一个项目同时也是一个文件夹。



### GOPATH目前状态
在`Go1.14`及之后的版本中启用了`Go Module`模式之后，不一定非要将代码写到`GOPATH`目录下，`所以也就不需要我们再自己配置GOPATH了`，使用默认的即可。

### GOPATH 和 GOROOT 的区别
GOPATH 标识 go项目开发源码所在目录。
GOROOT 标识 go源程序的安装目录，是Go 的代码，编译器和工具的区域，并不是我们编写的源代码。


## `GO111MODULE` 环境变量
在环境变量中，`GO111MODULE`变量作为`Go modules`的开关，其允许设置如下参数：

- `auto`：项目若包含`go.mod`文件，则启用`Go modules`；
- `on`：启用`Go modules`，go命令行会使用`modules`，而不会去`GOPATH`目录下查找依赖包，推荐设置；
- `off`：禁用`Go modules`，寻找依赖包的方式沿用旧版本通过`vendor`目录或者`GOPATH`模式来查找，不推荐设置；



  如果需要对`GO111MODULE`环境变量的值进行变更，则可以使用如下命令：
```go
go env -w GO111MODULE=on
```

## `GOPROXY`
### 背景

`GOPROXY`时代之前，在`Golang`开发时，模块依赖关系直接从版本控制（`VCS`）系统中的源存储库下载，如`GitHub、Bitbucket、Bazaar、Mercurial`或`SVN`。来自第三方的依赖项通常从公共源`repos`下载。这种形式缺乏确定性和安全性，以及开发中的两个基本需求：不变性和可用性。模块可以被作者删除，也可以编辑修改当前被发布的版本。

![](attachments/Pasted%20image%2020241217151822.png)

### 介绍
`GOPROXY`控制`Go Module`下载的来源，有助于确保构建的确定性和安全性。
设置`GOPROXY`，将`Go Module`下载请求重定向到`GOPROXY` 指向的缓存库。使用`GOPROXY`进行模块依赖关系的管理的有助于开发构建不变性需求。
另外`GOPROXY`的缓存还有助于确保模块始终可用，即使`VCS repo`中的原始模块已被销毁。


要使用公共`GOPROXY`，将`Golang`环境变量设置为其URL： 
```bash
go env -w GOPROXY=https://goproxy.io  
```
以上设置将所有模块下载请求重定向到`GoCenter`，从公共`GOPROXY`下载要比直接从`VCS`下载快得多。

### 国内的代理点
`GOPROXY` 的默认值是：`https://proxy.golang.org,direct`，该站点在国内无法正常访问，因此在开启`Go modules`时，需要设置国内的Go模块代理。目前社区使用比较多的有两个`https://goproxy.cn`和`https://goproxy.io`。

执行命令如下：
```go
go env -w GOPROXY=https://goproxy.cn,direct
```

### 多个代理地址
`GOPROXY`允许设置多个模块代理地址，多个地址之间以英文逗号“,”分隔开，当配置多个代理地址时，值代理地址列表中上一个Go模块代理地址返回404 或 410 错误时，Go 会自动尝试下一个模块代理地址，当遇到 `direct`时会回到源地址去抓取，当遇到`EOF`时中止并抛出错误。

若不想使用，则可以设置为“off”，这将会禁止Go在后续操作中使用任何的Go模块代理。

### direct 
`direct`是一个特殊指示符，指示 `go` 命令从开发模块的版本控制存储库下载模块，而不是使用代理。

### `GONOPROXY`
不应从代理下载的模块路径前缀的 glob 模式列表 (注：glob 是用于匹配符合指定模式的文件集合的一种语言，下同)。 `go` 命令将从它们开发的版本控制存储库中下载匹配的模块，而不管 `GOPROXY`。


## `GOPRIVATE`
### 背景
在使用gomod模式管理golang包的时候，下载开源的公共包还可以，但是一旦使用内部或者私有的包，就可能会出现如下所示的问题：
```text
server response: not found: git.xxx.com/xxxxxx/xxx@v0.6.8: unrecognized import path "git.xxx.com/xxxxxx/xxx": https fetch: Get "https://git.xxx.com/xxxxxx/xxx?go-get=1": dial tcp xx.xx.xx.xx:443: connect: connection refused

```

### 解决方法：GOPRIVATE
![](attachments/Pasted%20image%2020241217152021.png)

通常，Golang项目会同时使用开源和私有模块。一些用户使用 `GOPRIVATE` 环境变量来指定一个必须绕过 `GOPROXY` 和 `GOSUMDB` 的路径列表，并直接从`VCS repos`下载私有模块。

![](attachments/Pasted%20image%2020241216163142.png)

```go
$ export GOPROXY=https://gocenter.io,direct
$ export GOPRIVATE=*.http://internal.mycompany.com
```

## `GOSUMDB`
`GOSUMDB`环境变量（Go checksum database）在拉取依赖版本时（无论是在源站点还是Go module proxy）保证拉取的模块版本数据未经过篡改，若发现不一致，即可能存在篡改，则会立即中止拉取。

`GOSUMDB`的默认值为`sum.golang.org`，该站点在国内无法访问，不过可以通过之前设置的`GOPROXY`解决，`GOPROXY`设置的代理`goproxy.cn`同样支持代理`sum.golang.org`。

若该值设置为`off`，则禁止GO在后续操作中校验模块版本。

### `GONOSUMDB`
不应使用公共校验和数据库 [sum.golang.org](https://sum.golang.org/) 检查的模块路径前缀的 glob 模式列表。

## `GOMODCACHE` 环境变量
在 Go 1.15，可以通过 GOMODCACHE 环境变量设置模块缓存的位置。
即存储go下载的外部依赖模块文件的目录，默认值为`$GOPATH/pkg/mod`。
因此我们一般只需要更改`GOPATH`的值即可，此环境变量的值就会自动做出相应的变动。当然你也可以设置为其他值。

## `GOCACHE`环境变量
此目录存放go项目在构建过程中产生的缓存。

## GOARCH
![](attachments/Pasted%20image%2020241212142820.png)
`GOARCH`指的是目标处理器的架构，目前支持的有：
```go
arm
arm64
386
amd64
ppc64
ppc64le
mips
mipsle
mips64
mips64le
s390x
wasm
```
一共支持12种处理器的架构。
## GOOS
`GOOS`指的是目标操作系统.
它的可用值为：
```go
aix
android
darwin
dragonfly
freebsd
illumos
js
linux
netbsd
openbsd
plan9
solaris
windows
```
一共支持13种操作系统。



## 设置某个环境变量的值
```go
go env -w GO111MODULE="on"
```
##  获取某个环境变量的值
```go
go env GOPATH
```

# go项目的工程结构
基于Go Module，你可以在任意位置创建一个Go项目，而不再像以前一样局限在`$GOPATH/src`目录下。




# go命令使用

![](attachments/Pasted%20image%2020241216174155.png)

可以发现，go支持的子命令很多，同时还支持查看一些【主题】。我们可以使用`go help [command]`或者`go help [topic]`查看一些命令的使用帮助，或者关于某个主题的信息。

大部分==go的命令，都是接受一个全路径的包名作为参数==，比如我们经常用的`go build`。

## go命令参数

## go version

```go
#打印Go版本
$ go version

#打印Go版本用于构建特定的可执行文件。
$ go version ~/go/bin/gopls

#打印Go版本和模块版本用于构建特定可执行文件。
$ go version -m ~/go/bin/gopls

#打印Go版本和模块版本用于在目录中构建可执行文件。
$ go version -m ~/go/bin/
```

## go help
### `go help [command]`
### `go help [topic]`

## go doc

![](attachments/Pasted%20image%2020241216201156.png)

```bash
go help doc
usage: go doc [-u] [-c] [package|[package.]symbol[.method]]
```

`go doc`的使用比较简单，接收的参数是包名，或者以包里的结构体、方法等。如果我们不输入任何参数，那么显示的是当前目录的文档。

## go build
`go build`,是我们非常常用的命令，它可以启动编译，把我们的包和相关的依赖编译成一个可执行的文件。
```bash
usage: go build [-o output] [-i] [build flags] [packages]

```

![](attachments/Pasted%20image%2020241216180053.png)

## go clean
在我们使用`go build`编译的时候，会产生编译生成的文件，尤其是在我们签入代码的时候，并不想把我们生成的文件也签入到我们的Git代码库中，这时候我们可以手动删除生成的文件，但是有时候会忘记，也很麻烦，不小心还是会提交到Git中。要解决这个问题，我们可以使用`go clean`,它可以清理我们编译生成的文件，比如生成的可执行文件，生成obj对象等等。

```go
usage: go clean [-i] [-r] [-n] [-x] [build flags] [packages]
```


## go run
`go build`是先编译，然后我们再执行可以执行文件来运行我们的程序，需要两步。`go run`这个命令就是可以把这两步合成一步的命令，节省了我们录入的时间，通过`go run`命令，我们可以直接看到程序执行输出的结果。

`go run`命令需要一个go文件作为参数，这个go文件必须包含main包和main函数，这样才可以运行，其他的参数和`go build`差不多。 在运行`go run`的时候，如果需要的话，我们可以给我们的程序传递参数，比如：
```go
package main

import (
	"fmt"
	"os"
)


func main() {
	fmt.Println("输入的参数为：",os.Args[1])
}

```

输入如下命令执行：
```go
go run main.go 12
```

## go install


**go install**：`go install` 命令用于编译并安装指定的包或可执行文件。

```go
usage: go install [build flags] [packages]
```
当您执行 `go install <package>` 命令时，Go 工具会编译包的源代码，并将生成的可执行文件安装到 `$GOPATH/bin` 目录下；如果生成的是可引用的库，那么安装在`$GOPATH/pkg`目录下。

### 使用场景
`go install` 命令通常用于将本地项目或其他第三方项目等编译为可执行文件，并安装到系统路径中以供全局直接执行。

如果想要编译一个 Go 程序并在系统上安装可执行文件，则可以使用 `go install` 命令。

## go get
### 介绍
**go get**：该命令主要用于获取并安装指定的远程包或依赖库。

当执行：`go get <package>` 命令时，Go工具会下载指定包的源代码，并将其安装到 `$GOPATH/src` 目录下。

如果只是想下载某个包的源代码但不需要编译可执行文件，则可以使用 `go get` 命令。


### 注意
`go get`的本质是使用源代码控制工具下载这些库的源代码，比如`git`等，所以在使用之前必须确保安装了这些源代码版本控制工具。

```go
go get -u -v github.com/spf13/cobra
```

### go get 的变更历史

- Go 1.17 之前：`go get` 通过远程拉取或更新代码包及其依赖包，并`自动完成编译和安装`。实际分成两步操作：
```go
1. 下载源码包
2. 执行 go install
```
- Go 1.17 之后: `弃用go get命令的编译和安装功能`

![](attachments/Pasted%20image%2020241216203550.png)

```go
Starting in Go 1.17, installing executables with go get is deprecated. go install may be used instead.
In Go 1.18, go get will no longer build packages; it will only be used to add, update, or remove dependencies in go.mod.
Specifically, go get will always act as if the -d flag were enabled.

```

- 自 `Go 1.17` 起, 弃用 `go get` 命令安装可执行文件，使用 `go install` 命令替代
- 自`Go 1.18`起，`go get` 命令不再有编译包的功能。将只有添加，更新，移除 `go.mod` 文件中的依赖项的功能。
- `go get` 命令将默认启用 `-d` 选项。


### go get命令变更的原因

```text
Since modules were introduced, the go get command has been used both to update dependencies in go.mod and to install commands.
This combination is frequently confusing and inconvenient: in most cases, developers want to update a dependency or install a command but not both at the same time.

Since Go 1.16, go install can install a command at a version specified on the command line while ignoring the go.mod file in the current directory (if one exists).
go install should now be used to install commands in most cases.

go get’s ability to build and install commands is now deprecated, since that functionality is redundant with go install.
Removing this functionality will make go get faster, since it won’t compile or link packages by default.
go get also won’t report an error when updating a package that can’t be built for the current platform.

```

由于 `go 1.11` 之后 `go mod` modules特性的引入，使得`go get` 命令，既可以安装第三方命令，又可以从 `go.mod` 文件自动更新项目依赖。但多数情况下，开发者只想做二者之一。

自 `go 1.16` 起，`go install` 命令，可以忽略当前目录的 `go.mod`文件(如果存在)，直接安装指定版本的命令行应用。

`go get` 命令的编译和安装功能，因为和 `go install` 命令的功能重复，故被弃用。由于弃用了编译和安装功能，`go get` 命令将获得更高的执行效率, 也不会在更新包的时候，再出现编译失败的报错。



### go get 和  go install 区别

**（1）`go get`**:
`go get` 对 `go mod` 项目，添加，更新，删除 `go.mod` 文件的依赖项(仅源码)。不执行编译。侧重应用依赖项管理。

**（2）`go install`**
 `go install` 在操作系统中安装 `Go 生态的第三方命令行应用`。不更改项目 `go.mod` 文件。侧重可执行文件的编译和安装。


### go get 和  go install 如何选择

`go get` 由于具备更改 `go.mod` 文件的能力，如果我们不期望更改`go.mod` 文件，我们必须要避免执行 `go get` 命令，否则它会更改`go.mod` 文件，将我们安装的工具作为一个依赖。

所以，==`如果不是为了更新项目依赖，而是安装可执行命令，请使用 go install`。==


### go get 和 go mod tidy 

 `go build` 和 `go test` 默认情况下不再修改 `go.mod` 和 `go.sum`。可通过 `go mod tidy` ，`go get` 或者手动完成；

`go get` 主要被设计为修改 `go.mod` 追加依赖之类的，还存在类似 `go mod tidy` 之类的命令；

## go list
## go test

该命令用于Go的单元测试，它也是接受一个包名作为参数，如果没有指定，使用当前目录。 
`go test`运行的单元测试必须符合go的测试要求。
1. 写有单元测试的文件名，必须以`_test.go`结尾。
2. 测试文件要包含若干个测试函数。
3. 这些测试函数要以Test为前缀，还要接收一个`*testing.T`类型的参数。


```go
package main

import "testing"

func TestAdd(t *testing.T) {
	if Add(1,2) == 3 {
		t.Log("1+2=3")
	}

	if Add(1,1) == 3 {
		t.Error("1+1=3")
	}
}

```
这是一个单元测试，保存在`main_test.go`文件中，对main包里的`Add(a,b int)`函数进行单元测试。 如果要运行这个单元测试，在该文件目录下，执行`go test` 即可。

## go fmt
go fmt 统一代码风格，这样我们再也不用为大括号要不要放到行尾还是另起一行，缩进是使用空格还是tab而争论不休了，都给我们统一了。

```go
usage: go fmt [-n] [-x] [packages]
```
`go fmt`接受一个包名作为参数，如果不传递，则使用当前目录。`go fmt`会自动格式化代码文件并保存，它本质上其实是调用的`gofmt -l -w`这个命令。

![](attachments/Pasted%20image%2020241216192550.png)



## go vet
 go vet 会帮助我们检查我们代码中常见的错误。

```go
usage: go vet [-n] [-x] [build flags] [packages]
```
养成在代码提交或者测试前，使用`go vet`检查代码的好习惯，可以避免一些常见问题。

# 开发go项目
## 获取包
### 获取远程包
在获取远程包之前，我们需要先配置代理。
这里推荐`goproxy.io`代理，设置命令如下：
```go
go env -w GO111MODULE=on
go env -w GOPROXY=https://goproxy.io,direct
go get -v github.com/spf13/cobra
```
就可以下载这个库到我们`$GOPATH/pkg/mod`目录下了，这样我们就可以像导入其他包一样`import`了。

### 获取私有包
#### 背景
如果是私有的git库怎么获取呢？比如在公司使用gitlab搭建的git仓库，设置的都是private权限的。

#### GOPRIVATE设置
这种情况下我们可以配置下git，就可以了，在此之前你公司使用的gitlab必须要在7.8之上。然后要把我们http协议获取的方式换成ssh，假设你要获取`http://git.flysnow.org`，对应的`ssh`地址为`git@git.flysnow.org`，那么要在终端执行如下命令。

```bash
git config --global url."git@git.flysnow.org:".insteadOf "http://git.flysnow.org/"
```
这段配置的意思就是，当我们使用`http://git.flysnow.org/`获取git库代码的时候，实际上使用的是`git@git.flysnow.org`这个url地址获取的，也就是http到ssh协议的转换，是自动的，他其实就是在我们的`~/.gitconfig`配置文件中，增加了如下配置:
```bash
[url "git@git.flysnow.org:"]
	insteadOf = http://git.flysnow.org/
```

然后需要把`git.flysnow.org`加入`GOPRIVATE`环境变量中，因为它是你的私有仓库，不需要走`GOPROXY`代理。
```bash
# 设置不走 proxy 的私有仓库，多个用逗号相隔（可选）
go env -w GOPRIVATE=git.flysnow.org
```

现在我们就可以使用`go get`直接获取了，比如：
```bash
go get -v -insecure git.flysnow.org/hello

```
仔细看，多了一个`-insecure`标识，因为我们使用的是http协议， 是不安全的。当然如果你自己搭建的gitlab支持https协议，就不用加`-insecure`了，同时把上面的`url insteadOf`换成`https`的就可以了。

## 编译

### 跨平台编译
`GOOS`和`GOARCH`组合起来，支持生成的可执行程序种类很多，具体组合参考[https://golang.org/doc/install/source#environment](https://golang.org/doc/install/source#environment)。

如果我们要生成不同平台架构的可执行程序，只要改变这两个环境变量就可以了，比如要生成linux 64位的程序，命令如下：
```go
GOOS=linux GOARCH=amd64 go build flysnow.org/tour
```
前面两个赋值，是更改环境变量，这样的好处是只针对本次运行有效，不会更改我们默认的配置。

# 参考
```bash
# Go语言环境搭建详解（2021版）
https://www.flysnow.org/2021/05/15/install-golang

# go的命令使用【gon开发环境】
https://www.flysnow.org/2017/03/08/go-in-action-go-tools
```