```table-of-contents
```
# 循环
在Golang的流程控制中，循环语句有for和range两种。
## for

### 普通格式的 for
```go
// 完整格式的for
for init; condition; modif { }
```


`for 赋值表达式; 关系表达式或逻辑表达式; 赋值表达式 { }`
```go
for i := 0; i < 10; i++ { 

}
```
### 只有条件判断的for
```go
// 只有条件判断的for，实现while的功能
// 要在循环体中加上退出条件，否则无限循环
for condition { }
```

```go
n := 10 
for n > 0 { 
	n-- 
}
```

### 无限循环的for
好几种方式实现for的无限循环。只要省略for的条件判断部分就可以实现无限循环。
```go
for i := 0;;i++ 
for { } 
for ;; { }
for true { }
```

无限循环时，一般在循环体中加上退出语句，如break、os.Exit、return等。

```go
for {
    fmt.Println("hello world")
}
// 等价于
for true {
	fmt.Println("hello world")
}

```

## for-range
Golang range类似迭代器操作，可以对 **slice、map、数组、字符串**等进行迭代循环。

### 返回值
**两个返回值**
在字符串、数组和切片（slice）中它返回 (索引index, 值value) ，在集合map中返回 (键key, 值value)。

**一个返回值**
但若当只有一个返回值时，第一个参数是索引(index)或键(key)。

### 遍历字符串

```go
str := "abc"
for i, char := range str {
    fmt.Printf("%d => %s\n", i, string(char))
}
for i := range str { //只有一个返回值
    fmt.Printf("%d\n", i)
}
// 输出结果
// 0 => a
// 1 => b
// 2 => c
// 0
// 1
// 2
```


### 遍历切片

```go
nums := []int{1, 2, 3}
for i, num := range nums {
    fmt.Printf("%d => %d\n", i, num)
}
// 输出结果
// 0 => 1
// 1 => 2
// 2 => 3
```

### 遍历map

```go
kvs := map[string]string{"a": "apple", "b": "banana"}
for k, v := range kvs {
    fmt.Printf("%s => %s\n", k, v)
}
for k := range kvs { //只有一个返回值
    fmt.Printf("%s\n", k)
}
// 输出结果
// a => apple
// b => banana
// a
// b
```


# 注意事项
```go
for index,value := range XXX {}
```
## value是从XXX中拷贝的副本
value是从XXX中拷贝的副本，所以通过value去修改XXX中的值是无效的，在循环体中应该总是让value作为一个只读变量。如果想要修改XXX中的值，应该通过index索引到源值去修改(不同类型修改的方式不一样)。

```go
func main() {
    s1 := []int{11,22,33}
    for index,value := range s1 {
        value += 1      // 只在for结构中有效
        fmt.Println(index,value)
    }
    fmt.Println(s1)   // for外面的结果仍然是[11 22 33]
}
```

要在循环结构中修改slice，应该通过index索引的方式：
```go
func main() {
    s1 := []int{11,22,33}
    for index,value := range s1 {
        value += 1      // 只在for结构中有效
        fmt.Println(index,value)
    }
    fmt.Println(s1)   // for外面的结果仍然是[11 22 33]
}
```

## value是 per 循环而不是per迭代的
for/range 循环中，由于 func 捕获，或者显式/隐式的取引用，对循环变量产生了引用并且这个引用逃逸出了当前循环迭代（iteration）的生命周期范围。
而由于 Golang 一开始决定将将循环变量（i、k、v）的生命周期定义为整个循环，而不是每个迭代都有新一份的循环变量，导致了每一轮迭代产生的引用实际上都指向同一个值，而不是指向每一轮各自对应的值。

## 原因

==**golang 的循环变量是 per loop 的，而不是 per iteration 的**。
对循环变量产生了引用, 很容易产生问题==。

如果对循环变量产生了引用（比如闭包 capture，或者取指针），不同次迭代取到的指针都是同一个。
如果这个指针/引用被逃逸出了一次迭代的范围内（比如 append 到了一个数组里，或者被go/defer后面的闭包capture了），因为所有 iteration 里取到的指针都是同一个，指向的对象也都会是同一个（最后一轮iteration的结果）。

## 场景
### 场景一：使用循环的变量的显式引用
```go
func main() {
    var out []*int
    for i := 0; i < 3; i++ {
        // i := i
        out = append(out, &i)
    }
    fmt.Println("值:", *out[0], *out[1], *out[2])
    fmt.Println("地址:", out[0], out[1], out[2])
}
// 输出结果
// 值: 3 3 3
// 地址: 0xc000012090 0xc000012090 0xc000012090
```

**分析**
> `out`是一个整型指针数组变量，在for循环中，声明了一个`i`变量，每次循环将`i`的地址追加到`out`切片中，但是每次追加的其实都是`i`变量，因此我们追加的是一个相同的地址，而该地址最终的值是3。

**正确做法**
> 解开代码中的注释`// i := i`，每次循环时都重新创建一个新的`i`变量。


```go
func main() {
    a1 := []int{1, 2, 3}
    a2 := make([]*int, len(a1))

    for i, v := range a1 {
        a2[i] = &v
    }

    fmt.Println("值:", *a2[0], *a2[1], *a2[2])
    fmt.Println("地址:", a2[0], a2[1], a2[2])
}
// 输出结果
// 值: 3 3 3
// 地址: 0xc000012090 0xc000012090 0xc000012090
```
**分析**
> `range`在遍历值类型时，其中的`v`是一个局部变量，**只会声明初始化一次**，之后每次循环时重新赋值覆盖前面的，所以给`a2[i]`赋值的时候其实都是同一个地址`&v`，而`v`最终的值为`a1`最后一个元素的值，也就是3。

**正确做法**：
- `a2[i]`赋值时传递原始指针，即`a2[i] = &a1[i]`  
- 创建临时变量`t := v`；`a2[i] = &t`  
- 闭包(与上面的临时变量原理一样)，`func(v int) { a2[i] = &v }(v)`

### 场景二：循环中使用匿名函数
**现象1**
```go
func TestIterator13(t *testing.T) {
    var squares []func() int
    for i := 0; i < 3; i++ {
        squares = append(squares, func() int {
            return i * i
        })
    }
 
    for _, square := range squares {
        res := square()
        fmt.Println(res)
    }
}
 
// 执行结果
9
9
9
```
**分析**
原因是在 for 循环结束后，最后 i 的值被设置为了 3。
因为循环变量是共享的，打印的不是某一次的值，最后统一输出，值就会被改写，【==笼统的说法是匿名函数是引用类型==】

**正确做法**：
```go
func TestIterator14(t *testing.T) {
    var squares []func() int
    for i := 0; i < 3; i++ {
        i := i
        squares = append(squares, func() int {
            return i * i
        })
    }
 
    for _, square := range squares {
        res := square()
        fmt.Println(res)
    }
}
```

**现象2**
```go
var prints []func()
for _, v := range []int{1, 2, 3} {
    prints = append(prints, func() { fmt.Println(v) })
}
for _, print := range prints {
    print()
}
```
**分析**
这段程序的输出结果是什么？没有 & 取地址符，是输出 1，2，3 吗？
结果程序一运行，输出结果是 3，3，3。这又是为什么？

问题的重点之一：关注到闭包函数，实际上所有闭包都打印的都是相同的 v，也就是输出 3，原因是在 for 循环结束后，最后 v 的值被设置为了 3，仅此而已。

**正确做法**：
如果想要达到预期的效果，依然是使用万能的再赋值。改写后的代码如下：
```go
for _, v := range []int{1, 2, 3} {
    v := v
    prints = append(prints, func() { fmt.Println(v) })
}
```
增加 v := v 语句，程序输出结果为 1，2，3。


### 场景三：循环中使用goroutine
范例一：
```go
type MyInt int

func (mi *MyInt) Show() {
    fmt.Println(*mi)
}

func main() {
    ms := []MyInt{1, 2, 3, 4, 5}
    for _, m := range ms {
        go m.Show()
        // implicitly converted to `go (&m).Show()`
        // thus creating a reference to loop variable.
        // but you would never know this without more context.
    }
    time.Sleep(100 * time.Millisecond)
}
// prints 5 5 5 5 5
```


范例二：
```go
func main() {
    values := []int{1, 2, 3}
    wg := sync.WaitGroup{}
    for _, val := range values {
        wg.Add(1)
        go func() {
            fmt.Println(val)
            wg.Done()
        }()
    }
    wg.Wait()
}
// 输出结果
// 3
// 3
// 3
```

**分析**
对于主协程来讲，循环是很快就跑完的，而这个时候各个协程可能才开始跑，此时`val`的值已经遍历到最后一个了，所以各协程都输出了`3`。（如果遍历数据庞大，主协程遍历耗时较久的话，goroutine的输出会根据当时候的`val`的值，所以每次的输出结果不一定相同的。）

>**不使用goroutine，结果是符合预期的**
```go
# 使用 goroutine
func TestIterator6(t *testing.T) {
    for i := 0; i < 3; i++ {
        go func() {
            fmt.Println(i)
        }()
    }
 
    time.Sleep(10 * time.Millisecond)
}
 
// 执行结果
3
3
3



# 不使用 goroutine
func TestIterator9(t *testing.T) {
    for i := 0; i < 3; i++ {
        func() {
            fmt.Println(i)
        }()
    }
}
// 执行结果
0
1
2
```


**正确做法**：
- 使用临时变量
```go
for _, val := range values {
    wg.Add(1)
    val := val
    go func() {
        fmt.Println(val)
        wg.Done()
    }()
}
```

- 使用闭包
> 其实此中也是使用的临时变量。因为将迭代器变量作为实参，赋值给形参，形参就是临时变量。
```go
for _, val := range values {
    wg.Add(1)
    go func(val int) {
        fmt.Println(val)
        wg.Done()
    }(val)
}
```

### 场景四：循环中使用defer
**现象**
```go
func TestIterator10(t *testing.T) {
    for i := 0; i < 3; i++ {
        defer func() {
            fmt.Println(i)
        }()
    }
}
 
// 执行结果
3
3
3
```
**原理**  
加defer，遍历完了才执行，匿名函数中变量已经被修改了。


**解法1**
```go
func TestIterator11(t *testing.T) {
    for i := 0; i < 3; i++ {
        i := i
        defer func() {
            fmt.Println(i)
        }()
    }
}
 
// 执行结果
2
1
0
```

**解法2**
```go
func TestIterator12(t *testing.T) {
    for i := 0; i < 3; i++ {
        defer func(i int) {
            fmt.Println(i)
        }(i)
    }
}
 
// 执行结果
2
1
0
```

问题的本质是：==在defer的函数体中使用了for的迭代参数i.==

解决：不在defer的函数体中使用for 迭代参数i；在函数的参数中使用for迭代参数，或 迭代参数i 赋值给临时值i2。

## 解决方案
### 临时变量
```go
for _, a := range alarms {
    a := a
    go a.Monitor(b)
}
```
强制拷贝变量，把 per loop 的循环变量变成 per iteration的。

### `GOEXPERIMENT=loopvar`
问题的本质是 golang 设计之初，决定将循环变量设定为 per loop 的而不是 per iteration 的。想要根除这个问题，需要在语义层面修复。即将循环变量设定为 per iteration。

在Golang进入Go1版本之前就已经讨论过这个问题，当时的结论是：虽然很烦但是问题没有大到要改。但是过去这个 decade 已经展示出来当前语义设计的后果。

在 Go1.21 的新版本起，我们可以开启 `GOEXPERIMENT=loopvar` 来构建 Go 程序。
在 Go 1.22 中，我们计划更改 for 循环，使这些变量具有每次迭代的作用域，而不是每次循环的作用域。


来体验上面提到的 for 循环变量的问题。
构建命令：
```go
GOEXPERIMENT=loopvar go install my/program
GOEXPERIMENT=loopvar go build my/program
GOEXPERIMENT=loopvar go test my/program
GOEXPERIMENT=loopvar go test my/program -bench=.
```
我们对应到上述的第二个例子，程序的运行结果将发生如下改变：
```go
$ go run demo.go                        
3
3
3
$ GOEXPERIMENT=loopvar gotip run demo.go
1
2
3
```
以后就不再需要写 v := v 语句了。
# 参考
```c
# Go的循环遍历使用小坑
https://talkgo.org/t/topic/1734

# Go 迭代变量的陷阱
http://niliu.me/articles/4078.html
```