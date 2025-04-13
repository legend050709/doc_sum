```table-of-contents
```
# 介绍
# makefile
## 伪目标（phony target）


# make命令
## 常见参数

## make 调试
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


### 修改`makefile`的方式
可以直接在 Makefile 中添加一些调试信息。找到调用 `gcc` 的地方，添加 `@echo` 语句来打印命令。
例如：
```bash
$(CC) $(CFLAGS) -o my_program my_program.c 
@echo "gcc $(CFLAGS) -o my_program my_program.c"
```
## 其他
## 编译内核模块
## 范例

# 参考
```c

```
