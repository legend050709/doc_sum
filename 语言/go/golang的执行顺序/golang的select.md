```table-of-contents
```
# 介绍
Golang 中的 select 语句是用于多路复用的一种语言结构，用于同时等待多个通道上的数据，并执行相应的代码块。

也就是说 select 是用来监听和 channel 有关的 IO 操作，它与 select，poll，epoll 相似，当 IO 操作发生时，触发相应的动作，实现 IO 多路复用。
# 特性
## `select..case`
`select..case`则只能处理 channel类型。**即每个 case 必须是一个通信操作, 要么是发送要么是接收**。

select 将随机执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行。
- 每个 case 都必须是一个通信；
- 所有 channel 表达式都会被求值；
- 如果有多个 case 都可以执行，Select 会随机公平地选出一个执行。其他不会执行。  
    否则：  
    如果有 default 子句，则执行该语句。  
    如果没有 default 子句，select 将阻塞，直到某个通信可以运行；Go 不会重新对 channel 或值进行求值。
# Select的基本用法

## 随机选择
`Random Select`
```go
package main

import "fmt"

func main() {

	ch1 := make(chan int,1)
	ch2 := make(chan int,1)
	ch3 := make(chan int,1)

	ch6 := make(chan int,1)


	var i1, i2 int

	select {
	case i1 = <-ch1:
		fmt.Println("接收到了管道1的一条数据:", i1)
	case ch2 <- i2:
		fmt.Println("向管道2发送了一条数据:", i2)

	case i3, ok := <-ch3:
		if ok {
			fmt.Println("收到了管道3的数据:", i3)
		} else {
			fmt.Println("管道3已被关闭")
		}

	case ch6 <- 271828:
		fmt.Println("向管道6发送了一条数据:", 271828)

	default:
		fmt.Println("以上case皆不可运行,即没有进行通信")

	}

}

```

第二个case和第四个case是可以执行的,所以不会走default语句.输出为:
```go
向管道2发送了一条数据: 0
```
或
```go
向管道6发送了一条数据: 271828
```

二者都满足条件,被select选中执行的概率完全一样。

## 超时控制
利用 `select+time.After`。
```go
package main

import (
	"fmt"
	"time"
)

func main() {

	ch := make(chan string)

	go func() {
		time.Sleep(time.Second * 2)
		ch <- "写入某个值"

	}()

	select {
	case rs := <-ch:
		fmt.Println("结果是:", rs)

	case <-time.After(time.Second * 1):
		fmt.Println("超时!")
	}
}
```

## for-select
如果有多个 channel 需要读取, 且读取是不间断的, 就必须使用 `for + select` 机制来实现。

# for-select
golang里的select，用来监听channel的读写事件，当事件发生时，触发相应的动作。
```go
//select基本用法
select {
case <- chan1:
// 如果chan1成功读到数据，则进行该case处理语句
case chan2 <- 1:
// 如果成功向chan2写入数据，则进行该case处理语句
default:
// 如果上面都没有成功，则进入default处理流程
}
```
select执行后会直接结束。所以一般会跟for循环结合使用。

```go
func testSelect() {
  ticker := time.NewTicker(time.Second)
  defer ticker.Stop()
  for {
    select {
    case <-ticker.C:
      fmt.Println(1)
    }
  }
}
​
// 会一直循环下去
// 运行输出结果：
1
1
...
```
## for-select中的break
在select中break，并不会跳出for循环。
```go
func testSelectBreak() {
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()
   for {
      select {
      case <-ticker.C:
         fmt.Println(1)
         break // 无法跳出for循环
      }
   }
}
​
// 会一直循环下去
// 运行输出结果：
1
1
...
```

break结合标签可以跳到for循环之外：
```go
func testSelectBreakLabel() {
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()
​
Label:
   for {
      select {
      case <-ticker.C:
         fmt.Println(1)
         break Label // 跳出for循环
      }
   }
​
   fmt.Println("end")
}
​
// 运行输出结果：
1
end
```

## for-select中的goto
在for-select，使用goto结合标签可以跳到for循环之外。
> 与break的区别在于break中的Label标签一定要在for语句之前定义，goto中的标签没有这个要求。


```go
func testSelectGotoLabel() {
   ticker := time.NewTicker(time.Second)
   defer ticker.Stop()
​
   for {
      select {
      case <-ticker.C:
         fmt.Println(1)
         goto Label // 跳出for循环
      }
   }
​
Label:
   fmt.Println("end")
}
​
// 运行输出结果：
1
end
```
## for-select中的 continue
continue无法在select里单独使用，编译报错，使用的话必须在for里面，使用for-select模式。

```go
func testSelectContinue() {
  ticker := time.NewTicker(time.Second)
  defer ticker.Stop()
​
Label:
  for {
    select {
    case <-ticker.C:
      fmt.Println(1)
      continue Label // 并不会跳出for循环；而是跳过此轮循环，进入下一轮
      fmt.Println(2) // continue之后的语句不会执行
    }
  }
}
​
// 会一直循环下去
// 运行输出结果：
1
1
...
```
# Select 和 Switch对比
## 相同点
二者有个共同特性就是都通过`case`的方式来处理, 但除此之外几乎完全不同;

## 不同点
### `switch..case`
`switch..case` 可以处理各种类型，常用来做 接口 interface{} 的判断 (通过variable.(type)). 重点是**`switch..case`会依照 case 的顺序依序执行**。

而`select..case`则只能处理 channel类型；**即每个 case 必须是一个通信操作, 要么是发送要么是接收**。并且，select 将随机执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行。


## 范例
```go
package main
import "fmt"

func convert(val interface{}) {
	switch t := val.(type) {
	case int:
		fmt.Println("val为int类型", t)
	case string:
		fmt.Println("val为string类型", t)
	case float64:
		fmt.Println("val为float64类型", t)
	case float32:
		fmt.Println("val为float32类型", t)
	case []string:
		fmt.Println("val为字符串类型的切片", t)
	default:
		fmt.Println("val不是上列类型之一")
	}
}

func main() {
	var i interface{}
	i = float32(3.1415926)
	convert(i)
	i = "dashen"
	convert(i)
	i = 100
	convert(i)
	i = []string{"欧拉", "高斯"}
	convert(i)
}
```

输出为:
```go
val为float32类型 3.1415925
val为string类型 dashen
val为int类型 100
val为字符串类型的切片 [欧拉 高斯]
```


# 关键字的语法
## break
Go 语言中的 break 语句可以用于，立即跳出 **switch、for 和 select**；
> break在for中单独使用只能跳出当前循环，无法跳出多层循环。
但不同的是 Go 语言中的 break 语句可以指定标签。


范例：
```go
package main

import (
	"fmt"
)

func main() {
	for i := 0; i < 3; i++ {
		for j := 0; j < 10; j++ {
			if j == 2 {
				break
			}
			fmt.Println("oter j =", j)
		}
	}
	fmt.Println("come here oter!")
	fmt.Println(" ")
}


[Running] go run "/Users/wangyang/go/src/test/main.go"
oter j = 0
oter j = 1
oter j = 0
oter j = 1
oter j = 0
oter j = 1
come here oter!
```
### Break 与 标签
Break 与 标签跳转
- 标签必须在使用之前定义
- 标签后面只能跟 switch 和循环语句，不能插入其它语句
- 跳转到标签之后 switch 和循环不会再次被执行

范例：
```go
package main

import (
	"fmt"
)

func main() {
oter:
	for i := 0; i < 3; i++ {
		for j := 0; j < 10; j++ {
			if j == 2 {
				break oter
			}
			fmt.Println("oter j =", j)
		}
	}
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
oter j = 0
oter j = 1
```

```go
package main

import (
	"fmt"
)

func main() {
oter:
	switch num := 2; num {
	case 1:
		fmt.Println("1")
	case 2:
		fmt.Println("2")
		break oter
	default:
		fmt.Println("other")
	}
	fmt.Println("come here")
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
2
come here
```
### break 与 select
类似于switch中的break，break后的语句不会被执行到
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tick := time.Tick(time.Second)

	select {
	case t := <-tick:
		fmt.Println("t")
		break
		fmt.Println(t)
	}

	fmt.Println("main exit")
}
```

输出:

```sh
t
main exit
```

在for-select中，break只会影响到select，不会影响到for。
```go
package main

import (
	"fmt"
	"time"
)

func main() {
	tick := time.Tick(time.Second)

	for {
		select {
		case t := <-tick:
			fmt.Println("t")
			break
			fmt.Println(t)
		}

		fmt.Println("for")
	}

	fmt.Println("main exit")
}
```

输出:

```sh
t
for
t
for
t
for
#...
```

## continue
单独在select中是不能使用continue，会编译错误，只能用在for中。
Go 语言中的 continue 语句可以用于立即进入下一次循环，不会跳出当前的循环;
但不同的是 Go 语言中的 continue 语句可以指定标签;

```go
package main

import (
	"fmt"
)

func main() {
	for i := 0; i < 3; i++ {
		fmt.Println("i =========================", i)
		for j := 0; j < 4; j++ {
			if j == 2 {
				continue
			}
			fmt.Println("j=", j)
		}
	}
	fmt.Println("come here")
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
i ========================= 0
j= 0
j= 1
j= 3
i ========================= 1
j= 0
j= 1
j= 3
i ========================= 2
j= 0
j= 1
j= 3
come here
```

### Continue 与 标签
Continue 与 标签跳转
- 标签必须在使用之前定义
- 标签后面只能跟循环语句，不能插入其它语句
- 对于单层循环和直接编写 continue 一样
- 对于多层循环，相当于跳出最外层循环继续判断条件执行

```go
package main

import (
	"fmt"
)

func main() {
oter:
	for i := 0; i < 3; i++ {
		fmt.Println("i=================", i)
		for j := 0; j < 4; j++ {
			if j == 2 {
				continue oter
			}
			fmt.Println("j=", j)
		}
	}
	fmt.Println("come here")
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
i================= 0
j= 0
j= 1
i================= 1
j= 0
j= 1
i================= 2
j= 0
j= 1
come here
```

## goto
Go 语言中的 goto 用于在同一个函数中跳转。

### 范例

```go
package main

import "fmt"

func main() {
	num := 1
outer:
	if num <= 10 {
		fmt.Println(num)
		num++
		goto outer
	}

	fmt.Println("come here")
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
1
2
3
4
5
6
7
8
9
10
come here
```


```go
package main

import "fmt"

func main() {
	num := 1

	if num <= 10 {
		fmt.Println(num)
		num++
		goto outer
	}

outer:
	fmt.Println("come here")
}

[Running] go run "/Users/wangyang/go/src/test/main.go"
1
come here

```

## return
Golang 语言中 return 语句和 C 语言一模一样，都是用于结束函数，将结果返回给调用者。

# 参考
```c
golang基础(1) - select中的break、continue和return
https://tomjamescn.github.io/post/2019-07-11-golang-basic-1-break-continue-return-in-select/

https://zhuanlan.zhihu.com/p/595911952
https://github.com/binze51/binze51/issues/12

Go语言中常见问题
https://cloud.tencent.com/developer/article/2313298
```