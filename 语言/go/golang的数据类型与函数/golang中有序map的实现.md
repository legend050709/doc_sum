```table-of-contents
```
# 背景
## golang内嵌map的特点
### map是无序的
在Go语言 中的 map 是无序的，这意味着无法保证遍历 map 时的顺序与元素添加的顺序一致。

```go
package main  
  
import "fmt"  
  
func main() {  
    // 创建一个 map  
    myMap := make(map[string]int)  
  
    // 向 map 中添加键值对  
    myMap["apple"] = 10  
    myMap["banana"] = 5  
    myMap["orange"] = 7  
  
    // 遍历 map，并打印键值对  
    for key, value := range myMap {  
        fmt.Println(key, ":", value)  
    }  
}
```

每次运行此程序时，输出的键值对的顺序可能会不同，因为 map 是无序的。例如，一次运行的输出可能是：
```go
banana : 5  
apple : 10  
orange : 7
```
而另一次运行的输出可能是：
```go
orange : 7  
apple : 10  
banana : 5
```
### map是非线程安全的
多个协程对于map并发操作，会导致panic。

# map的无序性
Go 的 map 本质上是一个哈希表，因此它并不会像数组那样有固定顺序。换句话说，map 在存储数据时会自动打乱顺序，这也就是为什么你每次遍历它时，得到的顺序不一样。

这种无序性在很多时候没啥大问题，因为你往往只是需要根据键来快速取值，而不在乎顺序。然而，到了要输出结果的时候，比如要根据字母顺序列出某些项目，这时就有点尴尬了。


# 思路



## 基于key的顺序遍历
如果我们想要按照 key 的顺序获取 map 的值，需要先取出所有的 key 进行排序，然后按照这个排序的 key 依次获取对应的值。



## 基于插入顺序遍历

如果我们想要保证元素有序，比如按照元素插入的顺序进行遍历，可以使用辅助的数据结构，比如orderedmap，来记录插入顺序。

# 内嵌map + 有序结构
## 思路

思路很简单：把所有的键捞出来，扔进一个可以排序的容器，然后按顺序遍历这些键，再根据键来取对应的值。

## 方法
slice 是有序的，可以采用 **map+ slice**的方式来组织元素：
在Go语言中，map是无序的，每次迭代map的顺序可能不同。如果需要按特定顺序遍历map，可以采用以下步骤：

- 创建一个切片来保存map的键。
- 遍历map，将键存储到切片中。
- 对切片进行排序。
不论你想升序还是降序，都可以使用 Go 自带的 sort 包来帮忙。
- 根据排序后的键顺序，遍历map并访问对应的值。

## 范例

以下是一个示例代码，展示如何按键的升序遍历map：

```go
package main  
  
import (  
    "fmt"  
    "sort"  
)  
  
func main() {  
    m := map[string]int{  
        "b": 2,  
        "a": 1,  
        "c": 3,  
    }  
  
    keys := make([]string, 0, len(m))  
  
    for k := range m {  
        keys = append(keys, k)  
    }  
  
    sort.Strings(keys)  
  
    for _, k := range keys {  
        fmt.Println(k, m[k])  
    }  
}
```

这里使用的是升序排序，如果需要降序排序，可以使用如下方法进行排序：
```go
sort.Sort(sort.Reverse(sort.StringSlice(keys)))
```

## 应用场景
在实际开发中，这个方法还是挺有用的。

比如你在做 Web 应用时，可能需要从数据库或者其他数据源中取出一堆信息，而这些信息存放在 map 里。

如果你想按某个字段的字母顺序显示给用户看，这个时候上面的排序方案就能派上用场。

或者在一些需要生成报表的场景中，数据的顺序可能也很重要。map 的无序特点虽然在大多数情况下无伤大雅，但在这些特定场景下，显然有序的输出更为关键。

再比如，有时候你会希望按时间顺序输出事件日志，虽然时间戳可以作为值存储在 map 中，但如果你想按时间从早到晚排序输出日志，那么通过上面的手动排序就可以实现这样的需求。



# orderedmap

## 背景
关于有序map，很自然的就能够想到需要额外使用一个链表或者数组来记录key写入的顺序。go的标准库中没有提供，这里推荐一个用起来感觉不错的第三方的有序map：
```go
github.com/iancoleman/orderedmap
go get github.com/iancoleman/orderedmap
```


# 参考
```bash
# go orderedmap 有序map
https://juejin.cn/post/7057389822477860900
```