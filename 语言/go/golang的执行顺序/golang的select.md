```table-of-contents
```
# 介绍
Golang 中的 select 语句是用于**IO多路复用**的一种语言结构，用于同时等待多个通道上的数据，并执行相应的代码块。

也就是说 **select 是用来监听和 channel 有关的 IO 操作，它与 select，poll，epoll 相似，当 IO 操作发生时，触发相应的动作，实现 IO 多路复用**。

# 特性
## `select..case`
`select..case`则**只能处理 channel类型。即每个 case 必须是一个通信操作, 要么是发送要么是接收**。

## 特点
select 将**随机**执行一个可运行的 case。如果没有 case 可运行，它将阻塞，直到有 case 可运行。
- 每个 case 都必须是一个通信；
- 所有 channel 表达式都会被求值；
- 如果有多个 case 都可以执行，Select 会随机公平地选出一个执行。其他不会执行。  
- 如果没有case 可以执行
    如果有 default 子句，则执行该语句。  
    如果没有 default 子句，select 将阻塞，直到某个通信可以运行；Go 不会重新对 channel 或值进行求值。

**注意：nil channel上的操作会一直被阻塞，如果没有default case,只有nil channel的select会一直被阻塞。**

### 为什么随机选择
为什么是随机执行的呢？
随机的引入就是为了避免饥饿问题的发生，如果我们每次都是按照顺序依次执行的，若两个`case`一直都是满足条件的，那么后面的`case`永远都不会执行。


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

# select的超时控制
## 背景
select...case， 如果没有case 可以执行 ，并且没有 default 子句，select 将阻塞，直到某个通信可以运行。
为了防止长时间阻塞，可以设置定时器超时作为其中的一个case。

## time.After() 超时设置

谁也无法保证某些情况下的select是否会永久阻塞。很多时候都需要设置一下select的超时时间，可以借助time包的After()实现。


### 使用

time.After()的定义如下：
```go
func After(d Duration) <-chan Time
```

After()函数接受一个时长d，然后它After()等待d时长，等待时间到后，将等待完成时所处时间点写入到channel中并返回这个只读channel。

所以，将该函数赋值给一个变量时，这个变量是一个只读channel，而channel是一个引用类型的数据，所以它是一个指针。

```go
package main

import (
	"fmt"
	"time"
)
func main() {
	fmt.Println(time.Now())
	a := time.After(1*time.Second)
	fmt.Println(<-a)
	fmt.Println(a)
}

结果：
2018-11-20 19:05:11.5440307 +0800 CST m=+0.001994801 
2018-11-20 19:05:12.5496378 +0800 CST m=+1.007601901 
0xc042052060
```

### 范例
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

## time.Tick() 超时设置

因为Tick()也是在等待结束的时候发送数据到通道，所以它的返回值是一个channel，从这个channel中可读取每次等待完时的时间点。

### After和 Tick的区别
After(d)是只等待一次d的时长，并在这次等待结束后将当前时间发送到通道。
Tick(d)则是间隔地多次等待，每次等待d时长，并在每次间隔结束的时候将当前时间发送到通道。

### 范例

#### Tick 在select-case内
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    select {
    case <-time.Tick(2 * time.Second):
        fmt.Println("2 second over:", time.Now().Second())
    case <-time.After(7 * time.Second):
        fmt.Println("5 second over, timeover", time.Now().Second())
        return
    }
}
```

上面的示例，在等待2秒之后，就会因为读取到了time.Tick()的通道数据而终止，因为select并未在for循环内。


#### Tick 在for-select 的 case内
如果是for-select，第二个case将永远选择不到。因为每次select轮询中，第一个case都因为2秒而先被选中，使得**第二个case的评估总是被中断**。进入下一个select轮询后，又会重新开始评估两个case，分别等待2秒和7秒。

```go
func main() {
    for {
        select {
        case <-time.Tick(2 * time.Second):
            fmt.Println("2 second over:", time.Now().Second())
        case <-time.After(7 * time.Second):
            fmt.Println("5 second over, timeover", time.Now().Second())
            return
        }
    }
}
```

上面不正常执行的原因是因为每次select都会重新评估这些表达式。

#### Tick 在 for-select 外
如果把这些表达式放在select外面，则正常：

```go
package main

import (
    "fmt"
    "time"
)

func main() {
    tick := time.Tick(1 * time.Second)
    after := time.After(7 * time.Second)
    fmt.Println("start second:",time.Now().Second())
    // 打印当前时间的秒值
    for {
        select {
        case <-tick:
            fmt.Println("1 second over:", time.Now().Second())
        case <-after:
            fmt.Println("7 second over:", time.Now().Second())
            return // 此中存在return
        }
    }
}

结果：
start second: 9
1 second over: 10
1 second over: 11
1 second over: 12
1 second over: 13
1 second over: 14
1 second over: 15
1 second over: 16
7 second over: 16

```
将time.Tick()和time.After()放在for...select的外面，使得select每次只评估通道是否可读、可写事件，而不会重新执行time.Tick()和time.After()，这样就避免使得它们重新进入计时状态。

注意上面的输出结果中，有两行：
```go
1 second over: 16 
7 second over: 16
```
说明在第16秒的时候，两个case都评估为真了，但是这一次选择了第一个case，然后进入下一个select过程，因为select的随机选择性，它会保证所有满足条件的case尽量均衡分布，这次将选择第二个case，它仍然为第16秒，这时因为一次for和select调用所花的时间不可能会超过1秒而进入第17秒。


如下所示，删除 after 后的 return，可以看到 tick 可以多次超时。
```go
package main

import (
    "fmt"
    "time"
)

func main() {
    tick := time.Tick(1 * time.Second)
    after := time.After(7 * time.Second)
    fmt.Println("start second:",time.Now().Second())
    for {
        select {
        case <-tick:
            fmt.Println("1 second over:", time.Now().Second())
        case <-after:
            fmt.Println("7 second over:", time.Now().Second())
        }
    }
}
```

```text
结果如下所示：

start second: 56
1 second over: 57
1 second over: 58
1 second over: 59
1 second over: 0
1 second over: 1
1 second over: 2
1 second over: 3
7 second over: 3
1 second over: 4
1 second over: 5
1 second over: 6
1 second over: 7
1 second over: 8
1 second over: 9
1 second over: 10
1 second over: 11
1 second over: 12
1 second over: 13
1 second over: 14
1 second over: 15
1 second over: 16
1 second over: 17
1 second over: 18
1 second over: 19
1 second over: 20
1 second over: 21
1 second over: 22
1 second over: 23
1 second over: 24
1 second over: 25
1 second over: 26
1 second over: 27
1 second over: 28
1 second over: 29
1 second over: 30
1 second over: 31
1 second over: 32
1 second over: 33
1 second over: 34
1 second over: 35
1 second over: 36
1 second over: 37
1 second over: 38
1 second over: 39
1 second over: 40
1 second over: 41
1 second over: 42
1 second over: 43
1 second over: 44
1 second over: 45
1 second over: 46
1 second over: 47
1 second over: 48
1 second over: 49
1 second over: 50
1 second over: 51
1 second over: 52
1 second over: 53
1 second over: 54
1 second over: 55
1 second over: 56
1 second over: 57
1 second over: 58
1 second over: 59
1 second over: 0
1 second over: 1
1 second over: 2
1 second over: 3
...
...

```

## 注意

Tick() 和 After()放在select的内部和放在select的外部是完全不一样的。

# 关键字的语法
## break
Go 语言中的 break 语句可以用于，立即跳出 **switch、for 和 for-select**；
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

### break 与 select
类似于switch中的break，在select中执行break，break后的语句不会被执行到
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

#### 单独的select中使用break没什么用

```go
func test1() {
    select {
    case <-time.After(time.Second):
        fmt.Println("一秒后执行")
        //break
    }
}


func test2() {
    select {
    case <-time.After(time.Second):
        fmt.Println("一秒后执行")
        break //跳出当前select
    }
}
func main() {
    test1()
    fmt.Println("========================")
    test2()
}
/**
output:
一秒后执行
========================
一秒后执行
 */

```
以上代码中test1没有使用break test2使用了break。实际执行结果完全一样。可以 得出:
单独在select中使用break和不使用break没有啥区别。


**那为啥要在select中支持break呢？**
实际上在select中引入break主要的目的是为了break和label组合使用，来实现跳出多层作用域, 比如 for-select 作用域。

```go
func main() {

SELECT:
    for {
        select {
        case <-time.After(time.Second):
            fmt.Println("一秒后退出")
            //break 跳出select
            break SELECT  //带标签的break，实际上跳出到select外层的for语句块
        case <-time.After(time.Second * 10):
            fmt.Println("十秒后退出")
            break
        }
    }
    
    fmt.Println("select 语句结束")
}
/**
output:
一秒后退出
select 语句结束
 */

```

### Break 与 标签
#### 使用方法
Break 与 标签跳转的使用方法
- 标签必须在使用之前定义
- **标签后面只能跟 switch 和循环语句**，不能插入其它语句
- **跳转到标签之后 switch 和循环不会再次被执行**

#### 范例

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


## continue
单独在select中是不能使用continue，会编译错误，只能用在for/for-select中。
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
#### 使用方法
Continue 与 标签跳转
- 标签必须在使用之前定义
- **标签后面只能跟循环语句**，不能插入其它语句
- 对于单层循环和直接编写 continue 一样
- **对于多层循环，相当于跳出最外层循环继续判断条件执行**

#### 范例
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

# for-select
## 背景
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
select执行后会直接结束。

**如果有多个 可读可写的IO事件 需要读取, 且读取是不间断的, 就必须使用 `for + select` 机制来实现**。

```go

for {
	....
} // 表示永久循环；

select {
	case xxx:
		....
	case xxx:
		....
} // 表示 IO多路复用
```


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
### for-select 中使用break无法跳出 for
在for-select中break，并不会跳出for循环。break只能跳出select，无法跳出for。

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

### for-select 使用 break+标签跳出for

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

## for-select中的goto跳出for
在for-select，使用goto结合标签可以跳到for循环之外。


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

### goto跳转和 break + 标签跳转for的区别
goto跳转for循环和 break + 标签调整for的区别：
在于break+标签，Label标签一定要在for语句之前定义，goto中的标签没有这个要求。

## for-select中的 continue
continue无法在select里单独使用，编译报错，使用的话必须在for-select 或者 for 里面。
**continue 的语义就类似for中的语义，continue 后的代码不会被执行到。**

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
### `switch..case`  和 `select..case`区别

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



# 参考
```c
# Go基础系列：流程控制结构
https://www.cnblogs.com/f-ck-need-u/p/9866091.html

golang基础(1) - select中的break、continue和return
https://tomjamescn.github.io/post/2019-07-11-golang-basic-1-break-continue-return-in-select/

https://zhuanlan.zhihu.com/p/595911952
https://github.com/binze51/binze51/issues/12

Go语言中常见问题
https://cloud.tencent.com/developer/article/2313298
```