```table-of-contents
```

# 介绍
GCC默认的连接器是`ld`，一般不直接调用它，而是通过`gcc`或`g++`调用，如：
```bash
g++ -L. -la -lb main.cpp -o main 
```


- **`-L`指定库路径，`-l`指定库名**；
    
- `-la`实际指定的库名是`liba.a`，这是库的默认命名规则，**省略掉`lib`前缀**；
    
- `-l`指定的**库的链接顺序由右向左**，这点不太自然，也就是先尝试连接`libb.a`，再`liba.a`；同样，cmake的`target_link_libraries`指定的库的链接顺序跟GCC保持一致。

# 链接顺序
在使用 GCC 链接多个库时，库的**顺序非常重要**，因为链接器（`ld`）是按**从左到右**的顺序处理依赖关系的。

## 核心规则

### 依赖者在前，被依赖者在后
若库 **A** 依赖库 **B**（即 **A** 调用 **B** 中的符号），则链接顺序应为：`A → B`。 链接器在解析符号时，只会从**已扫描的库**中提取未定义的符号。
    
### 基础库放在末尾
标准库（如 `libm`、`libpthread`）通常需要放在命令的末尾，因为用户代码可能隐式依赖它们。



## 范例
### 范例一
#### 正确顺序
```bash
gcc main.o -lfoo -lbar -lm
```
- **解析过程**：
1. 先处理 `main.o`，发现未定义的符号（如 `foo_func`）。
2. 扫描 `libfoo`，提取 `foo_func`，可能引入新的未定义符号（如 `bar_func`）。
3. 扫描 `libbar`，提取 `bar_func`，可能引入数学函数（如 `sqrt`）。
4. 扫描 `libm`，提取 `sqrt`。

#### 错误顺序
```bash
gcc main.o -lm -lfoo -lbar
```
- **问题**：
- 链接器在扫描 `libm` 时，`main.o` 尚未引入对 `sqrt` 的依赖，导致 `libm` 被跳过。
- 后续处理 `libfoo` 和 `libbar` 时，若需要 `sqrt`，会报错 `undefined reference to 'sqrt'`。

## 常见问题与解决
### 循环依赖（A 依赖 B，B 也依赖 A）

**使用链接器分组**：通过 `-Wl,--start-group` 和 `-Wl,--end-group` 让链接器重复扫描库直到解析所有符号。
```bash
gcc main.o -Wl,--start-group -lA -lB -Wl,--end-group -lm
```
**代价**：链接时间可能变长。

### 隐式依赖标准库
始终将标准库（如 `-lm`, `-lpthread`, `-lc`）放在末尾.
```bash
gcc main.o -lfoo -lbar -lm
```

### 静态库与动态库混合链接
若静态库（`.a`）依赖动态库（`.so`），需确保静态库在前：
```bash
gcc main.o -lstatic_lib -ldynamic_lib
```

### 调试技巧
#### 查看未定义符号
使用 `nm` 工具检查库中的符号：
```bash
nm -gC libfoo.a | grep "需要查找的符号"
```

#### 链接过程详细输出
添加 `-Wl,--verbose` 查看链接过程：
```bash
gcc main.o -lfoo -lbar -Wl,--verbose
```

## 使用场景
![](attachments/Pasted%20image%2020250402113919.png)
DPDK 中 `librte_ipsec` 是 对于 `librte_cryptodev`的封装， 所以前者依赖于后者。

# 选项
## `-Wl`
### 介绍
`GCC/G++` 作为编译器前端，在调用链接器 `ld` 时，需要通过 `-Wl` 将特定参数传递给 `ld`。
比如：`--whole-archive` 和 `--no-whole-archive` 是`ld`专有的命令行参数，gcc 并不认识，要通gcc传递到 ld，需要在他们前面加 `-Wl`。

### 语法格式
`-Wl` 后的逗号分隔内容会被转换为 `ld` 的参数。

```bash
-Wl,<arg1>[,<arg2>][,<arg3>]...
```

## 静态库的选项
### `-Wl,--whole-archive` 和 `-Wl,--no-whole-archive`
#### 背景

```bash
# tree ./
./
├── a.c
├── a.h
└── main.c

# cat a.h
void func();

# cat a.c
#include "a.h"
#include <stdio.h>
void func() {
	printf("a-func\n");
}

# cat main.c
#include  "a.h"
#include  <stdio.h>
int  main()  {
    func();
    printf("main\n");
    return  0;
}
```


生成静态库：
```bash
# gcc -c a.c
# ar rcs liba.a a.o

# ll
total 20
-rw-r--r-- 1 root root   71 Apr  2 10:48 a.c
-rw-r--r-- 1 root root   13 Apr  2 10:47 a.h
-rw-r--r-- 1 root root 1480 Apr  2 10:49 a.o
-rw-r--r-- 1 root root 1622 Apr  2 10:49 liba.a
-rw-r--r-- 1 root root  115 Apr  2 10:48 main.c
```

生成可执行文件：
```bash
# gcc  -L.  -la  main.c  -o  main
/opt/rh/devtoolset-9/root/usr/libexec/gcc/x86_64-redhat-linux/9/ld: /tmp/cculevCJ.o: in function `main':
main.c:(.text+0xa): undefined reference to `func'
collect2: error: ld returned 1 exit status
```

##### 分析
明明库a里有`func`，为什么链接器找不到呢？

其实这涉及链接器一条默认行为：**如果静态库里的某个目标文件的符号都没被直接或间接使用，链接器就会忽略掉这个文件，用来优化二进制文件的大小**。  

上面的编译指令，a库在前面，链接器检测到没有被用到，其中的目标文件自然就被忽略掉了，所以后面的`main.c`就找不到`func`符号了。

##### 解决方法
(1) 方法一：
最简单的改法是将`main.c`提前，告诉链接器需要用到`func`符号，从而不会将`a.o`忽略掉。
```bash
# gcc main.c  -L.  -la   -o  main

# ll
total 40
-rw-r--r-- 1 root root    71 Apr  2 10:48 a.c
-rw-r--r-- 1 root root    13 Apr  2 10:47 a.h
-rw-r--r-- 1 root root  1480 Apr  2 10:49 a.o
-rw-r--r-- 1 root root  1622 Apr  2 10:49 liba.a
-rwxr-xr-x 1 root root 16480 Apr  2 10:50 main
-rw-r--r-- 1 root root   115 Apr  2 10:48 main.c
```

(2) 方法二：
这个例子仅仅只是说明问题，现实中很多情况要复杂得多，不能像上面那样简单地调整顺序解决。所以`ld`提供了专门的链接选项`--whole-archive`。
```bash
# gcc -L. -Wl,--whole-archive -la -Wl,--no-whole-archive  main.c  -o  main
# ./main
a-func
main
```




#### `--whole-archive` 和 `--no-whole-archive` 

当程序或者动态库，链接外部静态库时，默认情况下链接器只会包含那些被当前代码显式引用的符号。使用 `--whole-archive` 可以强制包含静态库的所有符号。链接时，使用 `--whole-archive` 可以将后续指定的静态库中所有对象文件（`.o`）全部包含到输出文件中，即使这些对象文件中的符号未被直接引用。

这会：
1. **避免符号丢失**：确保静态库的所有功能可用
2. **解决符号冲突**：明确指定需要完整包含的库
3. **保障初始化代码执行**：确保静态库的全局构造函数/初始化代码被执行

#####  `--whole-archive`

##### `--no-whole-archive`
结束 `--whole-archive` 的作用，后续的静态库恢复为默认的链接方式（仅包含被引用的对象文件）。


#### 优缺点
通过`--whole-archive` 和 `--no-whole-archive` ，这种将静态库中的符号一律链接的方法，比较暴力，虽然省事；
也==会增加二进制文件的大小，所以仅在必需的时候使用。可以只对特定的静态库执行这个选项进行包含==。


#### 使用场景
##### 构建动态库时包含静态库
在编译动态库时，若需将某个静态库完全内嵌到动态库中：
```bash
gcc -shared -o libfoo.so foo.o -Wl,--whole-archive -lbar -Wl,--no-whole-archive
```
这确保 `libbar.a` 的所有代码被合并到 `libfoo.so` 中，避免运行时依赖。

#### 范例
##### dpdk的 rte_mempool 的 ops 用到了 `rte_mempool_ring`

![](attachments/Pasted%20image%2020250402112135.png)
如上所示，虽然 `rte_mempool`  的 ops 默认使用了 `rte_mempool_ring`。
但是其静态库中没有这个 `rte_mempool_ring`的 `ops_mp_mc`这个符号。如下所示：
```bash
# readelf -a /usr/local/lib/dpdklib-debug2-22.11/lib64/librte_mempool.a | grep -i ops_mp_mc

# readelf -a /usr/local/lib/dpdklib-debug2-22.11/lib64/librte_mempool_ring.a | grep -i ops_mp_mc
    27: 0000000000000000   128 OBJECT  LOCAL  DEFAULT    7 ops_mp_mc
    33: 0000000000007a20    19 FUNC    LOCAL  DEFAULT    1 mp_hdlr_init_ops_mp_mc
```

这个是因为通过构造函数的形式进行注册。不需要显示的调用。
![](attachments/Pasted%20image%2020250402111730.png)

因此，pkg-config  输出静态库的时候需要使用 `--whole-archive` 和 `--no-whole-archive`  进行包裹这些静态库。
![](attachments/Pasted%20image%2020250402111207.png)

## 动态库的选项
### `-Wl, --export-dynamic`

# 参考
```bash

```