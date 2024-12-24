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

# cgo的应用场景
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

## 注意事项
### cgo中的 CString 
### cgo中的 GoString
### cgo中malloc申请的释放问题

# cgo的性能
# cgo带来的问题


# 参考
```bash
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