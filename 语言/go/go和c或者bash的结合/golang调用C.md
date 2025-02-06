```table-of-contents
```
# 背景
## c语言的优缺点
### 优点
### 缺点
### 应用场景
## go语言的优缺点
### 优点
### 缺点
### 应用场景
（1）go实现grpc server，go支持protocol buffer.

# cgo介绍
CGO 是 GO 语言里面的一个特性，CGO 属于 GOLANG 的高级用法，主要是通过使用 GOLANG 调用 CLANG 实现的程序库。

我们可以使用`import "C"` 来使用 CGO 这个特性。
```go
package main

/*
CGO的标准写法：
1. 先用注释的方式写入，单行注释和多行注释都支持
  1.1 编译器环境变量
	1.1.1 CFLAGS: C编译选项
	1.1.2 CXXFLAGS: C++编译选项
	1.1.3 CPPFLAGS: C和C++共有的编译选项
	1.1.4 FFLAGS: Fortran编译选项
	1.1.4 LDFLAGS: 链接选项（不区分C和C++）
  1.2 C代码
2. import "C"，相当于将所有的C函数放入虚拟的package C。之后通过`C.xxx`的方式来调用。需要紧跟注释之后。
*/

/*
#cgo LDFLAGS: -lm

#include <math.h>
double my_sqrt(double x) {
	return sqrt(x);
}
*/
import "C" // CGO的标准写法，相当于将C函数放入包`C`中
import "fmt"

func main() {
	a := 100
	fmt.Printf("sqrt(%v) = %v\n", a, C.my_sqrt(C.double(a))) // 调用上面自定义的函数
	fmt.Printf("sqrt(%v) = %v\n", a, C.sqrt(C.double(a)))    // 调用系统库函数（上述的m）
}

// Output:
//  sqrt(100) = 10
//  sqrt(100) = 10
```

通过上述的例子，我们可以看出，CGO可以调用C的库函数，也可以执行注释中的C代码的函数。一般注释中的C代码都是比较简短的。

## `import "C"`
`import "C"` 的上方可以写需要导入的库 C 库，需要注释起来，CGO 会将此处的注释内容当做 C 的代码，被称为**序文（preamble）**

# cgo的应用场景
# C和Go之间的数据传递
## 基本类型
### go到C的基本类型转换
#### go字符串转为C字符串：CString 
```go
func C.CString(string) *C.char              //go字符串转化为char*
```
其中`C.CString`针对输入的Go字符串，克隆一个C语言格式的字符串；
返回的字符串由C语言的`malloc`函数分配，不使用时需要通过C语言的`free`函数释放。

#### go的Byte切片转化为c指针：CBytes
```go
func C.CBytes([]byte) unsafe.Pointer        // go 切片转化为指针
```
`C.CBytes`函数的功能和`C.CString`类似，用于从输入的Go语言字节切片克隆一个C语言版本的字节数组，同样返回的数组需要在合适的时候释放。

### C到go的基本类型转换
#### c字符串转换为go字符串：GoString
```go
func C.GoString(*C.char) string             //C字符串 转化为 go字符串
func C.GoStringN(*C.char, C.int) string     
```
`C.GoString`用于将从`NULL`结尾的C语言字符串克隆一个Go语言字符串。
`C.GoStringN`是另一个字符数组克隆函数。


#### c指针转化为go的Byte切片：GoBytes
```go
func C.GoBytes(unsafe.Pointer, C.int) []byte
```
`C.GoBytes`用于从C语言数组，克隆一个Go语言字节切片。

## 指针类型
### go中直接访问C语言的内存来进行数据转换

#### 背景
**`C.GoString`用于将从NULL结尾的C语言字符串克隆一个Go语言字符串。`C.GoStringN`是另一个字符数组克隆函数。`C.GoBytes`用于从C语言数组，克隆一个Go语言字节切片**。

克隆方式实现转换的优点是接口和内存管理都很简单，缺点是克隆需要分配新的内存和复制操作都会导致额外的开销。


#### unsafe.Pointer
`unsafe.Pointer`在`C`语言中类似`void*`；==在`go`中可以通过 `unsafe.Pointer` 转换，将任何`C`中的指针类型传递给`C函数`中需要`void*`的参数==。

但是在`go`中无法用`unsafe.Pointer`进行运算。 
#### `uintptr`和`unsafe.Pointer`
`go`中有一种类型`uintptr`可以与`unsafe.Pointer`进行互相转换，并且可以进行运算。

#### 分析
如果不希望单独分配内存，可以在Go语言中直接访问C语言的内存空间。

于是我有如下思路：

- 利用`unsafe.Pointer`将`C`指针转化为`go`指针，也就是获取结构体数组的地址
- 然后转成`uintptr`
- 计算出元素偏移量，根据元素偏移量取出元素

#### 范例


## 结构体类型
### go中使用 C语言中的结构体
#### 结构体中的指针成员

## 数组
### go中的[]byte内容传递给C中的uint8数组
在 Go 语言中，`[]byte` 切片可以通过 cgo 机制转换为 C 语言中的 `uint8_t` 数组。

#### 步骤
1. **创建一个 `[]byte` 切片**：在 Go 中定义一个 `[]byte` 切片。
2. **使用 `C.CBytes` 函数**：将 Go 的 `[]byte` 切片转换为 C 的 `void*` 指针。
3. **转换为 `uint8_t*`**：通过类型转换将指针转换为 `uint8_t*`。
3. **传递`uint8_t*`和长度到C中**：传递起始地址和长度到C语言中，然后C语言中进行拷贝处理。

#### 范例
```go
package main

/*
#include <stdio.h>
#include <stdint.h>

// C 函数，用于打印 uint8_t 数组
void printUint8Array(uint8_t* arr, int length) {
    for (int i = 0; i < length; i++) {
        printf("%u ", arr[i]);
    }
    printf("\n");
}
*/
import "C"
import (
    "fmt"
    "unsafe"
)

func main() {
    // 创建 Go 的 []byte 切片
    goSlice := []byte{1, 2, 3, 4, 5}

    // 获取切片的长度
    length := len(goSlice)

    // 将 Go 的 []byte 切片转换为 C 的 uint8_t 数组
    cArray := (*C.uint8_t)(C.CBytes(goSlice))
    defer C.free(unsafe.Pointer(cArray)) // 确保在使用后释放内存

    // 调用 C 函数打印 uint8_t 数组
    C.printUint8Array(cArray, C.int(length))

    // 也可以在 Go 中打印，验证转换
    fmt.Println("Go slice:", goSlice)
}

```

### go中使用C语言中的数组

## 枚举类型
## 联合体类型
## 零拷贝


# cgo的使用
## cgo使用pkg-config
参考：[Golang使用pkg-config自动获取头文件和链接库的方法](https://blog.sina.com.cn/s/blog_48c95a190102w2ln.html)

### 背景
在Golang中使用cgo调用C库的时候，如果需要引用很多不同的第三方库，那么使用`#cgo CFLAGS:`和`#cgo LDFLAGS:`的方式会引入很多行代码。

首先这会导致代码很丑陋，最重要的是如果引用的不是标准库，头文件路径和库文件路径写死的话就会很麻烦。一旦第三方库的安装路径变化了，`Golang`的代码也要跟着变化，所以使用`pkg-config`无疑是一种更为优雅的方法，不管库的安装路径有何变化，我们都不需要修改Go代码。

### 范例
 首先假定我们在路径`/home/ubuntu/third-parties/hello`下安装了一个名称为`hello`的第三方C语言库，其目录结构如下所示，在`hello_world.h`中只定义了一个接口函数`hello`，该函数接收一个`char *`字符串作为变量并调用`printf`将其打印到标准输出。

```bash
# tree /home/ubuntu/third-parties/hello/
/home/ubuntu/third-parties/hello/
├── include
│   └── hello_world.h
└── lib
    ├── libhello.so
    └── pkgconfig
       └── hello.pc
======

# cat hello.pc 
prefix=/home/ubuntu/third-parties/hello
exec_prefix=${prefix}
libdir=${exec_prefix}/lib
includedir=${exec_prefix}/include
Name: hello
Description: The hello library just for testing pkgconfig
Version: 0.1
Libs: -lhello -L${libdir}
Cflags: -I${includedir}


# export PKG_CONFIG_PATH=/home/ubuntu/third-parties/hello/lib/pkgconfig
# pkg-config --list-all | grep libhello
libhello    libhello - The hello library just for testing pkgconfig

# export LD_LIBRARY_PATH=/home/ubuntu/third-parties/hello/lib
// 执行可执行文件的时候，可以查找到动态库
```

`Golang`调用C语言接口的代码示例，我们只需要`#cgo pkg-config: libhello`和`#include < hello_world.h >`两行语句即可实现对hello函数的调用。如果`C`语言库的安装路径发生了变化，只需修改`hello.pc`这个描述文件即可，`Golang`代码无需重新修改和编译。


```go
package main
// #cgo pkg-config: libhello
// #include < stdlib.h >
// #include < hello_world.h >
import "C"
import (
  "unsafe"
)
func main() {
  msg := "Hello, world!"
  cmsg := C.CString(msg)
  C.hello(cmsg)
  C.free(unsafe.Pointer(cmsg))
}
```

 最后，编译该程序代码，查看可执行程序是否正确链接了C语言库，执行程序验证能否正确调用库函数功能。

```go
# go build hello_world.go 
# ldd hello_world
linux-vdso.so.1 =>  (0x00007ffff63d3000)
libhello.so => /home/ubuntu/third-parties/hello/lib/libhello.so (0x00007fc31c0e1000)
libpthread.so.0 => /lib/x86_64-linux-gnu/libpthread.so.0 (0x00007fc31bec3000)
libc.so.6 => /lib/x86_64-linux-gnu/libc.so.6 (0x00007fc31bafe000)
/lib64/ld-linux-x86-64.so.2 (0x00007fc31c2e3000)

# ./hello_world 
Hello, world!
```

## cgo中使用C的宏
C语言中的宏是在编译的时候进行宏替换。对于Golang来说，应该无法替换。因此，cgo中应该无法直接引用C中的宏。


# cgo使用的注意事项
## cgo中malloc申请的释放问题

# cgo的性能
# cgo带来的问题


# golang使用C语言的lib库


# 参考
```bash
# CGO 和 CGO 性能之谜
https://cloud.tencent.com/developer/article/1650525

# CGO函数调用与数据转换【++++】
https://miaoerduo.com/2023/01/29/cgo/

# Go和C之间的类型转换
https://www.kancloud.cn/idzqj/customer/2128198

# cgo遍历C的结构体数组
https://www.kancloud.cn/idzqj/customer/2108413

# cgo中的 CString (Go->C), C.GoString(C->Go)
https://blog.csdn.net/cbmljs/article/details/86629171

# cgo is not Go
https://dave.cheney.net/2016/01/18/cgo-is-not-go

# cgo 优化：被踢出来三次，终于进入 Go 1.21 rc1 版本
https://uncledou.site/2023/cgo-optimization-in-go-1.21/

# cgo 优化第二弹：内存优化
https://uncledou.site/2023/cgo-memory-optimization/

# 又来折腾，cgo 内存优化无缘 golang 1.22
https://uncledou.site/2023/cgo-memory-optimization-delay/

# 如何写出高效的 cgo 代码
https://uncledou.site/2023/cgo-best-performance-way/


```