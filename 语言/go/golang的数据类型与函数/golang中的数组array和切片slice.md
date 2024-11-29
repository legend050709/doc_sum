```table-of-contents
```
# 切片slice
## 背景
Go 数组的长度不可改变，在特定场景中这样的集合就不太适用，Go 中提供了一种灵活，功能强悍的内置类型切片("==动态数组==")，与数组相比切片的长度是不固定的，可以追加元素，在追加时，可使切片的容量增大。

## 底层数据结构
切片底层的数据结构（src/runtime/slice.go）
```go
type slice struct {
    array unsafe.Pointer
    len   int
    cap   int
}
```
## 操作
### 声明
可以声明一个未指定大小的数组来定义切片：

```go
var s []byte
```
这种声明的切片变量，默认值是nil，容量和长度默认都是0。

### make创建
使用内置函数`make`可以指定**长度(len)**和 **容量(cap)** 来创建。
其中 **cap** 为可选参数, 省略了 cap，表示 cap 和 len 相等。
len 是数组的长度，并且也是切片的初始长度。
```go
func make([]T, len, cap) []T

s := make([]int, 5) // 分配int切片,长度和容量都是5 
s := make([]string, 3, 5) // 分配string切片,长度3,容量5
```

比如：
```go
var slice1 []type = make([]type, len)

或者
slice1 := make([]type, len)

或者
var slice1 []type
slice1 = make([]type, len)
```

#### make的作用
**总结来说**:
make主要用于==为slice分配内存,设置其长度和容量,并返回一个已经初始化的slice值==。
这对slice的使用很重要。只有通过make初始化过后,slice才可以直接使用其内存来存储元素。

####  make中的 len 和 cap 的区别
`length`:决定当前slice可以存储的元素个数,也就是说可以通过索引 `0 ~ length-1` 来访问元素。
`capacity`: 决定当前分配的底层数组可以存储的最大元素个数。

length 必须小于等于 capacity。也就是当前slice实际存储的元素不可以超过分配的最大容量。
一般来说,`length = capacity`。这种情况slice使用的底层数组刚刚好分配了需要的内存。
但也可以 `length < capacity`。这种情况表示分配的内存比实际需要的大,为后面的内容增加提供了动态扩展的空间。

当通过append等操作增加元素时,如果当前`length == capacity`,会自动的`resize`底层数组,增加实际分配的内存大小。
capacity 也决定了slice可以通过 append 扩展的范围。只有在 `length < capacity `时,append 才可以继续扩展 slice。

**总结来说**
length 决定了当前slice可访问的元素个数；
capacity 决定了底层数组的最大容量,也决定了slice通过append能扩展的范围。 
一般`length==capacity`时性能最优；
但`length<capacity`可以提高动态扩展的性能。 
两者规定了slice当前的状态和未来的增长能力。

### 声明并初始化
```go
s :=[] int {1,2,3 } 
```
直接初始化切片，[] 表示是切片类型，**{1,2,3}** 初始化值依次是 1,2,3，其 **cap=len=3**。

```go
创建无容量的切片
s := make([]int, 0, 0) 
等价于
s=[]int{} 
```


### 获取长度len
切片是可索引的，并且可以由 len() 方法获取长度。

### 获取容量cap
切片提供了计算容量的方法， cap() 可以测量切片最长可以达到多少。

### append 追加
### copy拷贝
### 切片截取

## 切片的容量扩展
### 小结
 slice 的数据结构，很简单，一个指向真实 array 地址的指针 ptr ，slice 的长度 len 和容量 cap 。
 ![](attachments/Pasted%20image%2020241122121356.png)
![](attachments/Pasted%20image%2020241122121334.png)

（1）每次cap改变的时候，指向array内存的指针都在变化。
（2）在实际使用中，我们最好事先预期好一个cap，这样在使用append的时候，可以避免反复重新分配内存复制之前的数据，减少不必要的性能消耗。

## 切片删除元素
Go 并没有提供删除切片元素专用的语法或函数，需要使用切片本身的特性来删除元素。

### 错误的删除
将指定索引位置直接置为默认值。
```go

func main() {
  nums := []int{1, 2, 3, 4}

  nums[2] = 0 // 删除第3个元素  

  fmt.Println(nums) // [1 2 0 4]
}
```

### 最后一个元素迁移的方法删除
**方法**：
先用最后一个元素覆盖要删除的元素,然后返回减少 1 长度的切片。

**问题**：
slice本身是有序的，此中删除某个指定位置的元素，将最后一个元素填补该位置，改变了slice的有序性。

```go
func removeElement(slice []int, i int) []int {
  if len(slice) == 0 ||  len(slice) <= i {
	  return slice
  }
  slice[i] = slice[len(slice)-1]
  return slice[:len(slice)-1]
}

```

比如删除索引为 2 的元素:
```go
data := []int{0, 1, 2, 3, 4}
data = removeElement(data, 2) // [0, 1, 4, 3]
```

### 截取法（修改原切片）
**方法**：
利用对 slice 的截取删除指定元素。

```go
// DeleteSlice1 删除指定元素。
func DeleteSlice1(a []int, elem int) []int {
    for i := 0; i < len(a); i++ {
        if a[i] == elem {
            a = append(a[:i], a[i+1:]...)
            i--
        }
    }
    return a
}

注意删除时，后面的元素会前移，所以下标 i 应该左移一位。
```

### 拷贝法（不改原切片）
**方法**：
重新使用一个 slice，将要删除的元素过滤掉。

**问题**：
是需要开辟另一个 slice 的空间，优点是容易理解，而且不会修改原 slice。

```go
// DeleteSlice2 删除指定元素。
func DeleteSlice2(a []int, elem int) []int {
    tmp := make([]int, 0, len(a))
    for _, v := range a {
        if v != elem {
            tmp = append(tmp, v)
        }
    }
    return tmp
}
```


### 移位法（修改原切片）
#### 方式一
利用一个下标 index，记录下一个有效元素应该在的位置。遍历所有元素，当遇到有效元素，将其移动到 index 且 index 加一。
该方法可以看成对第一种方法截取法的改进，因为每次指需移动一个元素，性能更加。
```go
// DeleteSlice3 删除指定元素。
func DeleteSlice3(a []int, elem int) []int {
    j := 0
    for _, v := range a {
        if v != elem {
            a[j] = v
            j++
        }
    }
    return a[:j]
}
```

#### 方式二
创建了一个 slice，但是共用原始 slice 的底层数组。这样也不需要额外分配内存空间，直接在原 slice 上进行修改。

```go
// DeleteSlice4 删除指定元素。
func DeleteSlice4(a []int, elem int) []int {
    tgt := a[:0]
    for _, v := range a {
        if v != elem {
            tgt = append(tgt, v)
        }
    }
    return tgt
}
```

### 性能对比
生成切片函数如下：
```go
func getSlice(n int) []int {
    a := make([]int, 0, n)
    for i := 0; i < n; i++ {
        if i%2 == 0 {
            a = append(a, 0)
            continue
        }
        a = append(a, 1)
    }
    return a
}
```

基准测试代码如下：
```go
func BenchmarkDeleteSlice1(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = DeleteSlice1(getSlice(10), 0)
    }
}
func BenchmarkDeleteSlice2(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = DeleteSlice2(getSlice(10), 0)
    }
}
func BenchmarkDeleteSlice3(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = DeleteSlice3(getSlice(10), 0)
    }
}
func BenchmarkDeleteSlice4(b *testing.B) {
    for i := 0; i < b.N; i++ {
        _ = DeleteSlice4(getSlice(10), 0)
    }
}
```

测试结果：
```go
(1)原切片长度为 10：
go test -bench=. main/slice
goos: windows
goarch: amd64
pkg: main/slice
cpu: Intel(R) Core(TM) i7-9700 CPU @ 3.00GHz
BenchmarkDeleteSlice1-8         17466486                65.07 ns/op
BenchmarkDeleteSlice2-8         14897282                85.22 ns/op
BenchmarkDeleteSlice3-8         21952129                50.78 ns/op
BenchmarkDeleteSlice4-8         22176390                54.68 ns/op
PASS
ok      main/slice      5.427s

(2)原切片长度为 100：
BenchmarkDeleteSlice1-8          1652146               762.1 ns/op
BenchmarkDeleteSlice2-8          2124237               578.4 ns/op
BenchmarkDeleteSlice3-8          3161318               359.9 ns/op
BenchmarkDeleteSlice4-8          2714158               423.7 ns/op

(3)原切片长度为 1000：
BenchmarkDeleteSlice1-8            56067             21915 ns/op
BenchmarkDeleteSlice2-8           258662              5007 ns/op
BenchmarkDeleteSlice3-8           432049              2724 ns/op
BenchmarkDeleteSlice4-8           325194              3615 ns/op
```

分析：
```text
从基准测试结果来看，性能最佳的方法是移位法，其中又属第一种实现方式较佳。
性能最差的也是最常用的方法是截取法。随着切片长度的增加，上面四种删除方式的性能差异会愈加明显。

实际使用时，我们可以根据不用场景来选择。
如不能修改原切片, 使用拷贝法;
可以修改原切片使用移位法中的第一种实现方式。
```

## 空(nil)切片

一个切片在未初始化之前默认为 nil，长度为 0。

测试一：
```go
package main

import "fmt"
func main() {
   var numbers = make([]int,3,5)

   printSlice(numbers)
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}

运行结果：
len=3 cap=5 slice=[0 0 0]
```

测试二：
```go
package main

import "fmt"
func main() {
   var numbers []int

   printSlice(numbers)

   if(numbers == nil){
      fmt.Printf("切片是空的")
   }
}

func printSlice(x []int){
   fmt.Printf("len=%d cap=%d slice=%v\n",len(x),cap(x),x)
}

结果：
len=0 cap=0 slice=[]
```



# 参考
```bash
# Go语言Array和Slice的区别
https://juejin.cn/post/6920807885228212231

# Go切片删除元素错过这篇你就out了
https://developer.aliyun.com/article/1353542
```