```table-of-contents
```
# 进程、线程、协程区分

Python中多任务的实现可以使用进程和线程，也可以使用协程。

## 基础知识
当我们挂起一个执行流的时，我们要保存的东西：

**栈**：其实在你切换前你的局部变量，以及要函数的调用都需要保存，否则都无法恢复；
**寄存器状态**：这个其实用于当你的执行流恢复后要做什么。

而寄存器和栈的结合就可以理解为**上下文**。

**上下文切换**的理解：
CPU看上去像是在并发的执行多个进程，这是通过处理器在进程之间切换来实现的，操作系统实现这种交错执行的机制称为上下文切换。

制权从当前进程转移到某个新进程时，就会进行上下文切换，即保存当前进程的上下文，恢复新进程的上下文，然后将控制权传递到新进程，新进程就会从它上次停止的地方开始。

## 协程介绍

我们通常所说的协程Coroutine其实是corporate routine的缩写，直接翻译为协同的例程，一般我们都简称为协程。
在linux系统中，线程就是轻量级的进程，而我们通常也把协程称为轻量级的线程即微线程。

协程是python中另外一种实现多任务的方式，比线程更小、占用更小执行单元（理解为需要的资源）。

在一个线程中的某个函数，可以在任何地方保存当前函数的一些临时变量等信息，然后切换到另外一个函数中执行。

注意不是通过调用函数的方式做到的，并且切换的次数以及什么时候再切换到原来的函数都由开发者自己决定。


## 为什么需要协程
在实现多任务时, 线程切换从系统层面来说，远不止保存和恢复CPU上下文那么简单。 
操作系统为了程序运行的高效性，每个线程都有自己缓存的数据，操作系统还会帮我们做这些数据的恢复操作。
所以线程的切换非常耗性能。协程的切换只是单纯的操作CPU的上下文，所以一秒钟切换上百万次系统都抗的住。

## 线程和协程
协程也被称为微线程，下面对比一下协程和线程：

线程之间需要上下文切换成本相对协程来说是比较高的，尤其在开启线程较多时，但协程的切换成本非常低。

同样的线程的切换更多的是靠操作系统来控制，而协程的执行由我们自己控制。

协程只是在单一的线程里不同的协程之间切换，其实和线程很像，线程是在一个进程下，不同的线程之间做切换，这也可能是协程称为微线程的原因吧。



# gevent
## 介绍

Gevent模块是一种基于协程的Python网络库，它用到Greenlet提供的，封装了在libev和libuv（event loop）事件循环之上的高层同步API。它让开发者在不改变编程习惯的同时，**用同步的方式写异步I/O的代码**。

## 特点

```bash
基于libev的快速事件循环(Linux上epoll，FreeBSD上kqueue）。
基于greenlet的轻量级执行单元。
API的概念和Python标准库一致(如事件，队列)。
可以配合socket，ssl模块使用。
能够使用标准库和第三方模块创建标准的阻塞套接字(gevent.monkey)。
默认通过线程池进行DNS查询,也可通过c-are(通过GEVENT_RESOLVER=ares环境变量开启）。
TCP/UDP/HTTP服务器
子进程支持（通过gevent.subprocess）
线程池
```

## 安装
```bash
pip install gevent
```

## 常用方法

gevent常用方法：


|   |   |
|---|---|
|gevent.spawn()|创建一个普通的Greenlet对象并切换|
|gevent.spawn_later(seconds=3)|延时创建一个普通的Greenlet对象并切换|
|gevent.spawn_raw()|创建的协程对象属于一个组|
|gevent.getcurrent()|返回当前正在执行的greenlet|
|gevent.joinall(jobs)|将协程任务添加到事件循环，接收一个任务列表|
|gevent.wait()|可以替代join函数等待循环结束，也可以传入协程对象列表|
|gevent.kill()|杀死一个协程|
|gevent.killall()|杀死一个协程列表里的所有协程|
|monkey.patch_all()|非常重要，会自动将python的一些标准模块替换成gevent框架|


## 范例
### 范例一：
```python
from gevent import monkey  # 为了能识别time模块的io
 
monkey.patch_all()  # 必须放到被打补丁者的前面，如 time，socket 模块之前
import gevent
import time
 
 
def gf(name):
    print(f'{name}:我想打王者！！')
    # gevent.sleep(2)
    time.sleep(2)
    print(f'{name}:我想吃大餐！！！')
 
 
def bf(name):
    print(f'{name}:一起打！！！')
    # gevent.sleep(2)
    time.sleep(2)
    print(f'{name}:一快去吃！！')
 
 
if __name__ == "__main__":
    start = time.time()
    # 创建协程对象
    g1 = gevent.spawn(gf, '张三')
    g2 = gevent.spawn(bf, '李四')
 
    # 开启任务
    g1.join()
    g2.join()
    end = time.time()
    print(end - start)
```

运行结果：

![](attachments/Pasted%20image%2020240508152309.png)


说明：
`gevent.spawn()`方法会创建一个新的greenlet协程对象，并运行它。`gevent.joinall()`方法会等待所有传入的greenlet协程运行结束后再退出，这个方法可以接受一个`timeout`参数来设置超时时间，单位是秒。



### 范例二
```python
import time

import gevent
from gevent import monkey; monkey.patch_all()

def test(str):
    time.sleep(1)
    return str

if __name__ == '__main__':
    g1 = gevent.spawn(test, "a")
    g2 = gevent.spawn(test, "b")
    g3 = gevent.spawn(test, "c")
    gevent.joinall([g1, g2, g3])
    print(g1.value)
    print(g2.value)
    print(g3.value)

```

说明：
```text
模拟了一个耗时的异步任务，可以看到，a,b,c是同时打印的，也就是说不会出现等待， 可以使用这种方式执行并行的任务。
```

### 范例三：同步和异步
```python3
# cat aa.py
import gevent
import random
def task(pid):
    gevent.sleep(random.randint(0,2)*0.001)
    print('Task %s done' % pid)
def synchronous():
    for i in range(5):
        task(i)
def asynchronous():
    threads = [gevent.spawn(task, i) for i in range(5)]
    gevent.joinall(threads)

if __name__ == "__main__":
    print('Synchronous:')
    synchronous()
    print('Asynchronous:')
    asynchronous()
```

结果如下所示：
```bash
# python3 aa.py
Synchronous:
Task 0 done
Task 1 done
Task 2 done
Task 3 done
Task 4 done
Asynchronous:
Task 0 done
Task 1 done
Task 4 done
Task 2 done
Task 3 done
```

# 其他
## gevent是如何确保多个协程之间不共享相同的线程本地变量的

gevent可以使用`Local`类来创建协程专用的线程本地变量。该类提供了`value`属性来存储线程本地变量的值，并使用`link_local`函数将变量绑定到当前协程。这样，每个协程将具有自己的线程本地变量，并且不会与其他协程共享。

```python
import gevent
from gevent.local import local


# 创建线程本地变量
my_local = local()


def my_coroutine(name):
    my_local.value = name  # 在当前协程中设置线程本地变量的值
    print('Hello,', my_local.value)  # 打印线程本地变量的值


# 创建并启动两个协程
gevent.spawn(my_coroutine, 'Alice')
gevent.spawn(my_coroutine, 'Bob')

# 等待所有协程完成
gevent.wait()
```

运行此示例代码将输出以下内容：

```
Hello, Alice
Hello, Bob
```

# 参考
```bash

```