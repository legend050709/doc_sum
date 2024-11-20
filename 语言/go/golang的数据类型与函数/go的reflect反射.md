```table-of-contents
```
# 背景
空接口`interface{}` 虽然能保存任意的值（即任意类型的实参传递给空接口类型的形参），但也带来了一个问题：一个空的接口会隐藏值对应的类型，以及其中的所有的公开的方法。
因此只有我们知道具体的动态类型才能使用类型断言来访问内部的值；
如果我们事先不知道空接口指向的值的具体类型，我们可能就束手无策了。

这个时候我们想要知道一个接口类型的变量具体是什么（什么类型），有什么能力（有哪些方法），就需要一面“镜子”能够反射（`reflect`）出这个变量的具体内容。在Go语言中也正好有这样的工具——`reflect`。

# 反射的作用

Go语言提供了一种机制，在运行时可以**「更新和检查变量的值、调用变量的方法和变量支持的内在操作」**，但是在**「编译时并不知道这些变量的具体类型」**，这种机制被称为反射。

反射有何用：
- 上面我们提到空接口，它能接收任何东西
- 但是怎么来判断空接口变量存储的是什么类型呢？上面介绍的类型断言可以实现
- 如果想获取存储变量的类型信息和值信息就需要使用到反射。


因此，**反射就是可以动态获取变量类型信息和值信息的机制**。

```go
package main
import (
  "fmt"
  "reflect"
)
func main() {
  var name string = "微客鸟窝"
  // TypeOf会返回变量的类型，比如int/float/struct/指针等
 reflectType := reflect.TypeOf(name)

 // valueOf返回变量的的值，此处为"微客鸟窝"
 reflectValue := reflect.ValueOf(name)

 fmt.Println("type: ", reflectType) //type:  string
 fmt.Println("value: ", reflectValue) //value:  微客鸟窝
}
```


# 反射的实现

## 接口的定义

在golang中，interface也是一个结构体，记录了2个指针：

- 指针1，指向该变量的类型
- 指针2，指向该变量的value

如下，空接口的结构体就是上述2个指针，第一个指针的类型是`type rtype struct`；
；非空接口由于需要携带的信息更多(例如该接口实现了哪些方法)，所以第一个指针的类型是itab，在itab中记录了该变量的动态类型: `typ *rtype`。

```golang
// emptyInterface is the header for an interface{} value.
type emptyInterface struct {
    typ  *rtype
    word unsafe.Pointer
}

// nonEmptyInterface is the header for a interface value with methods.
type nonEmptyInterface struct {
    // see ../runtime/iface.go:/Itab
    itab *struct {
        ityp   *rtype // static interface type
        typ    *rtype // dynamic concrete type
        link   unsafe.Pointer
        bad    int32
        unused int32
        fun    [100000]unsafe.Pointer // method table
    }
    word unsafe.Pointer
}
```

## reflect 包

反射是由reflect包来提供支持的，它提供两种类型来访问接口变量的内容,即Type 和 Value。reflect包提供了两个函数来获取任意对象的Type 和 Value：

```go
1. func TypeOf(i interface{}) Type
获取变量的类型信息，如果为空则返回「nil」

1. func ValueOf(i interface{}) Value
获取数据的值，如果为空则返回「0」
```

### TypeOf
函数 TypeOf 的返回值 reflect.Type 实际上是一个接口，定义了很多方法来获取类型相关的信息：



### ValueOf


# 参考
```c
# 深入理解 interface和reflect
https://turling.me/2019/10/13/Golang-series-interface-and-reflect/

# 图解go反射实现原理
https://i6448038.github.io/2020/02/15/golang-reflection/

# 深入理解Golang的reflect原理
https://kunkkawu.com/archives/shen-ru-li-jie-golang-de-reflect-yuan-li
http://liangjf.top/2020/09/22/161.go-reflect%E5%8F%8D%E5%B0%84/

# 【Golang】反射的三大laws
http://www.randyfield.cn/post/2022-07-08-the-laws-of-reflect/
https://juejin.cn/post/7183132625580605498

# 深入理解 go reflect - 反射为什么慢
https://juejin.cn/post/7186859098661453884
```