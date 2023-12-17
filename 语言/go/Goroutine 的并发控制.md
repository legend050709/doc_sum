```table-of-contents
```
# 背景
我们知道 Golang 是一门擅长高并发的编程语言，可以通过 Goroutine 快速地创建并发任务，但是如何有效地管理这些执行的 Goroutine 是一个值得思考的问题。
# Goroutine 并发控制的几种方式
通常我们有下面几种方式实现 Goroutine 的控制：

- 使用 **WaitGroup**，根 goroutine 通过 `add()` 来记录已经开启的 Goroutine 数量，通过 `wait()` 来等待执行任务的 goroutine 的 `done()`，实现同步的工作；
- 使用 for/select + stop channel，通过向 stop channel 中传递 stop signal 实现 goroutine 的结束；
- 使用 **Context**， 可以控制具有复杂层级关系的 goroutine 任务，此时使用前两种方式实现会比较复杂，使用 context 会更优雅。

# 范例
## 两个协程交替打印
### 方法一：有缓冲的chan
实现两个协程轮流输出A 1 B 2 C 3 .... Z 26;

```go

func ChannelFunc() {
	// 思想：两个g，一个输出数字，一个输出字母，重点是如何控制两个g的打印顺序，让其可以轮流打印

	// 分别使用两个缓存为1的chan，来控制两个g的打印顺序
	strChan := make(chan int, 1)
	numChan := make(chan int, 1)

	strChan <- 0 // 先往字符chan中塞入，此时strChan再塞入会堵塞

	// 负责打印字母
	go func() {
		for i := 65; i <= 90; i++ {
			<-strChan // strChan取出，因为之前先塞入了，所以此处不会堵塞，会直接打印字符A..Z
			fmt.Printf("%v ", string(rune(i))) // 打印字母
			numChan <- i // numChan 塞入，塞入后，另一个g的numChan取出操作才能进行
		}
		return
	}()

  // 负责打印数字
	go func() {
		for i := 1; i <= 26; i++ {
			<-numChan // 一直阻塞，直到字母被打印，这样每次数字都是在字母后面被打印的
			fmt.Printf("%v ", i) /// 打印数字
			strChan <- i // strChan塞入，此处塞入后，上面协程的strChan取出操作才能进行，才会打印字母，这样保证了打印完数字后，紧接着打印字母
		}
		return
	}()

	time.Sleep(1 * time.Second)
	fmt.Println()
	
	// A 1 B 2 C 3 D 4 E 5 F 6 G 7 H 8 I 9 J 10 K 11 L 12 M 13 N 14 O 15 P 16 Q 17 R 18 S 19 T 20 U 21 V 22 W 23 X 24 Y 25 Z 26 
}

```
### 方法二：无缓冲的chan
实现两个协程轮流输出A 1 B 2 C 3 .... Z 26;
```go
func ChannelFunc() {
	strChan := make(chan int)
	numChan:= make(chan int)

	// 打印字母
	go func() {
		for i := 65; i <= 90; i++ {
			// 保证字母先被打印
			fmt.Printf("%v ", string(rune(i)))
			strChan <- i // strChan塞入，通知数字可以被打印了
			<-numChan // numChan一直被阻塞，直到被g2通知可以打印字母了
		}
		return
	}()

	// 打印数字
	go func() {
		for i := 1; i <= 26; i++ {
			<-strChan // strChan拉取，一直被阻塞，直到被strChan被塞入，即被g1通知数字可以打印了之后才解除阻塞
			fmt.Printf("%v ", i)
			numChan <- i // numChan塞入，通知g1可以打印字母
		}
		return
	}()

	time.Sleep(1 * time.Second)
	fmt.Println()
}

```
### 方法三：设置GOMAXPROCS=1
Go用两个协程交替打印100以内的奇偶数；如下所示：

```go
package main

import (
	"fmt"
	"runtime"
	"time"
)

func main() {
	//设置可同时使用的CPU核数为1
	runtime.GOMAXPROCS(1)
	go func() {
		for i := 1; i < 101; i++ {
			//奇数
			if i%2 == 1 {
				fmt.Println("协程1打印:", i)
			}
			//让出cpu
			runtime.Gosched()
		}
	}()
	go func() {
		for i := 1; i < 101; i++ {
			//偶数
			if i%2 == 0 {
				fmt.Println("协程2打印:", i)
			}
			//让出cpu
			runtime.Gosched()
		}
	}()
	time.Sleep(3 * time.Second)
}

```
## 多个协程交替打印

### 范例一：N个协程交替打印1-100；
```go

func main() {
	gorutinenum := 5
	var chanslice []chan int
	exitchan := make(chan int)

	for i := 0; i < gorutinenum; i++ {
		chanslice = append(chanslice, make(chan int, 1))
	}

	res := 1
	j := 0
	for i := 0; i < gorutinenum; i ++ {
		go func(i int) {
			for {
				<-chanslice[i]
				if res > 100 {
					exitchan <- 1
					break
				}

				fmt.Println(fmt.Sprintf("gorutine%v:%v", i, res))
				res ++

				if j == gorutinenum-1 {
					j = 0
				}else {
					j ++
				}
				chanslice[j] <- 1
			}
		}(i)
	}
	chanslice[0] <- 1

	select {
	case <-exitchan:
		fmt.Println("end")
	}
}

```

### 范例二：使用3个goroutine交替打印ABC
```go
package main

import (
	"fmt"
	"sync"
)
func main() {
	var ch1, ch2, ch3 = make(chan struct{}), make(chan struct{}), make(chan struct{})
	var wg sync.WaitGroup
	wg.Add(3)
	go func(s string) {
		defer wg.Done()
		for i := 1; i <= 10; i++ {
			<- ch1
			fmt.Print(s)
			ch2 <- struct{}{}
		}
		<- ch1
	}("A")
	go func(s string) {
		defer wg.Done()
		for i := 1; i <= 10; i++ {
			<- ch2
			fmt.Print(s)
			ch3 <- struct{}{}
		}
	}("B")
	go func(s string) {
		defer wg.Done()
		for i := 1; i <= 10; i++ {
			<- ch3
			fmt.Println(s)
			ch1 <- struct{}{}
		}
	}("C")
	ch1 <- struct{}{}
	wg.Wait()
}

```

### 范例三：使用N个协程交替打印英文字母
```go
package main

import "fmt"

func main() {
	chanNum := 4
	chanQueue := make([]chan struct{}, chanNum)
	var result = 0
	exitChan := make(chan struct{})
	for i := 0; i < chanNum; i++ {
		chanQueue[i] = make(chan struct{})
		if i == chanNum - 1 {
			go func(i int) {
				chanQueue[i] <- struct{}{}
			}(i)
		}
	}
	for i := 0; i < chanNum; i++ {
		var lastChan, curChan chan struct{}
		if i == 0 {
			lastChan = chanQueue[chanNum - 1]
		} else {
			lastChan = chanQueue[i - 1]
		}
		curChan = chanQueue[i]
		go func(i byte, lastChan, curChan chan struct{}) {
			for {
				if result > 20 {
					exitChan <- struct{}{}
				}
				<- lastChan
				fmt.Printf("%c\n", i)
				result++
				curChan <- struct{}{}
			}
		}('A'+byte(i), lastChan, curChan)
	}
	<- exitChan
	fmt.Println("done")
}

```
- 第一个for循环中为chanQueue数组中的每个channel实例化。同时给最后一个channel写一条数组，为了第一次输出能从第一个channel中输出，如果不写会造成死锁，而且需要创建一个协程来写，否则程序直接报错。
- 第二个for循环中lastChan意思是，如果当前chan要打印数据，必须是上一个chan打印完后才能打印。
- 在这里使用匿名函数的时候最好使用这种带参数的传参，而不是直接使用全局变量，否则因为协程并发执行的不确定性可能造成死锁。


# 参考
```c

```