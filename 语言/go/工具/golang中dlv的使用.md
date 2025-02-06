```table-of-contents
```
# 介绍

# 使用
## 输出到文件
**背景**
某个dlv命令输出太多，标准输出放不下；

**使用**
```bash
(dlv) transcript -x /home/xxx/debug/sources
(dlv) goroutines -t 20 
(dlv) transcript -off
```
其中-x是标准输出就不打印了，直接输出到文件中；
收集完数据后，记得 `transcript -off`关掉

## 输出字符串限制
通过`config -list` 可以查看可以配置的选项，其中 `max-string-len`可以配置输出的最大字符串长度.

## 打断点
进入调试
```bash
dlv attach PID
```

```go
$ dlv attach 22300
Type 'help' for list of commands.
(dlv) b main.go:12
Breakpoint 1 set at 0x6f34ee for main.main.func1() ./main.go:12
(dlv) c # 执行到这里之后，会卡住
```

# 应用
## gcore生成core文件，通过dlv分析

```bash
gcore `pidof go_keepalived`
./dlv core /opt/kgw/bin/go_keepalived core.`pidof go_keepalived`
>transcript -x /tmp/bb
>goroutines -t 20
>transcript -off
>quit
```

## golang程序异常时生成core文件
单纯的启动`golang`程序，即使设置了 `ulimit -c`，发生异常时，也不会生成`core`文件。如下所示：
```bash
./go_add_test 1 2 
1 和 2 是参数。
```
需要设置额外的环境变量，才会生成`coredump`文件，如下所示。
```bash
GOTRACEBACK=crash ./go_add_test 1 2 
```

# 参考
```bash

```