```table-of-contents
```
# 概述
`libevent`,`libev`,`libuv`都是c实现的异步事件库，注册异步事件，检测异步事件，根据事件的触发先后顺序，调用相对应回调函数处理事件。处理的事件包括：`网络 io 事件、定时事件以及信号事件`。这三个事件驱动着服务器的运行。

```bash
1. 网络io事件：
	linux：epoll、poll、select
	mac：kqueue
	window：iocp
2. 定时事件：
	红黑树
	最小堆：二叉树、四叉树
	跳表
	时间轮
3. 信号事件
```

`libevent` 和 `libev` 对 `windows` 支持比较差，由此产生了 `libuv` 库；`libuv` 基于 `libev`，在`window` 平台上更好的封装了 `iocp`；`node.js` 基于 `libuv`；

# libevent

## 定时器事件的实现
libevent 中，用 **最小堆(min-heap)** 来管理所有定时器。 root node 就是最近即将触发的 timer event。node value 就是 timer 触发的时间(秒)。

# 参考
```bash

```