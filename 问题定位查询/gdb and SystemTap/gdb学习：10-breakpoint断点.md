```table-of-contents
```
# 断点分类
GDB 调试器支持在程序中打 3 种断点，分别为普通断点、观察断点和捕捉断点。其中 break 命令打的就是普通断点，而 watch 命令打的为观察断点

# 普通断点：break

默认情况下，程序不会进入调试模式，代码会瞬间从开头执行到末尾。要想观察程序运行的内部细节（例如某变量值的变化情况），可以借助 GDB 调试器在程序中的某个地方设置断点，这样当程序执行到这个地方时就会停下来。  
  
所谓断点（BreakPoint），读者可以理解为障碍物，人遇到障碍物不能行走，程序遇到断点就暂停执行。  
  
在 GDB 调试器中对  C、C++程序打断点，最常用的就是 break 命令，有些场景中还会用到 tbreak 或者 rbreak 命令

## 使用

break 命令（可以用 b 代替）常用的语法格式有以下 2 种。
```bash
1、(gdb) break location     // b location  

2、(gdb) break ... if cond   // b .. if cond
... 可以是表 1 中所有参数的值，用于指定打断点的具体位置；cond 为某个表达式。
整体的含义为：每次程序执行到 ... 位置时都计算 cond 的值，如果为 True，则程序在该位置暂停；反之，程序继续执行。
```

### location 参数的表示方式

|location 的值|含 义|
|---|---|
|linenum|linenum 是一个整数，表示要打断点处代码的行号。要知道，程序中各行代码都有对应的行号，可通过执行 l（小写的 L）命令看到。|
|filename:linenum|filename 表示源程序文件名；linenum 为整数，表示具体行数。整体的意思是在指令文件 filename 中的第 linenum 行打断点。|
|+ offset  <br>- offset|offset 为整数（假设值为 2），+offset 表示以当前程序暂停位置（例如第 4 行）为准，向后数 offset 行处（第 6 行）打断点；-offset 表示以当前程序暂停位置为准，向前数 offset 行处（第 2 行）打断点。|
|function|function 表示程序中包含的函数的函数名，即 break 命令会在该函数内部的开头位置打断点，程序会执行到该函数第一行代码处暂停。|
|filename:function|filename 表示远程文件名；function 表示程序中函数的函数名。整体的意思是在指定文件 filename 中 function 函数的开头位置打断点。|



## 条件断点
仅当满足特定条件时断点生效，适合循环、分支中的问题：

```bash
b 12 if x > 15  # 第12行仅当 x>15 时中断
b add if a == 10 # 函数 add 仅当参数 a=10 时中断
```

### 函数名
### 文件名+行号
### 条件设置

## commands
**断点命令列表**，让==GDB在每次到达某一断点时自动执行一组命令==。

### 使用方法
```bash
commands breakpoint-number // breakpoint-number为断点号，表示将以下命令添加到指定的断点上  
...
commands // 任何有效的GDB命令，一行一个，以end结束  
...
c   
end 
```

## 多线程场景下：给指定线程设置断点
```bash
- `info threads`：查看所有线程（编号、状态、所属进程）。
- `thread [线程号]`：切换到指定线程。
- `break [位置] thread [线程号]`：给指定线程设断点。
- `set scheduler-locking on`：锁定当前线程执行（防止其他线程干扰，调试单线程逻辑时常用）。
- `thread apply [线程号] bt`：查看指定线程的调用栈（如 `thread apply all bt` 查看所有线程栈）。
```

## GDB tbreak命令
tbreak 命令可以看到是 break 命令的另一个版本，tbreak 和 break 命令的用法和功能都非常相似，唯一的不同在于，使用 tbreak 命令打的断点仅会作用 1 次，即使程序暂停之后，该断点就会自动消失。

tbreak 命令的使用格式和 break 完全相同，有以下 2 种：
```bash
1、(gdb) tbreak location  
2、(gdb) tbreak ... if cond
```

其中，`location、...` 和 `cond` 的含义都和 break 命令中的参数含义相同，即表 1 也同样适用于 tbreak 命令。

## GDB rbreak 命令

和 break 和 tbreak 命令不同，rbreak 命令的作用对象是 C、C++ 程序中的函数，它会在指定开头的函数（正则表达式表示函数名）位置打断点。

rbreak 命令的使用语法格式为：
```bash
(gdb) rbreak regex
```
regex 为一个正则表达式，程序中函数的函数名只要满足 regex 条件，rbreak 命令就会其内部的开头位置打断点。值得一提的是，rbreak 命令打的断点和 break 命令打断点的效果是一样的，会一直存在，不会自动消失。




# 观察断点：watch
使用 GDB 调试程序的过程中，借助观察断点可以监控程序中某个变量或者表达式的值，只要发生改变，程序就会停止执行。相比普通断点，观察断点不需要我们预测变量（表达式）值发生改变的具体位置。
> 所谓表达式，就是包含多个变量的式子，比如 a+b 就是一个表达式，其中 a、b 为变量。

```bash
(gdb) watch cond

其中，conde 指的就是要监控的变量或表达式。
```

```bash
watch x          # 写监控：x 被修改时中断（最常用）
rwatch x         # 读监控：x 被读取时中断
awatch x         # 读写监控：x 被读或写时中断
```
## 条件观察断点

```bash
watch expr if cond

参数 expr 表示要观察的变量或表达式；参数 cond 用于代指某个表达式。
```


## 查看观察断点
```bash
(gdb) info watchpoints
```

## watch命令的实现原理
watch 命令实现监控机制的方式有 2 种，一种是为目标变量（表达式）设置硬件观察点，另一种是为目标变量（表达式）设置软件观察点。

### 软件观察点
所谓软件观点（software watchpoint），即用 watch 命令监控目标变量（表达式）后，GDB 调试器会以**单步执行**的方式运行程序，并且每行代码执行完毕后，都会检测该目标变量（表达式）的值是否发生改变，如果改变则程序执行停止。  
  
可想而知，这种“实时”的判别方式，一定程度上会影响程序的执行效率。但从另一个角度看，调试程序的目的并非是为了获得运行结果，而是查找导致程序异常或 Bug 的代码，因此即便软件观察点会影响执行效率，一定程度上也是可以接受的。

### 硬件观察点
所谓硬件观察点（Hardware watchpoint），和前者最大的不同是，它在实现监控机制的同时不影响程序的执行效率。简单的理解，系统会为 GDB 调试器提供少量的寄存器（例如 32 位的 Intel x86 处理器提供有 4 个调试寄存器， DR: debug register），每个寄存器都可以作为一个观察点协助 GDB 完成监控任务。  
  
需要注意的是，基于寄存器个数的限制，如果调试环境中设立的硬件观察点太多，则有些可能会失去作用，这种情况下，GDB 调试器会发出如下警告：
```bash
Hardware watchpoint num: Could not insert watchpoint
```

解决方案也很简单，就是删除或者禁用一部分硬件观察点。  
  
除此之外，受到寄存器数量的限制，可能会出现：无法使用硬件观察点监控数据类型占用字节数较多的变量（表达式）。比如说，某些操作系统中，GDB 调试器最多只能监控 4 个字节长度的数据，这意味着 C、C++ 中 double 类型的数据是无法使用硬件观察点监测的。这种情况下，可以考虑将其换成占用字符串少的 float 类型。  
  
目前，大多数 PowerPC 或者基于 x86 的操作系统，都支持采用硬件观点。并且 GDB 调试器在建立观察断点时，会优先尝试建立硬件观察点，只有当前环境不支持硬件观察点时，才会建立软件观察点。借助如下指令，即可强制 GDB 调试器只建立软件观察点：
```bash
set can-use-hw-watchpoints 0

注意，在执行此命令之前建立的硬件观察点，不会受此命令的影响。
```

注意，awatch 和 rwatch 命令只能设置硬件观察点，如果系统不支持或者借助如上命令禁用，则 GDB 调试器会打印如下信息：
```bash
Expression cannot be implemented with read/access watchpoint.
```

## 注意
对于使用 watch（rwatch、awatch）命令监控 C、C++ 程序中变量或者表达式的值，有以下几点需要注意：

### 局部变量的监控
- 当监控的变量（表达式）为局部变量（表达式）时，一旦局部变量（表达式）失效，则监控操作也随即失效；
### 指针的监控
- 如果监控的是一个指针变量（例如 `*p`），则 `watch *p` 和 `watch p` 是有区别的，前者监控的是 p 所指数据的变化情况，而后者监控的是 p 指针本身有没有改变指向；
### 数组的监控
- 数组的监控：这 3 个监控命令还可以用于监控数组中元素值的变化情况，例如对于 `a[10]` 这个数组，watch a 表示只要 a 数组中存储的数据发生改变，程序就会停止执行。


# 捕捉断点：catch
捕捉断点和前 2 种断点不同，普通断点作用于程序中的某一行，当程序运行至此行时停止执行，观察断点作用于某一变量或表达式，当该变量（表达式）的值发生改变时，程序暂停。而捕捉断点的作用是，监控程序中某一事件的发生，例如程序发生某种异常时、某一动态库被加载时等等，一旦目标事件发生，则程序停止执行。
> 注：用捕捉断点监控某一事件的发生，等同于在程序中该事件发生的位置打普通断点。

## 使用

建立捕捉断点的方式很简单，就是使用 catch 命令，其基本格式为：
```bash
(gdb) catch event
```
其中，event 参数表示要监控的具体事件。对于使用 GDB 调试 C、C++ 程序，常用的 event 事件类型如表 1 所示。

|event 事件|含 义|
|---|---|
|throw [exception]|当程序中抛出 exception 指定类型异常时，程序停止执行。如果不指定异常类型（即省略 exception），则表示只要程序发生异常，程序就停止执行。|
|catch [exception]|当程序中捕获到 exception 异常时，程序停止执行。exception 参数也可以省略，表示无论程序中捕获到哪种异常，程序都暂停执行。|
|load [regexp]  <br>unload [regexp]|其中，regexp 表示目标动态库的名称，load 命令表示当 regexp 动态库加载时程序停止执行；unload 命令表示当 regexp 动态库被卸载时，程序暂停执行。regexp 参数也可以省略，此时只要程序中某一动态库被加载或卸载，程序就会暂停执行。|


除表中罗列的以外，event 参数还有其它一些写法，感兴趣的读者可查看 [GDB官网](https://sourceware.org/gdb/current/onlinedocs/gdb/Set-Catchpoints.html#Set-Catchpoints)进行了解，这里不再过多赘述。



# 参考
```bash
# GDB watch命令：监控变量值的变化
https://c.biancheng.net/view/8191.html

# GDB catch命令：建立捕捉断点
https://c.biancheng.net/view/8199.html


```