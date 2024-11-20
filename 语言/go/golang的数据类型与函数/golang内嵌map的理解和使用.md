```table-of-contents
```
# map的特点
## map的key的类型需要是可比较的
**key 的 类型 必须是可比较的**（comparable），也就是可以通过 == 和 != 操作符进行比较；
value 的值和类型无所谓，可以是任意的类型，或者为 nil。

在 Go 语言中，bool、整数、浮点数、复数、字符串、指针、Channel、接口都是可比较的，包含可比较元素的 struct 和数组，这俩也是可比较的，而 ==slice、map、函数值都是不可比较==的。

那么，上面这些可比较的数据类型都可以作为 map 的 key 吗？显然不是。==通常情况下，我们会选择内建的基本类型，比如整数、字符串做 key 的类型==，因为这样最方便。

## map的value的类型是任意的

value 可以是任意类型的；
通过使用空接口类型，我们可以存储任意值，但是使用这种类型作为值时需要先做一次类型断言。

### 函数作为map的值
map 也可以用函数作为自己的值。
#### 使用场景

#### 范例

```go
package main
import "fmt"

func main() {
    mf := map[int]func() int{
        1: func() int { return 10 },
        2: func() int { return 20 },
        5: func() int { return 50 },
    }
    fmt.Println(mf)
}

输出结果为：map[1:0x10903be0 5:0x10903ba0 2:0x10903bc0]: 整形都被映射到函数地址。
```

注：函数类型 不带有 {} ; 如果带有{} 则是函数的实现；如果 {} 后含有 (), 则是 函数的调用。

### slice 作为map的值
#### 使用场景
既然一个 key 只能对应一个 value，而 value 又是一个原始类型，那么如果一个 key 要对应多个值怎么办？

例如，当我们要处理unix机器上的所有进程，以父进程（pid 为整形）作为 key，所有的子进程（以所有子进程的 pid 组成的切片）作为 value。通过将 value 定义为 `[]int` 类型或者其他类型的切片，就可以优雅的解决这个问题。

#### 范例




## map是无序的
map 是无序的，所以当遍历一个 map 对象的时候，迭代的元素的顺序是不确定的，无法保证两次遍历的顺序是一样的，也不能保证和插入的顺序一致。

### 处理

1》如果我们想要按照 key 的顺序获取 map 的值，需要先取出所有的 key 进行排序，然后按照这个排序的 key 依次获取对应的值。

2》如果我们想要保证元素有序，比如按照元素插入的顺序进行遍历，可以使用辅助的数据结构，比如orderedmap，来记录插入顺序。

## map是非线程安全的

Go 内建的 map 对象不是线程（goroutine）安全的，并发读写的时候运行时会有检查，遇到并发问题就会导致 panic。

## map是引用类型
map 是 **引用类型** 的： 内存用 make 方法来分配。



# 操作
## map的申明
```go
var map1 map[keytype]valuetype
比如：
var map1 map[string]int
```
（`[keytype]` 和 `valuetype` 之间允许有空格，但是 gofmt 移除了空格）。

```go
// 先声明map
var m1 map[string]string

// 再使用make函数创建一个非nil的map，nil map不能赋值
m1 = make(map[string]string)

// 最后给已声明的map赋值
m1["a"] = "aa"
m1["b"] = "bb"

```


```go
func main() {
	var m map[string]int
	if m == nil {
		fmt.Println("m is nil")
	}

	fmt.Printf("%p\n", m)
}
```

当判断是否为nil时，打印出来了 `m is nil`。
当尝试给nil的map 进行赋值时，会直接崩溃 `m["age"] = 18`, 当执行这段代码时就会panic。

## map创建
可以使用`make`函数创建Map，给map分配内存。


```go
// 直接创建
m2 := make(map[string]string)

// 然后赋值
m2["a"] = "aa"
m2["b"] = "bb"
```

## 申明并初始化map
在==声明时直接创建并初始化==Map：

```go
// 初始化 + 赋值一体化
m3 := map[string]string{
	"a": "aa",
	"b": "bb",
}

```

```go
func main(){
  var m = map[string]int{} //这里的{} 不能少
  m["age"] = 18
  fmt.Pringln(m)
}
```

## 添加和更新元素

```go
m["three"] = 3  // 添加元素 m["one"] = 10   // 更新元素 
```

## 删除元素

使用`delete`函数删除元素：

```go
delete(m, "two") 
```

## 查找元素

```go
value, exists := m["one"] 
if exists { 
	fmt.Println(value) 
} 
```

## 遍历Map

使用`range`关键字遍历Map：

```go
for key, value := range m { 
	fmt.Printf("%s: %d\n", key, value)
} 
```
## map 的长度
使用`len`函数获取Map的长度：
```go
length := len(m) 
fmt.Println(length) 
```

## map的容量

和数组不同，map 可以根据新增的 key-value 对动态的伸缩，因此它不存在固定长度或者最大限制。但是你也可以选择标明 map 的初始容量 `capacity`，就像这样：`make(map[keytype]valuetype, cap)`。

例如：
```go
map2 := make(map[string]float, 100)
```

当 map 增长到容量上限的时候，如果再增加新的 key-value 对，map 的大小会自动加 1。
make初始化map时，默认不标注容量，则容量为0，后续自动扩容。

所以出于性能的考虑，对于大的 map 或者会快速扩张的 map，即使只是大概知道容量，也最好先标明。

## 其他
### new map的理解

如果你错误的使用 new() 分配了一个引用对象，你会获得一个空引用的指针，相当于声明了一个未初始化（未分配内存）的变量并且取了它的地址。

#### 范例
new 方式 `new(Type)` 方式返回的是`*Type`， 返回的是一个指针，这里的指针已经不是空指针了，是有具体的内存地址了

```go
func main() {

	m := new(map[string]int)
	if m == nil {
		fmt.Println("m is nil")
	}

	fmt.Printf("%p\n", m)
	fmt.Println(*m)
}

```
这里判断`m==nil` 失败.

但是依然不能给这个 m 赋值, `(*m)["age"]=18` 依然提示是向nil map 赋值, 需要将 m 指向一个具体的值后才可以正常的赋值.
```go
func main() {

	m := new(map[string]int)
	*m = map[string]int{}

	(*m)["name"] = 18
	fmt.Println(*m)
}
```
#### 总结


对于map类型，**不要使用 new，永远用 make 来构造 map**。
使用new来声明和直接声明map，其实没有什么区别，只不过一个返回的是`(*map)`类型，一个是`(map)`类型。
声明之后，还需要创建map，最好通过make方式，也可以直接通过初始化的方式。

# map的底层实现
## map的基本结构

在Go的底层实现中，`map` 是通过哈希表（Hash Table）来实现的。哈希表通过哈希函数将键（key）映射到表中的一个位置（也称为槽位或桶），从而实现对数据的快速访问。Go的`map`结构大致可以分为以下几个部分：
- **哈希表**：存储键值对的实际容器，由多个桶（bucket）组成。
- **桶（Bucket）**：存储键值对的数组，每个桶可以包含多个键值对（通过链表或红黑树等数据结构连接）。
- **哈希函数**：用于将键映射到哈希表中的特定桶。
- **负载因子（Load Factor）**：表示哈希表的当前填充程度，即已存储的键值对数量与哈希表总容量的比值。



## map 的容量扩展

### 扩容的介绍

在Go语言中，`map` 是一种内置的数据结构，用于存储键值对（key-value pairs）的集合。它提供了高效的查找、插入和删除操作。

与数组或切片不同，`map` 的大小（即其能存储的键值对数量）是动态变化的，这得益于其背后的复杂内存管理机制，特别是其容量扩展策略。

### 容量扩展的触发条件
Go的`map`在运行时根据需要自动扩展其容量，以确保操作的高效性。容量扩展的触发主要基于负载因子的增长。当向`map`中添加新的键值对时，如果添加后的负载因子超过了某个阈值（在Go的当前实现中，这个阈值大约是6.5），则触发容量扩展。

### 容量扩展的步骤

容量扩展的具体过程涉及以下几个步骤：

1. **计算新容量**：通常，新容量是旧容量的两倍（或更大，以适应更大的增长需求），以确保有足够的空间来存储新的键值对，同时减少哈希冲突的概率。
    
2. **重新哈希**：将旧哈希表中的所有键值对重新计算哈希值，并根据新的哈希表大小重新分配到新的桶中。这一步骤是容量扩展中最耗时的部分，因为它需要遍历整个旧哈希表。
    
3. **替换旧哈希表**：一旦所有键值对都被重新分配到新的哈希表中，旧哈希表将被丢弃，新的哈希表成为`map`的当前表示。

### 容量扩展的性能考量

容量扩展虽然保证了`map`的灵活性和可扩展性，但也带来了性能上的开销。这些开销主要体现在以下几个方面：

1. **内存分配**：每次容量扩展都需要分配新的内存空间来存储新的哈希表，这可能导致内存碎片化和额外的内存使用。
    
2. **重新哈希**：重新计算所有键值对的哈希值并重新分配到新的桶中是一个计算密集型的过程，尤其是在`map`包含大量键值对时。
    
3. **临时性能下降**：在容量扩展期间，对`map`的读写操作可能会暂时变慢，因为系统需要同时处理旧哈希表和新哈希表的数据。

### 容量扩展的优化措施

- **（1）渐进式扩容**：在某些情况下，Go的`map`实现可能会采用渐进式扩容策略，即不是一次性将所有键值对重新哈希到新表中，而是逐步迁移，以减少对系统性能的影响。
    
- **（2）智能扩容策略**：根据`map`的使用模式和负载情况，动态调整扩容的时机和幅度，以平衡内存使用和性能开销。



# map的性能

## 查找性能
map 传递给函数的代价很小：在 32 位机器上占 4 个字节，64 位机器上占 8 个字节，无论实际上存储了多少数据。

通过 key 在 map 中寻找值是很快的，比线性查找快得多，但是仍然比从数组和切片的索引中直接读取要慢 100 倍；所以如果你很在乎性能的话还是建议用切片来解决问题。

## 扩容性能

### 问题

和数组不同，map 可以根据新增的 key-value 对动态的伸缩，因此它不存在固定长度或者最大限制。但是你也可以选择标明 map 的初始容量 `capacity`，就像这样：`make(map[keytype]valuetype, cap)`。

例如：
```go
map2 := make(map[string]float, 100)
```

当 map 增长到容量上限的时候，如果再增加新的 key-value 对，map 的大小会自动加 1。
make初始化map时，默认不标注容量，则容量为0，后续自动扩容。

对于map和slice，如果使用过程中需要做频繁的扩容操作，而这部分代码又是热点代码，那么就有必要对其做性能优化。 

### 范例
```go
// alloc.go
func map_alloc() {
    // m := make(map[int]int)     // 1
    m := make(map[int]int, 100)   // 2

    for i := 0; i < 100; i++ {
        m[i] = i
    }
}

func slice_alloc() {
    // s := make([]int, 0)        // 1
    s := make([]int, 0, 100)      // 2
    for i := 0; i < 100; i++ {
        s = append(s, i)
        // fmt.Println("len: ", len(s), "cap: ", cap(s))
    }
}


// alloc_test.go
func BenchmarkMap(b *testing.B) {
    for i := 0; i < b.N; i++ {
        map_alloc()
    }
}

func BenchmarkSlice(b *testing.B) {
    for i := 0; i < b.N; i++ {
        slice_alloc()
    }
}
```

**（1） map 的测试**：
执行 BenchmarkMap 测试，两种情况结果如下： 
1）BenchmarkMap-12 185745 5708 ns/op 5395 B/op 16 allocs/op 
2）BenchmarkMap-12 406057 2946 ns/op 2924 B/op 6 allocs/op
可见：指定初始化容量的写法速度大约快一倍，分配内存少接近一半。

**（2） slice 的测试**：
执行 BenchmarkSlice 测试，结果如下：
1）BenchmarkSlice-12 2624802 452 ns/op 2040 B/op 8 allocs/op
2）BenchmarkSlice-12 20625380 49.9 ns/op 0 B/op 0 allocs/op。
可见：指定恰当的初始化capacity，执行效率上快了将近10倍，而且没有内存分配操作。

**（3）为什么 `slice_alloc` 的第一种写法中，需要做8次内存分配操作呢？**

如果打印出每次 `append` 后的切片长度和容量，就能看出其容量经历了8次变化：0 -> 1 -> 2 -> 4 -> 8 -> 16 -> 32 -> 64 -> 128。而第二种写法，其容量一直是100，保持不变。


### 优化方式

一个常见的优化手段是在变量声明时，若事先知道其后续大小，则可指定其容量，避免在扩充过程中频繁的`resize`或`reallocation`操作。

# map的使用
## map和slice的混用
### map的值为切片
假设我们想获取一个 map 类型的切片，我们必须使用两次 `make()` 函数，第一次分配切片，第二次分配 切片中每个 map 元素。

#### 范例
```go
package main
import "fmt"

func main() {
    // Version A:
    items := make([]map[int]int, 5)
    for i:= range items {
        items[i] = make(map[int]int, 1)
        items[i][1] = 2
    }
    fmt.Printf("Version A: Value of items: %v\n", items)

    // Version B: NOT GOOD!
    items2 := make([]map[int]int, 5)
    for _, item := range items2 {
        item = make(map[int]int, 1) // item is only a copy of the slice element.
        item[1] = 2 // This 'item' will be lost on the next iteration.
    }
    fmt.Printf("Version B: Value of items: %v\n", items2)
}

输出结果：
Version A: Value of items: [map[1:2] map[1:2] map[1:2] map[1:2] map[1:2]]
Version B: Value of items: [map[] map[] map[] map[] map[]]
```

分析：
需要注意的是，应当像 A 版本那样通过索引使用切片的 map 元素。
在 B 版本中获得的项只是 map 值的一个拷贝而已，所以真正的 map 元素没有得到初始化。

### 元素类型为map的切片

#### 范例
```go
package main

import "fmt"

func main() {
    // 值为切片的 map: map[string][]int, 例: ["语文"][100,99]

    // 先创建外层的 map
    a := make(map[string][]int, 2)

    // 再创建内层的切片
    a["语文"] = make([]int, 0, 2)
    // 给切片赋值
    a["语文"] = append(a["语文"], 100, 99)

    a["数学"] = make([]int, 0, 2)
    a["数学"] = append(a["数学"], 97, 93)

    fmt.Println(a)
}

输出结果：
map[数学:[97 93] 语文:[100 99]]
```

### 范例
#### 统计字符串中出现的字符及其所有位置
```go
package main

import (
    "fmt"
)

func main() {
    s := "abcab"

    //创建一个map，键是字符，值是一个切片
    //它存储了每个字符所对应的它在字符串中的所有出现位置的切片
    hash := make(map[rune][]int)

    for i, j := range s {

        if _, ok := hash[j]; ok {
            hash[j] = append(hash[j], i)
        } else {
            //创建切片
            hash[j] = make([]int, 0)
            hash[j] = append(hash[j], i)
        }

    }
    fmt.Println(hash)

}

结果：
map[97:[0 3] 98:[1 4] 99:[2]]
```

#### 统计字符串中出现的每个单词的次数
```go
package main

import (
    "fmt"
    "strings"
)

func main() {

    s := "how do you do"

    // 得到一个切片
    words := strings.Split(s, " ")

    // 遍历这个切片, 如果字母存在, v+1, 如果字母不存在, 创建

    a := make(map[string]int, 10) // 创建一个map 用于接收

    // 遍历 words 切片, 把每个单词拿出来
    for _, word := range words {
        // 在 map 中查找, 如果存在, 计数+1, 如果不存在, 则计数=1
        v, ok := a[word]
        if ok {
            a[word] = v + 1
        } else {
            a[word] = 1
        }
    }

    fmt.Println(a) // map[do:2 how:1 you:1]
}


结果：
map[do:2 how:1 you:1]
```

# map的使用建议
## 避免频繁扩容
如果可能，尽量预估`map`的初始大小，并设置合适的初始容量，以减少因扩容带来的性能开销。可以使用`make(map[KeyType]ValueType, initialCapacity)`来指定初始容量。

## 合理设计键
选择具有良好分布特性的键可以减少哈希冲突，提高`map`的查找和插入效率。

## 注意并发访问

`map`在Go中不是并发安全的，因此在多线程环境下访问`map`时需要加锁或使用其他并发控制机制。

## 性能评估
对于性能敏感的应用，建议通过基准测试来评估`map`操作的实际性能，并根据测试结果调整`map`的使用策略。


# 参考
```bash

```