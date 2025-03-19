```table-of-contents
```

# 背景

当调试一个程序的时候，理想状态是**不重启应用程序就获取core文件**。

# 介绍
使用 **gcore (generate-core-file)** 命令可以在**不影响原程序继续运行的情况下获取正在运行的程序的内存转储文件（coredump文件）**，以便后续的调试和分析，后续可以直接使用gdb进行调试。


注：使用 `gcore` 时，生成的核心转储`coredump`文件是该进程在**调用 `gcore` 时刻**的内存状态快照。

## 具体说明
###  内存快照  

当你运行 `gcore <pid>` 时，`gcore` 会在指定的进程（由 `<pid>` 指定）上创建一个内存映像。这个映像反映了该进程在调用 `gcore` 时的内存状态，包括堆、栈、数据段和代码段等内容。

### 生成的核心转储

生成的核心转储文件是一个**特定时刻的快照**，而不是一段时间的内存镜像。尽管在生成过程中进程可能继续运行，但最终的核心转储文件只包含 `gcore` 被调用时的内存状态。



# 工作原理

`gcore`命令会调用`/proc/PID/maps`文件，它是内核用来跟踪进程内存映射的文件。`gcore`命令根据这个文件来定位进程的内存块，并将它们写入转储文件中。

默认情况下，`gcore`命令会根据进程ID生成转储文件的名称，格式为`core.PID`。可以通过使用`-o` 选项指定其他的输出文件名。

# 影响

生成内核转储文件可能需要一定的时间和资源（比如：IO资源，CPU资源），因为它需要将进程的整个内存空间写到磁盘上。
比如：**执行完 gcore命令，发现并没有马上生成 coredump文件，而是一段时间之后，才生成 coredump文件**。



# 使用


![](attachments/Pasted%20image%2020240527164258.png)

```bash
gcore [options] pid
```


```c
# 其中10235是进程的pid  
$ gcore -o coredump 10235  
  
$ ll coredump*  
-rw-r--r-- 1 work work 3.7G 2021-11-07 23:05:46 coredump.10235
```

## 流程 

(1) 查看进程PID
(2) `gcore -o coredump_file PID`
(3) `gdb 可执行文件路径  coredump_file `



# 范例

# 问题 

## `gcore`不要和`strace`混合使用

### 问题

比如，基于当前的named来生成gcore。
```bash
# ps -ef |grep -i -e named -e gcore
named    12102     1  9 May23 ?        09:01:21 /opt/bind/sbin/named -u named -c /etc/named.conf -t /var/named/chroot
```

使用如下的命令， 会出现 `strace` 卡柱，无法生成对应的 `coredump`文件。
```bash
strace -tt -T -f gcore -o /var/named/logs/named_aaa2.core 12102

注：如果直接通过   gcore -o /var/named/logs/named_aaa2.core 12102，则没有上诉问题，coredump文件正常生成。
```

![](attachments/Pasted%20image%2020240527163348.png)


![](attachments/Pasted%20image%2020240527163531.png)


# 其他

## gdb 中生成 coredump

### 背景

### `gdb`中生成`coredump`而不影响原程序运行

```bash
gdb -p PID
(gdb) gcore /var/named/logs/named_aaa2.core
(gdb) quit
```

非交互式的脚本如下所示：


## pstack 和 gcore



# 参考
```c

```