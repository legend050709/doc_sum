```table-of-contents
```
# 拷贝场景
## 函数中变量赋值
## 函数参数
## 函数返回值
## 函数接收者

# 数据的拷贝
## 数据类型
Go中的数据可以分为值类型和引用类型，值类型存储的是具体的值，引用类型存储的是指向某个地址的指针。

如下图所示，变量`a`代表的`地址A`中存储的就是`值111`，存储的是值本身，所以通过变量能直接拿到值。变量`b`代表的`地址B`中不是直接存储的值，而是存储的`一个指针（指向地址A）`，通过`地址B`能间接拿到`地址A`中存储的值
![](attachments/Pasted%20image%2020240207114313.png)

先说结论，**Go里面没有`引用传递`，Go语言是`值传递`**。区别在于数据类型，即引用类型的值传递还是非引用类型的值传递。

### 值类型
值类型：基本数据类型（int/string/bool），数组，结构体(struct)等。
其拷贝为值拷贝，**准确来说，应该是非引用类型的值拷贝**。

#### 值传递
指在调用函数时将实际参数复制一份传递到函数中，这样在函数中如果对参数进行修改，将不会影响到实际参数。

### 引用类型
引用类型：map，slice，channel，func，interface、指针 等。
其拷贝为引用拷贝，**准确来说，应该是引用类型的值拷贝**。

#### 引用传递
指在调用函数时将实际参数的地址直接传递到函数中，那么在函数中对参数所进行的修改，将影响到实际参数。

而Go语言中的一些让你觉得它是`引用传递`的原因，是因为Go语言有`值类型`和`引用类型`，但是它们都是`值传递`。

#### 浅拷贝与深拷贝
浅拷贝与深拷贝的不同在于对引用类型的处理，浅拷贝遇到引用类型时，会复制指针，而深拷贝遇到引用类型时，会复制引用类型指向的具体的值。

## 注意

注意：切片在一定条件下也是值拷贝。  
注意：针对结构体类型的变量，如果里面有指针字段。发生了拷贝，新变量的指针字段和源变量的指针字段指向相同的地址空间。  
注意：如果结构体中有锁的话，记得重新初始化一个锁对象，否则会出现非预期的死锁。
```go
type User struct {
     mut sync.Mutex
     name string
 }

 func main() {
     u1 := &User{name: "test"}
     u1.Lock()
     defer u1.Unlock()
     tmp := *u1
     u2 := &tmp
     // u2.Mutex = sync.Mutex{} // 没有这一行就会死锁
     fmt.Printf("%#p\n", u1)
     fmt.Printf("%#p\n", u2)
     u2.Lock()
     defer u2.Unlock()
 }

$ go run test.go
c00000c060
c00000c080
fatal error: all goroutines are asleep - deadlock!

可以使用go vet test.go检查这个问题

```

# 非引用类型的值拷贝
## struct的拷贝
```go
package main  
  
import "fmt"  
  
type person struct {  
    name string  
}  
  
func (p person) modify() {  
    p.name = "jacky"  
}  
  
func modify(p person) {  
    p.name = "jacky"  
}  
  
func main() {  
    p := person{"larry"}  
    p.modify()  
    // modify(p)  
    fmt.Println(p)  
}
```

## 数组的拷贝
```go
package main  
  
import "fmt"  
  
func modify(a [3]int) {  
    a[0] = 4  
}  
  
func main() {  
    a := [3]int{1, 2, 3}  
    modify(a)  
    fmt.Println(a)  
}
这种情况是值拷贝，返回[1, 2, 3]

```

# 引用类型的值拷贝

## map的拷贝
```go
func main() {
    users := make(map[int]string)
    users[1] = "user1"

    fmt.Printf("before modify: user:%v\n", users[1])  // before modify: user:user1
    modify(users)
    fmt.Printf("after modify: user:%v\n", users[1])  // after modify: user:user2
}

func modify(u map[int]string) {
    u[1] = "user2"
}
```

通过查看源码我们可以看到，实际上`make`底层调用的是`makemap`函数，主要做的工作就是初始化`hmap`结构体的各种字段
```go
func makemap(t *maptype, hint int64, h *hmap, bucket unsafe.Pointer) *hmap {
    //...
}
```
通过查看`src/runtime/hashmap.go`源代码发现，`make`函数返回的是一个`hmap`类型的指针`*hmap`。也就是说`map===*hmap`。 现在看`func modify(p map)`这样的函数，其实就等于`func modify(p *hmap)`，相当于传递了一个指针进来。

## chan类型
`chan`类型本质上和`map`类型是一样的，这里不做过多的介绍，参考下源代码:
```go
func makechan(t *chantype, size int64) *hchan {
    //...
}
```
`chan`也是一个引用类型，和`map`相差无几，`make`返回的是一个`*hchan`。

## slice的拷贝
map和chan使用make函数返回的实际上是 `*hmap`和`*hchan`指针类型，也就是指针传递。
slice虽然也是引用类型，但是它又有点不一样。
简单来说就是，slice本身是个结构体，但它内部第一个元素是一个指针类型，指向底层的具体数组。
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
slice在传递时，形参是拷贝的实参这个slice，但他们底层指向的数组是一样的，拷贝slice时，其内部指针的值也被拷贝了，也就是说指针的内容一样，都是指向同一个数组。

### 范例一
```go
func main() {
    arr := make([]int, 0)
    arr = append(arr, 1, 2, 3)
    fmt.Printf("outer1: %p, %p\n", &arr, &arr[0])
    modify(arr)
    fmt.Println(arr)  // 10, 2, 3
}

func modify(arr []int) {
    fmt.Printf("inner1: %p, %p\n", &arr, &arr[0])
    arr[0] = 10
    fmt.Printf("inner2: %p, %p\n", &arr, &arr[0])
}

//输出：
//outer1: 0x14000112018, 0x14000134000
//inner1: 0x14000112030, 0x14000134000
//inner2: 0x14000112030, 0x14000134000
//[10 2 3]
```

因为`slice`是引用类型，指向的是同一个数组。
可以看到，在函数内外，arr本身的地址`&arr`变了，但是两个指针指向的底层数据，也就是`&arr[0]`数组首元素的地址是不变的。

### 范例二
再来看另外一个稍微复杂的例子，函数内部使用`append`。这个会稍微不一样。
```go
func main() {
    arr := make([]int, 0)
    //arr := make([]int, 0, 5)
    arr = append(arr, 1, 2, 3)
    fmt.Printf("outer1: %p, %p, len:%d, capacity:%d\n", &arr, &arr[0], len(arr), cap(arr))
    //modify(arr)
    appendSlice(arr)
    fmt.Printf("outer2: %p, %p, len:%d, capacity:%d\n", &arr, &arr[0], len(arr), cap(arr))
    fmt.Println(arr)
}

func appendSlice(arr []int) {
    fmt.Printf("inner1: %p, %p, len:%d, capacity:%d\n", &arr, &arr[0], len(arr), cap(arr))
    //modify(arr)
    arr = append(arr, 1)
    fmt.Printf("inner2: %p, %p, len:%d, capacity:%d\n", &arr, &arr[0], len(arr), cap(arr))
    //modify(arr) //&arr[0]的地址是否相等，取决于初始化slice的时候的capacity是否足够
}
```

分两种情况:
#### 情况一：make slice的时候没有分配足够的capacity
`arr := make([]int, 0)`；像这种写法，那么输出就是：
```text
outer1: 0x14000114018, 0x1400012e000, len:3, capacity:3
inner1: 0x14000114030, 0x1400012e000, len:3, capacity:3
inner2: 0x14000114030, 0x1400012c060, len:4, capacity:6
outer2: 0x14000114018, 0x1400012e000, len:3, capacity:3
[1 2 3]
```
![](attachments/Pasted%20image%2020240207115957.png)

1. outer1: 外部传入一个`slice`，引用类型，值传递。
2. inner1: 由于是值传递，所以arr的地址`&arr`变了，但是两个arr指向的底层数组首元素`&arr[0]`，也就是`array unsafe.Pointer`。
3. inner2: 在内部调用`append`后，由于`cap容量`不够，所以扩容，`cap=cap*2`，重新在新的地址空间分配底层数组，所以数组首元素的地址改变了。
4. 回到函数外部，外部的slice指向的底层数组为原数组，内部的修改不影响原数组。

#### 情况二： make slice的时候分配足够的capacity
`arr := make([]int, 0, 5)`；像这种写法，那么输出就是：

```text
outer1: 0x1400000c030, 0x1400001c050, len:3, capacity:5
inner1: 0x1400000c048, 0x1400001c050, len:3, capacity:5
inner2: 0x1400000c048, 0x1400001c050, len:4, capacity:5
outer2: 0x1400000c030, 0x1400001c050, len:3, capacity:5
[1 2 3]
```
虽然函数内部`append`的结果同样不影响外部的输出，但是原理却不一样。
![](attachments/Pasted%20image%2020240207120728.png)
不同点：

1. 在内部调用`append`的时候，由于`cap 容量`足够，所以不需要扩容，在原地址空间增加一个元素，底层数组的首元素地址相同。
2. 回到函数外部，打印出来还是`[1 2 3]`,是因为外层的`len`是3，所以只能打印3个元素，实际上第四个元素的地址上已经有数据了。只不过因为`len`为3，所以我们无法看到第四个元素。

那正确的append应该是怎么样的呢：
```text
appendSlice(&arr)

func appendSlice(arr *[]int) {
    *arr = append(*arr, 1)
}
```
传指针进去，这样拷贝的就是这个指针，指针指向的对象，也就是slice本身，是不变的，我们使用`*arr`可以对slice进行操作。


## 小结
- Go里面没有`引用传递`，Go语言是`值传递`。
- 如果需要函数内部的修改能影响到函数外部，那么就传指针。
- map/channel本身就是指针，是引用类型，所以直接传map和channel本身就可以。
- slice的赋值操作其实是针对slice结构体内部的指针进行操作，也是指针，可以直接传slice本身。
- slice的append操作同时需要修改结构体的`len/cap`，类似于struct，如果需要传递到函数外部，需要传指针。（或者使用函数返回值）

# 参考
```bash
# Golang是值传递还是引用传递
https://zhuanlan.zhihu.com/p/542218435

# go的值类型和引用类型1——传递和拷贝
https://studygolang.com/articles/33388?hmsr=joyk.com&utm_source=joyk.com&utm_medium=referral
```