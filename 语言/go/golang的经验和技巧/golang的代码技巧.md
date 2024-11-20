```table-of-contents
```
# 概述
本文提供了一些 Golang 工程项目的最佳实践，分别从可读性、健壮性和效率三个方面进行描述。

# 可读性(Readable)
## if，else and happy path
```go
// Bad example 1
func abs(x int) {
    if x >= 0 {
        return x
    } else {
        return -x
    }
}

// Good example 1
func abs(x int) {
    if x >= 0 {
        return x
    }
    return -x
}

```
上面这个例子是去掉不必要的 else，这样可以让我们的代码可读性更高。

```go
// Bad example 2
func aFunc() error {
    err := doSomething()
    if err == nil {
        err := doAnotherThing()
        if err == nil {
            return nil // happy path
        }
        return err
    }
    return err
}

// Good example 2
func aFunc() error {
    err := doSomething()
    if err != nil {
        return err
    }
    err := doAnotherThing()
    if err != nil {
        return err
    }
    return nil
}

```
显而易见，第二种代码可读性比第一种要高得多，这就是尽早返回错误的写法，同时也消除了 happy path，让我们的代码变得更加的清晰易懂。

```go
// Bad example 3
var a int
if flag {
    a = 1
} else {
    a = -1
}
// Good example 3
var a = -1
if flag {
    a = 1
}

```
进行类似 bool 判断后赋值操作时，我们可以提前分配默认值，提升可读性。

## init()
我们==尽量不要使用 init() 函数来进行赋值和初始化等操作==；
因为 init() 函数在包被引用时就会自动调用，可能会产生意想不到的副作用。

```go
// Bad example
package tiktokclients

func init() {
    tiktokclient = buildNewTiktokClient()
    ...
}

```
上述的写法会在每一个即使不想 build tiktokclient 的引用 tiktokclients 包的地方都调用一次 init() 方法，导致 tiktokclient 不断的 build，从而产生问题。

```go
// Good example
package tiktokclients

func InitClient() {
    tiktokclient = buildNewTiktokClient()
    ...
}

```
使用显示的初始化方法，需要初始化的地方才进行初始化，从而避免 init() 函数的副作用。

## comments
注释可以辅助我们更好的理解代码，但是不好的注释反而可能起到反作用。
```go
// Bad example 1
func Abs(num int) int {
    // if num is negative
    if num < 0 {
        reutrn -num
    }
    // if num is non-negative
    return num
}

// Good example 1
// Return abosulote value of an int value
func Abs(num int) int {
    if num < 0 {
        reutrn -num
    }
    return num
}

```
注释不用解释函数里面每一个逻辑是什么样的，直接解释这个函数是什么作用就可以了。
对于我们而言，其实写出可读性更加好的代码的意义要比写注释更加大，因为注释可能会过时，而代码一定是最新的.

# 健壮性(Robust)
健壮的代码指的是我们的代码发生错误之后，我们的程序不至于崩溃而还能继续执行。在 Golang 中，最重要的就是处理 panic 和 error。

## panic
```go
// Bad example
func main() {
    fmt.Println("start")
    panic(errors.New("this is a panic))
    p := recover()
    fmt.Print("panic: %v", p)
    fmt.Println("end")
}

// Good example
func main() {
    defer func() {
        p := recover()
        fmt.Print("panic: %v", p)
    }
    fmt.Println("start")
    panic(errors.New("this is a panic))
    fmt.Println("end")
}

```
发生 panic 的函数会直接抛出 panic 而不会继续运行之后的代码，所以为了捕获 panic，我们需要在函数开头使用 defer 匿名函数来保证可以捕获到 panic

## error
```go
// Bad example 1 (Go < 1.13)
func main() {
    e := io.EOF
    fmt.Println(e == io.EOF) // true
    e = fmt.Errorf("contenxt: %w", io.EOF)
    fmt.Println(e == io.EOF) // false
}

// Good example 1
func main() {
    e := io.EOF
    fmt.Println(errors.Is(e, io.EOF)) // true
    e = fmt.Errorf("contenxt: %w", io.EOF)
    fmt.Println(errors.Is(e, io.EOF)) // true
}

// Bad example 2 (Go < 1.13)
func main() {
    _, err := os.Open("non-existing")
    _, ok := err.(*fs.PathError)
    fmt.Println(ok) // true

    e := fmt.Errorf("contenxt: %w", err)
    _, ok := err.(*fs.PathError)
    fmt.Println(ok) // false
}

// Good example 2
func main() {
    _, err := os.Open("non-existing")
    var pathError *fs.PathError
    fmt.Println(errors.As(err, &pathError) // true

    e := fmt.Errorf("contenxt: %w", err)
    fmt.Println(errors.As(e, &pathError) // true
}

```
在 Golang 的 1.13 版本之前对于 err 的处理可能会出现一些不符合直觉的错误，比如 fmt.Errorf 拼接之后的错误无法溯源和正确转化等，此时我们可以使用 errors.Is() 和 errors.As() 方法解决此类问题。

# 效率(Efficient)
关于效率，可以使用指针，因为传递指针的时候不会发生值的 copy，可以节省资源提高性能，那么在哪些情况下我们应当使用指针或不使用指针呢？

|Good|Bad|  
|-|-|  
|当方法内部需要修改传递进来的参数时|指针指向一个 interface|  
|需要传递一个很大的数据时|方法内部不需要修改传递进来的参数时|  
|为了保持代码一致性，都使用了指针时|需要传递的数据很小时|

## 指针的坑
使用指针是还有一个坑，就是如果在循环中使用指针，会导致循环时取到的值是同一个
```go
// Bad example
func main() {
    x := []int{1,2,3}
    r := make([]*int, 0, len(x))

    for _, v := range x {
        r = append(r, &v)
    }

    // Print out 3,3,3
    for _, v := range r {
        fmt.Print(*v)
    }
}

// Good example
func main() {
    x := []int{1,2,3}
    r := make([]*int, 0, len(x))

    for _, v := range x {
        c := v
        r = append(r, &c)
    }

    // Print out 1,2,3
    for _, v := range r {
        fmt.Print(*v)
    }
}

```
第二种写法也是标准的循环中使用 goroutine 时的写法，先用一个局部变量承接循环的值，再将局部变量传递到 goroutine 中。
## deprecation
我们的代码是不断更新的，当旧的方法不再使用时，我们需要在其上添加 `Deprecated` 注释，以表示不再建议使用，在 Goland 等 IDE 中还会根据这个注释进行相应的强提示，告诉我们使用更新的方法。

# 参考
```text
# Golang On The Toilet
https://wenzhiquan.github.io/2021/04/18/2021-04-18-gott/
```