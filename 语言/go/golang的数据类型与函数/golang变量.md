```table-of-contents
```
# 变量的申明与赋值
## 变量的申明然后赋值
在Go中，**必须先声明变量，再赋值或使用变量**。最复杂的声明+赋值操作为：
```go
package main

import ( "fmt" )

func main(){
    var x int
    x=10
    fmt.Println("x =",x)
}

```
此处声明了一个变量x，其数据类型为`int`。默认情况下，Go在变量的声明期间会为其做初始化赋值：int类型初始化赋值为0，booleans初始化赋值为false，strings初始化赋值为""，等等。

## 变量的申明并同时赋值
可以将声明和赋值操作合并：
```go
var x int = 10
```

还有一种更方便的声明+赋值方式：
```go
x := 10
```

### `:=`的使用
`:=`在Go中属于类型推断操作，它包含了变量声明和变量赋值两个过程。
需要注意的是，变量声明之后不能再次声明(除非在不同的作用域)，之后只能使用`=`进行赋值。

例如，执行下面的代码将报错：
```go
package main

import ("fmt")

func main(){
	x:=10
	fmt.Println("x =",x)
	x:=11
	fmt.Println("x =",x)
}

错误如下所示：
# command-line-arguments
.\test.go:8:3: no new variables on left side of :=


分析：
报错信息很明显，`:=`左边没有新变量。
```

如果仔细看上面的报错信息，会发现`no new variables`是一个复数。实际上，Go允许我们使用`:=`一次性声明、赋值多个变量，而且只要左边有任何**一个**新变量，语法就是正确的。
```go
func main(){
    name,age := "longshuai",23
    fmt.Println("name:",name,"age:",age)
    
    // name重新赋值，因为有一个新变量weight
    weight,name := 90,"malongshuai"
    fmt.Println("name:",name,"weight:",weight)
}
```

需要注意，name第二次被`:=`赋值，Go第一次推断出该变量的数据类型之后，就不允许`:=`再改变它的数据类型，因为只有第一次`:=`对name进行声明，之后所有的`:=`对name都只是简单的赋值操作。

例如，下面将报错：
```go
weight,name := 90,80

错误如下所示：
.\test.go:11:14: cannot use 80 (type int) as type string in assignment
```

## 一次性申明多个变量
可以一次性声明、声明并赋值多个变量：
```go
var x, y, z int

x, y, z := 10, 20, 30

var (
	x = 10
	y = 20
	z = 30
)

```



# 参考
```bash

```