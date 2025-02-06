```table-of-contents
```
# 条件判断if-else
# 分支选择 switch-case

##  结构

```go
switch simpleStatement; condition {
    case expression1,expression2:
        statements
    case expression3:
        statements
    default:
        statements
    }
```

和 if 语句类似，switch 语句也可以在条件语句之前执行一段简短的表达式（可以用于很方便的控制变量的作用域），switch case 开始执行时，会先执行这一个表达式（空也算一种），然后计算出条件语句的值，接着按从上到下，从左到右的顺序一个一个的执行 case 语句的条件表达式，如果值相等的话就会进入执行 case 条件下对应的语句。  
如果所有的 case 条件都没有能匹配上的话，然后就会尝试执行 default 下对应的逻辑。


### case 条件合并

如果有时候多个 case 条件对应的处理逻辑是一样的话，Go 语言中的 case 条件是可以合并的。

#### 合并方法
多个条件用逗号分隔，判断顺序是从左到右。
只要有一个符合，就满足条件：

```go
switch var1 {
    case val1,val2,val3:
        statement1
    case val4,val5: 
        statement2
    default:
        statement
}
```


```go
val := 20
switch val {
case 10, 11, 15:
    println(11, 15)
case 16, 20, 22:      // 命中
    println(16, 20, 22)
default:
    println("nothing")
}

```

即使是表达式比较结构，也一样可以使用逗号分隔多个表达式，这时和使用逻辑或"||"是等价的：
```go
func main() {
    val := 21
    switch {
    case val % 4 == 0:
        println(0)
    case val % 4 == 1, val % 4 == 2:  //命中
        println(1, 2)
    default:
        println("3")
    }
}
```


## switch支持的类型

不像 Java 只支持整型进行判断（其他类型都是通过转化成整型实现的），Go 里的 switch 的参数是一个表达式，支持任何类型进行比较，甚至 switch 的条件还可以是一个空的，这个时候等价于 switch true，可以用于简化多个 if 条件的场景。

```go
func price(weight int) int  {
    if weight > 10 {
        return 100
    } else if weight > 8 {
        return 110
    } else if weight > 5 {
        return 120
    } else {
        return 150
    }
}
```

比如上方这个多重 if else 判断逻辑就可以用下方这个无参数的 switch case 语句替代：
```go
func price(weight int) int  {
    switch  {
    case weight>10:
        return 100
    case weight>8:
        return 110
    case weight>5:
        return 120
    default:
        return 150
    }
}
```

## 分类
switch语句用于提供分支测试。有两种swithc结构：expression switch和type switch。
### expression switch

对于expression switch，也有三种形式：等值比较、表达式比较、初始化表达式。

#### 等值比较
当var 的值为val1时，执行statement1，当var1的值为val2时，执行statement2，都不满足时，执行默认的语句statement。

```go
switch var {
    case val1:
        statement1
    case val2:
        statement2
    default:
        statement
}
```

等值比较局限性很大，只能将var 和case中的值比较是否相等。如果想比较不等，或者其它表达式类型，可以使用下面的表达式比较结构。

#### 表达式比较
评估每个case结构中的condition，只要评估为真就执行，然后退出(默认情况下)。

```go
switch {
    case condition1:
        statement1
    case condition2:
        statement2
    default:
        statement
}
```

#### 初始化表达式

可以和if一样为switch加上初始化表达式，同样作用域只在switch可见。但注意，initialization后面记得加上分号";"结尾。见下文示例。

```go
switch initialization; {  // 不要省略分号
    case condition1:
        statement1
    case condition2:
        statement2
    defautl:
        statement
}
```

### type switch
即 switch-case 的 类型断言。


## break

Go 语言中匹配到一个 case 条件执行完对应的逻辑之后就会跳出这个 switch 语句，等价于每个 case 处理逻辑之后都加了一个隐式的 break 语句。

## fallthrough
### 背景

与Java、PHP等语言不同，Go语言的switch case语句不需要在每个case后面添加break语句，默认在执行完case之后，会自动break，从switch语句中转义出来。
即：**默认情况下case命中就结束，所以所有的case中只有一个会被执行**。

### fallthrough 的介绍

如果想要执行多个，可以在执行完的某个case的最后一个语句上加上`fallthrough`，它会无条件地直接跳转到下一条case并执行，如果下一条case中还有fallthrough，则相同的逻辑。

此外，fallthrough的后面必须只能是下一个case或default，不能是额外的任何语句，否则会报错。

```go
func main() {
    val := 21
    switch val % 4 {
    case 0:
        println(0)
    case 1, 2:         // 命中
        println(1, 2)  // 输出
        fallthrough    // 执行下一条，无需条件评估
        // println("sd") //不能加此行语句
    case 3:
        println(3)     // 输出
        fallthrough    // 执行下一条，无需条件评估
    default:
        println("end")  // 输出
    }
}

// 结果
1 2
3
end
```

### 注意
#### fallthrough的透传会直接传递给下一个case，不需要判断下一个case的条件
fallthrough的透传会直接传递给下一个case，而不会去判断下一个case的条件。

#### 多种类型的switch-case类型断言不允许 fallthrough
使用switch判断interface{}的类型的时候，是不被允许使用fallthrough的。

```go
package main

import (
	"fmt"
)

func main() {
	var num interface{}
	var num1 int = 1
	num = num1
	switch num.(type) {
	case int:
		fmt.Printf("当前num类型是:%T\n", num)
    fallthrough
	case float64:
		fmt.Printf("当前num类型是:%T\n", num)
	default:
		fmt.Printf("当前num类型是:%T\n", num)
	}
}

```

![](attachments/Pasted%20image%2020241022160803.png)


#### `fallthrough`用于跳过某个case

```go
swtich i {
    case 0: fallthrough
    case 1: statement1
    default: statement
}
```
它表示等于0或等于1的时候都执行statement1。这和前面的case合并功能是一样的。

### 范例

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    switch num := time.Now().Month(); {
    case num <= 3:
        fmt.Println("当前是第一季度")
    case num > 6:
        fmt.Println("当前是下半年")
    default:
        fmt.Println("未知月份")
    }
}

执行后结果当然会输出：当前是第一季度
而不会继续执行，输出“当前是下半年”或是“未知年份”的提示。
```

可以使用fallthrough关键词来强制执行下面的case语句的内容。
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    switch num := time.Now().Month(); {
    case num <= 3:
        fmt.Println("当前是第一季度")
        fallthrough
    case num > 6:
        fmt.Println("当前是下半年")
    default:
        fmt.Println("未知月份")
    }
}

会输出
当前是第一季度
当前是下半年
```

# 循环结构 for
# 跳转
## break
## continue
## label标签
## goto

# 参考
```bash
# Go基础系列：流程控制结构
https://www.cnblogs.com/f-ck-need-u/p/9866091.html
```