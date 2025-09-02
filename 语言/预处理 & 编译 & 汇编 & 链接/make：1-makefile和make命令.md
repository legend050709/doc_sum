```table-of-contents
```
# 介绍
# makefile
## 伪目标（.PHONY：phony target）
### 作用
- **避免目标名和文件名冲突导致命令不执行**
- **无条件执行命令**
- **少了文件时间戳检查，执行更快**

`.PHONY` 就是告诉 `make`： 这些目标名只是任务标签，不是文件名，别去检查文件，直接执行。

```bash
            不加 .PHONY                         加了 .PHONY
┌───────────────────────────┐         ┌───────────────────────────┐
│  1. 用户运行: make clean   │         │  1. 用户运行: make clean   │
└──────────────┬────────────┘         └──────────────┬────────────┘
               │                                      │
               ▼                                      ▼
    1. make 检查是否有同名文件                2. make 直接认为 "clean"
    (文件名 = 目标名)                              是一个伪目标
               │                                      │
    ┌──────────┴──────────┐               ┌──────────┴──────────┐
    │  clean 文件不存在   │               │   忽略文件检查      │
    │  → 执行命令         │               │   → 必定执行命令    │
    └──────────┬──────────┘               └──────────┬──────────┘
               │                                      │
    │ clean 文件存在且最新 │                  (无视文件存在与否)
    │ → 不执行命令        │                      执行命令
               │                                      │
               ▼                                      ▼
        命令可能不运行                         命令一定会运行

```




## makefile中的函数/宏
### 定义

#### 范例
```bash
define check_dpdk
	$(shell PKG_CONFIG_PATH='$(PKG_CONFIG_PATH)' pkg-config --exists libdpdk 2>/dev/null && echo y)
endef
```

这里定义了一个Make函数（或称为宏）`check_dpdk`
函数体使用`$(shell ...)`执行shell命令：
- `PKG_CONFIG_PATH='$(PKG_CONFIG_PATH)'`：将Make变量`PKG_CONFIG_PATH`传递给shell环境变量
- `pkg-config --exists libdpdk 2>/dev/null`：检查libdpdk是否存在（错误输出被丢弃）
- `&& echo y`：如果存在则输出字母"y"



### 调用

## makefile中变量

## makefile中条件表达式


# make命令
## 常见参数

## make 编译调试
### `make -n`
`make -n` 会显示将要执行的命令，但不会实际执行它们。
这可以让你==看到 `gcc` 的调用参数==。

如下所示：
```bash
# pwd
/root/usr/src/uoa-1.0.2

# ll
total 68
-rw-r--r-- 1 root root   159 Feb 14 15:32 dkms.conf
-rw-r--r-- 1 root root   398 Feb 14 15:32 Makefile
-rw-r--r-- 1 root root 47362 Feb 14 15:32 uoa.c
-rw-r--r-- 1 root root  1753 Feb 14 15:32 uoa_extra.h
-rw-r--r-- 1 root root  5411 Feb 14 15:32 uoa.h

# cat Makefile
obj-m	+= uoa.o

ifeq ($(KERNDIR), )
KDIR	:= /lib/modules/$(shell uname -r)/build
else
KDIR	:= $(KERNDIR)
endif
PWD	:= $(shell pwd)

ccflags-y := -I$(src)/../../include

ifeq ($(DEBUG), 1)
ccflags-y += -g -O0
endif

all:
	$(MAKE) -C $(KDIR) M=$(PWD) modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) modules clean

install:
	if [ -d "$(INSDIR)" ]; then \
		install -m 664 uoa.ko $(INSDIR)/uoa.ko; \
	fi
```

![](attachments/Pasted%20image%2020250214155340.png)


### `make VERBOSE=1`
可以在执行 `make` 时设置这个变量为 `1`，以显示详细的编译命令。

![](attachments/Pasted%20image%2020250214155758.png)



### `make -d`
`make -d` 将打印出详细的调试信息，包括所有的命令和参数。这是一个比较详细的输出，适合调试复杂的构建问题：

注：`make -d`的输出特别多，特别详细。


### 修改`makefile`添加打印

在Makefile中，我们经常需要添加调试信息来观察变量的值或命令的执行情况。
通常使用`echo`「在规则（rule）的命令部分」或者`info`、`warning`「在Makefile的变量赋值或函数部分」来实现。

范例，如下所示：
```bash
 $(info This is an info message)

 VAR := $(shell uname)
 $(warning The operating system is $(VAR))

 all:
     @echo "Building all..."
```



#### 规则内使用shell命令：echo 或 printf

Makefile中的规则（rule）的命令部分是由`shell`执行的，因此我们可以使用`shell`命令（如`echo`、`printf`）来输出信息。
在规则的命令部分，我们可以使用shell命令来打印调试信息。注意，命令前面如果有`@`，则表示不显示命令本身，只显示输出。如果没有`@`，则命令本身也会被显示。

> 注：**echo 和 printf，只能在`target：`「即：规则（rule）的命令部分」后面的语句中使用，且前面是个TAB**。

##### 详细介绍

**使用场景**：在规则的命令部分执行时打印信息。它是由shell执行的，因此它只在命令执行时运行。  

**特点**：
- 是shell命令，不是Makefile内置函数。
- 只在命令执行时打印信息。
- 可以输出到标准输出或重定向到文件。（如 `echo "Content" > file.txt`）
- 在命令前加上`@`可以禁止回显命令本身（只显示输出）。
- `printf`：提供格式化输出，支持 `%s`、`%d` 等格式符

**执行时机**：在规则（recipe）执行阶段运行（即 `make` 执行命令时）。

**输出位置**：标准输出（stdout）。

**用途**：在规则中打印调试信息、状态提示或生成文件内容。

**注意事项**：
- 由于`echo 和 printf`是shell命令，因此它只能在规则命令块中使用（必须以Tab开头）。
- 如果使用`@`，则命令本身不会被Make打印出来，只显示输出。
- 注意在Makefile中，`echo`命令可能会因不同的shell（如sh、bash）而略有差异，但通常行为一致。

##### 范例
```bash
debug:
    @echo "==== 调试信息开始 ===="
    echo "项目名称: $(PROJECT_NAME)"
    @printf "构建目录: %s\n" $(BUILD_DIR)
    @echo "源文件列表: $(SRCS)"
```

#### 函数内使用内置函数： info 或 warning 或 error
`$(info ...)`，`$(warning ...)`，`$(error ...)` 是 Make内置的打印函数。

##### 详细介绍

**执行时机**：在 Makefile **解析阶段**打印信息（即 `make` 开始执行前）。它不会在执行规则的命令时运行，而是在Make读取Makefile时立即执行。

**特点**：
- 是Make的内置函数。
- 输出在Make解析到该行时立即显示，因此它可以在规则之外使用。

**输出位置**：`info`的输出 是 标准输出（stdout），`warning`的输出是标准错误（stderr）。

**注意事项**：
- 由于它在解析阶段执行，因此不能依赖任何在后续定义或修改的变量（除非它在变量赋值之后被调用）。
- 可以出现在Makefile的任何位置，包括规则内部和条件语句中，但注意它是在解析时执行的。
- 无法在规则中实时打印变量（它在解析阶段已执行完毕）。

##### 使用方法
**（1）打印字符串**
```bash
$(info xxxx-msg) #输出字符串xxxx-msg，不需要加""，info后加空格。
=>  xxxx-msg  
```

**（2）打印变量**
```bash
$(info  $(TARGET_DEVICE)) #打印变量TARGET_DEVICE，变量名用$())
=> x86_64
```

**（3）字符串、变量混合打印**
```bash
$(info  TARGET_DEVICE is: $(TARGET_DEVICE)) 
=> TARGET_DEVICE is: x86_64
```

##### `info/warning/error`之间区别
info只输出信息：
```bash
$(info  TARGET_DEVICE is: $(TARGET_DEVICE)) 
=> TARGET_DEVICE is: x86_64
```

warning输出信息和对应的行号：
```bash
$(warning  TARGET_DEVICE is: $(TARGET_DEVICE)) 
=>Makefile:27: TARGET_DEVICE is: x86_64
```

error输出信息和对应的行号，并停止makefile的编译：
```bash
$(error  TARGET_DEVICE is: $(TARGET_DEVICE)) 
=>Makefile:27: *** TARGET_DEVICE is: x86_64。 停止。
```


#### 小结

|特性|`echo`|`$(info)`|`$(warning)`|`$(error)`|
|---|---|---|---|---|
|**执行阶段**|规则执行时|解析阶段|解析阶段|解析阶段|
|**输出位置**|stdout|stdout|stderr|stderr|
|**中断构建**|否|否|否|**是**|
|**显示行号/文件名**|否|否|是|是|
|**使用位置**|规则内|任意位置|任意位置|任意位置|
|**典型用途**|脚本输出|日志/配置信息|非致命警告|致命错误|

最佳实践：
- **调试规则**：用 `echo` 打印实时变量（如 `@echo "CC = $(CC)"`）。
- **静态检查**：用 `$(info)`/`$(warning)` 在解析阶段验证配置。
- **致命错误**：用 `$(error)` 阻止无效构建。
- **避免混淆**：
    - 解析阶段函数（`info`/`warning`/`error`）在规则内部使用时，会在规则执行前被解析。
    - 若需在规则中实时打印，必须用 `echo`。

### gcc 调试 和  make 调试
#### gcc 编译调试技巧
```bash
# 显示所有警告（包含额外警告）
gcc -Wall -Wextra -o program source.c

# 将警告视为错误（强制解决所有警告）
gcc -Werror -o program source.c

# 显示详细编译过程
gcc -v -o program source.c

# 仅预处理（调试宏定义问题）
gcc -E source.c > preprocessed.c
```

**a) 查看基本错误信息**
```bash
gcc -o test test.c

编译器会直接输出错误和警告信息，包括错误位置和原因。
```

**b) 启用详细模式（-v）**
```bash
gcc -v -o test test.c

显示编译器调用的详细过程，包括头文件路径、库路径和调用的子命令（如预处理器、汇编器、链接器）。
```

**c) 显示宏定义问题（-E）**
```bash
gcc -E test.c

只运行预处理器，展开所有宏，方便调试宏错误。
```

#### make编译调试技巧
```bash
# 显示实际执行的命令（但不执行）
make -n

# 详细模式（显示所有决策过程）
make -d

# 显示变量值
make -p | grep CC

# 只针对特定目标调试
make --debug=v target
# 调试指定目标的构建过程，可选调试级别：
    basic：基本调试（默认）
    verbose：详细输出
    implicit：显示隐含规则
    jobs：显示并行构建信息
    
# 重定向输出到文件
make > build.log 2>&1
```

#### makefile以及代码的编译技巧

如下所示，编译debug版本的二进制文件。

（1）Makefile中条件编译：
```bash
CC = gcc
CFLAGS = -Wall -Wextra

# 调试模式开关
DEBUG ?= 0
ifeq ($(DEBUG),1)
	# 定义 DEBUG_ENABLED 宏。
    CFLAGS += -g -ggdb3 -O0 -DDEBUG_ENABLED
else
    CFLAGS += -O2
endif

PROGRAM = app
SRC = main.c utils.c

$(PROGRAM): $(SRC)
    $(CC) $(CFLAGS) -o $@ $^
```

（2）C 代码中的条件编译：
```c
#ifdef DEBUG_ENABLED
#define DEBUG_LOG(fmt, ...) fprintf(stderr, "[DEBUG] %s:%d: " fmt, __FILE__, __LINE__, ##__VA_ARGS__)
#else
#define DEBUG_LOG(fmt, ...)
#endif

void critical_function() {
    DEBUG_LOG("Entering critical function\n");
    // ...
}
```

（3）make 传递变量：
```bash
make DEBUG=1 # 构建调试版本 
gdb ./app # 调试程序
```


## make 编译传递变量


## 其他
## 编译内核模块
## 范例

# 参考
```c

```
