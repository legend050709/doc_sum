```table-of-contents
```
# 介绍
GNU-C的一大特色（却不被初学者所知）就是`__attribute__`机制。`__attribute__`可以设置函数属性（Function    Attribute）、变量属性（Variable Attribute）和类型属性（Type Attribute）。  
`__attribute__`书写特征是：`__attribute__`前后都有两个下划线，并切后面会紧跟一对原括弧，括弧里面是相应的`__attribute__`参数。

# 使用参考
```bash
参考 ucx: src/ucs/sys/compiler_def.h

```

# `__attribute__((__packed__))`
在C或C++中，使用 `__attribute__((__packed__))` 修饰一个结构体的成员变量是没有作用的。
`__attribute__((__packed__))` 只能应用于整个结构体或联合体，而不能单独用于结构体的某个成员。

## 特性
###  `__packed__` 的作用范围：整体结构体
当你将整个结构体声明为 `__packed__` 时，结构体内的所有成员（包括嵌套的结构体）都将被紧凑排列，不会有额外的填充。
```c
struct __attribute__((__packed__)) MyStruct {
    char a;    // 1 byte
    int b;     // 4 bytes
    short c;   // 2 bytes
};

```
在上面的例子中，`MyStruct` 的内存布局将是紧凑的，`a`、`b`、`c` 将依次排列，没有填充。

### 对结构体的单个成员的无效性
如果你尝试将 `__attribute__((__packed__))` 应用于结构体的单个成员，它不会有任何效果，因为这个属性只影响结构体的整体布局，而不会影响单个成员的对齐。
```c
struct MyStruct {
    char a;  // 1 byte
    int b __attribute__((__packed__));  // 无效
    short c; // 2 bytes
};

```
在这个例子中，`b` 的 `__packed__` 属性是无效的，`MyStruct` 的整体布局仍然会根据 `int` 和 `short` 的对齐要求进行填充。

#### 如何实现成员的紧凑排列
如果你确实需要某个成员的紧凑排列，通常的做法是将该成员放在一个单独的结构体中，并对该结构体使用 `__packed__` 属性。比如：
```c
struct Inner {
    int b;  // 4 bytes
} __attribute__((__packed__));

struct MyStruct {
    char a;  // 1 byte
    struct Inner inner; // 4 bytes, no padding
    short c; // 2 bytes
};
```
在这个例子中，`Inner` 结构体是紧凑的，因此 `MyStruct` 中的 `inner` 成员也会按照紧凑的方式排列。

## 范例
```c
$ cat bbb.c
#include <stdio.h>

struct Inner {
    char a;
    int b;
    int c;
};

struct Inner2 {
    char a;
    int b;
    int c;
}__attribute__((__packed__));

struct Outer {
    short x;
    __attribute__((__packed__)) struct Inner inner;
    long y;
} __attribute__((__packed__));

struct Outer2 {
    short x;
    struct Inner inner;
    long y;
} __attribute__((__packed__));

struct Outer3 {
    short x;
    struct Inner2 inner;
    long y;
} __attribute__((__packed__));


struct Outer4 {
    short x;
    __attribute__((__packed__)) struct Inner inner;
    long y;
};

int main() {
  int size_inner = sizeof(struct Inner);
  int size_inner2 = sizeof(struct Inner2);
  int size_outer = sizeof(struct Outer);
  int size_outer2 = sizeof(struct Outer2);
  int size_outer3 = sizeof(struct Outer3);
  int size_outer4 = sizeof(struct Outer4);
  printf("size inner:%d, size inner2:%d, size outer:%d, size outer2:%d size outer3:%d, size outer4:%d \n",
       size_inner, size_inner2, size_outer, size_outer2, size_outer3, size_outer4);
  return 0;
}
```

测试结果：
```bash
$ ./bbb
size inner:12, size inner2:9, size outer:22, size outer2:22 size outer3:19, size outer4:24
```

# `__attribute__((const))`


# `__attribute__((always_inline))`

# `__attribute__ ((noinline))`

## 背景
在使用`gdb`调试的时候，某个函数可能无法进入，无法知道具体在函数的哪个位置出现了问题。
主要是编译器优化，比如添加了`-flto`
### 作用

1. **禁止函数内联**：编译器将不会尝试将该函数内联到调用处，而是生成一个独立的函数体，并通过函数调用的方式执行。
2. **控制代码大小**：防止内联展开导致代码膨胀，特别是对于较大的函数。
3. **调试友好**：在调试时，保持函数调用栈的完整性，便于跟踪和断点设置。
4. **避免优化副作用**：在某些特定场景下，防止内联优化带来的副作用（如某些性能计数器统计）。

### 使用场景

1. **性能分析**：当使用性能分析工具（如perf、gprof）时，保持函数边界，以便准确测量函数执行时间。
2. **调试**：在调试时，确保函数调用关系清晰，避免内联导致断点设置困难或栈回溯不准确。
3. **特定内存布局**：在需要精确控制地址布局的场景下（如某些嵌入式系统或内存敏感的应用）。
4. **函数指针用途**：当函数被取地址（如用作回调函数）时，避免因内联导致函数地址问题（虽然编译器通常会对这种情况做处理，但显式声明可确保安全）。
5. **大型函数**：对于函数体较大且调用次数不多的函数，内联可能导致代码膨胀而不利于缓存，此时禁止内联。

## 使用方法对比

|控制方法|作用范围|优先级|
|---|---|---|
|`__attribute__((noinline))`|单个函数|最高|
|`__attribute__((always_inline))`|单个函数|高|
|`-fno-inline`|全局禁止内联|中|
|`-finline-functions`|全局启用内联|低|

**使用原则**：优先使用函数级属性，避免全局设置破坏其他优化。


# `__attribute__((__section__(xxx)))`自定义内存段
## 介绍
`__attribute__((section("xxx")))` 是 GCC 和 Clang 编译器支持的扩展属性，用于**将变量或函数放置到指定的段（Section）中**。它允许开发者直接控制代码或数据在内存中的布局，常见于系统底层开发（如操作系统、固件、驱动等）。

```c
__attribute__((section("xxx")))  // 常规写法
__attribute__((__section__("xxx")))  // 双下划线避免宏冲突（系统头文件常用）

注：上面的两种写法是等效的。
```

## 使用方法
### 变量
```c
// 将变量放置在 .my_data 段
__attribute__((section(".my_data"))) int config[128];

// 只读数据放入 .ro_custom 段
__attribute__((section(".ro_custom"))) const char secret_key[] = "ABCDEF";
```

在GCC文档中，`section`属性可以应用于函数和变量（包括全局变量和静态局部变量）。那么结构体呢？
==结构体的定义本身（即类型定义）通常不被分配存储空间，因此不能直接应用`section`属性==。
但是，我们可以考虑两种情况：
（1）修饰结构体类型的变量（即结构体实例）
（2）修饰结构体类型定义（即类型本身）

实际上：
- 我们可以用`__attribute__((__section__("xxx")))`修饰一个结构体变量（实例），将这个变量放入指定的段。
- 但是，不能直接将这个属性放在结构体类型定义上，因为类型定义并不直接对应存储。
然而，GCC允许将属性放在结构体、联合体或枚举类型定义上，但这些属性通常用于类型本身的特性（比如对齐、打包等），而不是用于指定段。对于`section`属性，它不能用于类型定义，因为类型定义不产生独立的数据段。
因此，正确的做法是：将`section`属性应用于结构体变量（实例）上。

### 函数
```c
// 关键函数放入 .secure_functions 段
__attribute__((section(".secure_functions"))) void encrypt_data() {
    // ...
}

// 初始化函数放入 .init 段
__attribute__((section(".init"))) void early_init() { /* 启动代码 */ }
```

### 多属性组合
```c
// 同时指定段和对齐要求 
__attribute__((section(".ccmram"), aligned(128))) uint8_t buffer[1024];
```

## 应用场景
### 性能优化
#### 缓存行伪共享(False Sharing)
**问题场景**：多核CPU中不同核心频繁访问同一缓存行内的不同变量.

```c
// 默认布局：两个频繁写入的变量可能共享同一缓存行
atomic_int counter1;
atomic_int counter2;  // 与counter1相邻，可能同一缓存行

// 核心1: 
void thread1() { while(1) counter1++; }

// 核心2:
void thread2() { while(1) counter2++; } // 导致核心1的缓存行频繁失效
```

**修复方案**：
```c
// 强制分离到不同缓存行
__attribute__((section(".core1_data"), aligned(64))) 
atomic_int counter1;

__attribute__((section(".core2_data"), aligned(64))) 
atomic_int counter2;
```


# `__attribute__(format xxx)`
`__attribute__(format xxx)`属性用于在编译时对 `printf`风格的可变参数函数进行格式字符串安全检查。它可以使编译器检查函数声明和函数实际调用参数之间的格式化字符串是否匹配。该功能十分有用，尤其是处理一些很难发现的`bug`。

```c
具体的使用如下所示： 
__attribute__((format(printf, a, b))) 
__attribute__((format(scanf, a, b)))
```

## `__attribute__((format(printf, x, y)))`


### 作用
用于在编译时对 **`printf`风格的可变参数函数**进行格式字符串安全检查。以下是详细分析：

**（1）参数类型验证**：
确保格式字符串中的占位符（如 `%d`, `%s`）与后续的实际参数类型匹配。（如 `%d` 是否对应整型、`%s` 是否对应字符串）。

**（2）参数数量验证**：
检查参数个数是否与格式字符串要求一致。

**（3）预防运行时错误** 
避免因格式字符串与参数类型不匹配导致的**内存越界、程序崩溃或安全漏洞**（如格式化字符串攻击）。

### 说明
#### 参数分析
- **`printf`**:
第一个参数 `printf` 是告诉编译器，按照 `printf` 函数的检查标准来检查；

- **`X`**：
函数所有的参数列表中，格式字符串参数(`format`)的位置索引（从 1 开始计数）。

- **`Y`**：
函数所有的参数列表中，可变参数（`...`）的位置索引（从 1 开始计数）。 （若函数无额外固定参数，`Y` 通常为 `X + 1`）

**参数位置计算**:
```c
// 示例：格式字符串是第2个参数，可变参数从第3个开始
void log_file(FILE *fp, const char *fmt, ...) 
     __attribute__((format(printf, 2, 3))); // X=2, Y=3
```

#### 使用步骤
**声明函数时添加属性**  
    在函数原型中使用 `__attribute__((format(printf, X, Y)))` 修饰。

**实现函数逻辑**  
    函数内部需通过 `va_list` 处理可变参数（参考标准 `printf` 实现）。

#### 使用限制
**（1）仅适用于 GCC/Clang**  
非 GNU 兼容编译器（如 MSVC）不支持此语法，需用 `#ifdef __GNUC__` 做条件编译。

```c
// 条件编译实现跨平台
#if defined(__GNUC__)
    #define CHECK_PRINTF_ARGS(X, Y) __attribute__((format(printf, X, Y)))
#else
    #define CHECK_PRINTF_ARGS(X, Y) // 其他编译器留空
#endif

void safe_printf(const char *fmt, ...) CHECK_PRINTF_ARGS(1, 2);
```


**（2）静态检查限制**  
格式字符串必须是**字符串常量**（编译器才能静态分析），变量传递的格式字符串无法检查。
编译器的检查仅在编译时生效，无法处理运行时动态生成的格式字符串（如 `sprintf` 生成的字符串）。
```c
// 无法触发检查（format_str 是变量）
const char *format_str = "%d";
debug_log(format_str, "hello"); // 无编译警告，但运行时错误！
```

#### 适用场景

|场景|收益|
|---|---|
|**自定义日志函数**|避免日志输出因类型错误崩溃|
|**安全敏感代码**|防止格式化字符串漏洞（如 `%n` 写内存攻击）|
|**大型项目维护**|编译器自动检查接口一致性，减少人工审查成本|

**推荐实践**：所有自定义的 `printf` 风格的**可变长参数的函数**都应添加此属性，以提升代码安全性和可维护性。


#### 使用范例
```c
#include <stdarg.h>
#include <stdio.h>

// 声明：格式字符串是第1个参数，可变参数从第2个开始
void debug_log(const char *format, ...) 
    __attribute__((format(printf, 1, 2))); // X=1, Y=2

// 函数实现
void debug_log(const char *format, ...) {
    va_list args;
    va_start(args, format);
    vprintf(format, args); // 使用 vprintf 处理可变参数
    va_end(args);
}

int main() {
    int num = 42;
    const char *text = "Hello";

    // 正确用法（编译通过）
    debug_log("Number: %d, Text: %s\n", num, text);

    // 错误示例（编译器警告！）
    // debug_log("Number: %d", text);        // 类型不匹配（%d vs char*）
    // debug_log("Text: %s");                // 参数不足
    return 0;
}
```


当格式字符串与参数不匹配时，编译器会生成明确警告（需开启 `-Wall` 或 `-Wformat`）。
```bash
gcc va_test.c -o va_test -Wall

注：需要添加  -Wall，否则可能编译的时候，存在错误但是不报错。
```
![](attachments/Pasted%20image%2020250610155025.png)

# 参考
```bash
```