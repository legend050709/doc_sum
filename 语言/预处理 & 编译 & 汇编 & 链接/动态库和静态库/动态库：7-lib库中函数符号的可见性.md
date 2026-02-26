```table-of-contents
```
# 概述
符号的可见性对于动态库至关重要，在Linux系统中，默认情况下，动态库的符号都是全局可见的，但实际应用中，我们往往希望隐藏掉不必要的符号。
因此，我们需要一些手段来控制符号的可见性。

# 背景
比如：`libtpa` 作为一个用户态协议栈的动态库，`libtpa.so`中使用了`dpdk`的静态库中的函数，`libtpa.so`作为一个动态库供存储业务来使用。
同时存储业务也使用了`spdk`的动态库。
`DPDK/SPDK`中存在一些同名的全局变量（比如一个链表），串接各个`ops`。
加载`libtpa.so(包含DPDK的函数)` 是会给各个`ops`串接到全局变量中。
加载`spdk`的动态库，也会尝试给同一个全局变量串接各个`ops`，串接时发现`ops`重复注册了，就出错了。


## 思路
`libtpa.so`库只暴漏特定的 `tpa_*` 开头的函数，其他的`dpdk`相关的函数符号不可见，就好了。
这样：`libtpa.so`中`dpdk`相关的函数在`libtpa.so`库内是全局可见；在库`libtpa.so`之外，就不可见了。

那么就会出现：即使是同名的全局变量（比如一个链表），由于各自==在自身的库中全局可见，库外不可见（可以理解为库级别的static）==；
可以就相当于是2个全局变量，每个都是在自身的库中可见，那么就不存在重复注册的问题了。

# 符号的可见性问题
## 基础范例
我们还是以一个例子来看符号可见性的问题，假如我们有下面代码
```cpp
//a.c
int myintvar = 5;
 
int func0 () {
  return ++myintvar;
}
 
int func1 (int i) {
  return func0() * i;
}
```

```cpp
//main.cpp
extern int myintvar;
int main(){
    printf("%d",myintvar); 
}
```

我们将`a.c`编译为动态库，并查看其中的符号
```shell
> clang -fPIC -shared a.c -o a.so
> nm a.so

0000000000000f70 T _func0
0000000000000f90 T _func1
0000000000001000 D _myintvar
                 U dyld_stub_binder
```

我们发现三个符号的类型均为大写字母T或D，说明他们是global的符号，全局可见。 因此我们的`main`函数可以打印出`5`。

## 隐藏符号的可见性
如果我们想要隐藏`a.so`中的所有符号，只需要==加上`-fvisibility=hidden`==的`Compiler flag`即可，此时`a.so`中的所有符号都变成了不可见
```shell
> clang -fPIC -shared -fvisibility=hidden a.c -o a.so
> nm a.so

0000000000000f70 t _func0
0000000000000f90 t _func1
0000000000001000 d _myintvar
                 U dyld_stub_binder
```

由于这种方式会一次性hide掉所有符号，因此不够灵活，假如我们的动态库只需要导出`func1`，而隐藏`func0`和`myintvar`，该怎么做呢？我们至少有三种方法，包括使用`static`关键字，定义符号的`GNU visibility`，以及使用exported symbol list。每种方式都有各自的优缺点，我们接下来一一讨论

### 编译时：使用static关键字修饰变量和函数
在C/C++中被`static`声明的变量符号类型会变成`local`，也就是说禁止该符号被外部链接，则编译器不会为该符号生成任何信息，因此这种方式是一种最简单的方式，我们修改`a.c`如下
```cpp
static int myintvar = 5;
 
static int func0 () {
  return ++myintvar;
}
 
int func1 (int i) {
  return func0() * i;
}
```

重新编译动态库，并查看符号表
```shell
> clang -fPIC -shared a.c -o a.so
> nm a.so

0000000000000fa0 t _func0
0000000000000f80 T _func1
0000000000001000 d _myintvar
                 U dyld_stub_binder
```
我们发现`_func0`和`_myintvar`的符号类型变成了小写的`t`和`d`，说明这两个符号变成了`local`的。
虽然`static`可以隐藏符号，但是它同样限制了符号的作用域，`func0`和`myintvar`只可以在`a.c`中使用，即被`static`修饰的符号，只可在定义它们的文件中使用。

我们来看一个例子，假设我有个`b.c`如下
```c
extern int myintvar;
int func2(int x){
    return x+myintvar;
}
```
它依赖`a.c`中的全局变量`myintvar`，当我们将`a.c`和`b.c`一起编译为一个动态库时，`b.c`将无法看到`myintvar`这个符号，因为它只对`a.c`可见。
```shell
> clang -fPIC -shared a.c b.c -o lib.so
Undefined symbols for architecture x86_64:
  "_myintvar", referenced from:
      _func2 in b-ad7f57.o
ld: symbol(s) not found for architecture x86_64
```


小结一下：==使用`static`这种方式更多的是用于控制文件内的符号可见性，而不用于控制低级别的符号可见性==。
实际上，大多数函数或者变量不会依赖于static来控制符号可见性。

### 编译时：使用`visibility`属性修饰变量和函数


更常用的方法是使用GNU的visibility关键字，常用的有两个

- `default`，符号将被导出，默认可见
- `hidden`，符号不被导出，不能被其它对象使用

我们修改`a.c`的代码如下
```c
int myintvar __attribute__ ((visibility ("hidden")));
int __attribute__ ((visibility ("hidden"))) func0 () {
  return ++myintvar;
}
```

重新编译动态库并查看其符号
```shell
> clang -fPIC -shared a.c a.so
> nm a.so

0000000000000f70 t _func0
0000000000000f90 T _func1
0000000000001000 d _myintvar
                 U dyld_stub_binder
```

可见其符号类型和上面一样，`myintvar`以及`_fun0`变成了`local`的。不同的是，`_myintvar`此时对所有动态库源文件可见(前面的`b.c`)。实际上，隐藏的符号(`_myintvar`,`_func0`)将不会出现在动态符号表中，但是还被保留在符号表中用于做静态链接。

> 注意，对于用 visibility 属性指定的变量，将它声明为 static 可能会让编译器感到混淆

#### 小结
 **使用编译器属性：**
（1）在头文件中声明需要导出的函数和结构体时，使用`__attribute__((visibility("default")))`。
（2）在编译时添加`-fvisibility=hidden`选项，隐藏所有未明确标记为导出的符号。

### 链接时：使用版本脚本（version script）控制

对于符号的可见性，我们可以在链接时。
通过==链接器脚本/版本脚本（version script）来控制。
版本脚本通过global和local两个段来定义符号的可见性，global段中的符号会被导出，而local段中的符号则被隐藏== 。

具体来说，对于上面例子，我们可以使用下面的列表。
```shell
//exportmap.map
{
  global: func1;
  local: *;
};
```

接下来我们将`a.c`编译为动态库，并查看其symbol
```shell
> gcc -shared -o mylib.so a.c -fPIC -Wl,--version-script=exportmap.map
> nm mylib.so

0000000000201024 b __bss_start
...
0000000000000590 t func0
00000000000005b4 T func1
...
0000000000201020 d myintvar
```

我们看到只有`func1`是global的，说明符号表起到了作用。

# 符号的覆盖
此时如果有另一个文件`b.c`中有这样一行代码
```c
int myintvar = 10;
```

那么我们将`b.c`,`a.so`一起和`main.c`进行编译，并观察输出结果
```shell
> clang a.so b.c main.c
> ./a.out //10
```

我们发现输出结果为10，也就是说动态库中的符号被覆盖掉了。

# 参考
```bash

```