```table-of-contents
```
# meson
## 介绍
Meson（The Meson Build System）是个项目构建系统，类似的构建系统有 `Makefile`、`CMake`、`automake` …。 Meson 是一个由 Python 实现的开源项目，其思想是，开发人员花费在构建调试上的每一秒都是浪费。
```text
Meson is an open source build system meant to be both extremely fast, and, even more importantly, as user friendly as possible.
（Meson是一个开源的编译系统，非常快速和用户友好。）
```
## Meson 的依赖
Meson 是依赖 Python 与 Ninja 实现的，依赖的版本如下：
- Python (version 3.6 or newer)
- Ninja (version 1.8.2 or newer)

## Meson的特性
特性：

- 支持多平台，比如Linux, macOS, Windows, GCC, Clang, Visual Studio
- 支持多种语言，比如C, C++, D，Fortran, Java, Rust
- 在可读性和用户友好的非图灵完整DSL中构建定义
- 多种操作系统和裸机的交叉编译
- 在不牺牲正确性的情况下，针对极快的完整和增量构建进行了优化


## 安装meson
### pip安装
```bash
# pip3 install meson 
# pip3 install ninja
```

可通过`pip3 install meson`命令安装，如果在root环境下，它会在系统范围内安装。
相反,你也可以使用 `pip3 install --user meson`命令来为`user`用户单独安装，此过程不需要任何特殊权限. Meson会被安装到`~/.local/`目录下,所以你需要将 `~/.local/bin`添加至你的`PATH`.


### yum安装
```bash
# yum install -y meson ninja-build
# 或者
# dnf install -y meson ninja-build
```

### 安装高版本的meson
```bash
比如：出错如下所示：
meson.build:4:0: ERROR:  Meson version is 0.47.2 but project requires >= 0.53.2.
```



## meson.build描述文件
`meson`和`makefile` 一样，需要写描述文件告诉`meson`要构建什么，这个描述文件 就是`meson.build`，`meson`根据`meson.build`中的定义生成具体的构建定义文件`build.ninja`, `ninja`根据`build.ninja`完成具体构建。
所以，不像`make`直接根据`Makefile`文件完成构建，`meson`  
需要和`ninja`配合一起完成构建。

## meson使用
### 查看版本
```bash
meson --version
```
### 查看支持哪些编译选项
通过 `meson configure` 命令可以查看 Meson 内置的编译参数、默认值以及可选值。
```
项目目录如下：
meson_project
├── build               # Meson的构建目录
├── meson.build         # Meson的配置文件
└── meson_project.c     # C/C++源文件
```

```bash
# 进入Meson项目的根目录
$ cd meson_project

# 查看Meson的编译参数
$ meson configure

```
### 指定编译参数
Meson 还支持在生成项目编译配置时，通过 `-D` 指定编译参数

```bash
# 进入Meson项目的根目录
$ cd meson_project

# 指定编译参数，生成输出目录
$ meson build -Dprefix=/usr -Dtests=disabled

# 进入输出目录
$ cd build

# 编译代码
$ ninja -j8
```

### Meson 打印编译信息
通过 `--verbose` 参数，Messon 和 Ninja 可以打印详细的编译信息，包括编译项目时，执行的所有命令。
```bash
# 进入输出目录
$ cd build

# 编译代码
$ meson compile --verbose

# 或者

# 编译代码
$ ninja --verbose

注： meson compile 等价于 ninja.
```

# ninja
## 介绍
项目开发中一般将 Meson 和 Ninja 配合使用，Meson 负责构建项目依赖关系，Ninja 负责编译代码。

## 安装
## 使用

![](attachments/Pasted%20image%2020241209145938.png)

### 查看版本
```bash
ninja --version
```
### 编译
#### 多线程编译
ninja也可以使用多线程编译，类似于make的多线程编译。

```bash
cd dpdkbuild
ninja -j8 // 多线程编译
ninja install
```
#### 清理构建产物
类似于`make clean`, 如果 `build.ninja`文件中实现了`clean`的动作，`ninja clean` 也可以清理之前的编译产物。

如果需要更改构建配置，可以在构建目录下运行以下命令重新配置项目：
```bash
meson reconfigure

或：删除构建编译的目录，重新meson构建。
rm -rf meson_build
mkdir -p meson_build
meson meson_build
```

### 安装和取消安装

`build.ninja` 如果存在 `install`, `uninstall` 选项， 
则可以通过执行 `ninja install` 和 `ninja uninstall`, 将编译产物放入到指定的目录。

> 注：类似于 `make install` 和 `make uninstall`, `install` 和 `uninstall` 并不是 `ninja`的 编译选项，而是在 `ninja`文件，（比如：`build.ninja`，类似于Makefile ) 文件中实现了 `install` 和 `uninstall` 的动作。

#### 安装
如果 `build.ninja`文件中存在 `install` 动作。
```bash
ninja install
```
#### 取消安装
如果 `build.ninja`文件中存在 `uninstall` 动作。
```bash
ninja uninstall
```

# meson+ninja和make对比
meson+ninja构建项目的流程如下所示：

**(1) 第一步**
执行： `meson build` （相当于`configure`）,会在`build`目录下生成 `build.ninja` 文件（相当于`Makefile`）和`compile_command.json`文件。

前提：使用`meson`构建前相应的源码需要存在 `meson.build`构建描述文件.

**(2) 第二步**
执行： `ninja -C build` （相当于`make`命令）
```bash
ninja -C build 等价于  cd build; ninja
```

**(3) 第三步**
执行：`ninja -C build install`（相当于make install）
```bash
ninja -C build install 等价于  cd build; ninja install
```

# 使用meson和ninja
## 构建可执行项目
### 创建项目
```bash
meson_demo
├── main.c
└── meson.build
```
`main.c` 的文件内容
```c
#include <stdio.h>

int main(int argc, char *argv[]) {
    printf("Hello World!\n");
    return 0;
}
```

`meson.build` 的文件内容：
```bash
project('meson_demo', 'c') # 指定项目名称和编程语言的类型
exe = executable('main', 'main.c') # 指定可执行文件的文件名和入口源文件
```

这个文件定义了一个 meson_demo 的工程，并且定义了 main 这个构建目标，以及 构建 使用的源文件 main.c。

### 构建项目
```bash
# 进入项目目录
$ meson_demo

# 生成构建目录
# build是构建目录的名称，可以自定义。这个是告诉meson在哪个目录下构建(这里是源码根目录下的build目录)。
# meson一定要在一个和源码独立的目录里做构建，
# 这样多次构建可以指定不同的构建目录和构建配置，相互之间不受影响。
$ meson build   # 或者 meson setup build

===========

# 进入构建目录
$ cd build

# 编译项目代码
$ ninja

# 运行可执行文件
$ ./main

meson setup builddir
```

## 构建静态库项目
### 创建静态库的项目
目录结构如下：
```bash
static_lib_project
├── meson.build
└── src
    ├── static_lib.c
    └── static_lib.h
```

`static_lib.h` 的文件内容：
```c
#ifndef _THIRD_LIB_
#define _THIRD_LIB_
 
void info_print();
 
#endif
```

`static_lib.c` 的文件内容：
```c
#include <stdio.h>
#include "static_lib.h"

void info_print()
{
    printf("hello static library\n");
}
```

`meson.build` 的文件内容：
```c
project('static_lib_project', 'c') 
#指定项目名称和编程语言的类型

static_library('static_lib', 'src/static_lib.c') 
#指定静态库文件的文件名和入口源文件
```

### 构建项目
```bash
# 进入项目目录
$ cd static_lib_project

# 生成构建目录，build是构建目录的名称，可以自定义
$ meson build   # 或者 meson setup build

# 进入构建目录
$ cd build

# 编译项目代码
$ ninja

# 项目成功编译后，会生成静态库文件"libstatic_lib.a“ 
$ ls -al
drwxr-xr-x. 6 clay clay 4096 08月 12 21:05 .
drwxr-xr-x. 4 clay clay   46 08月 12 10:13 ..
-rw-r--r--. 1 clay clay 2972 08月 12 10:13 build.ninja
-rw-r--r--. 1 clay clay  430 08月 12 10:13 compile_commands.json
-rw-r--r--. 1 clay clay 3564 08月 12 21:05 libstatic_lib.a
drwxr-xr-x. 2 clay clay   31 08月 12 21:05 libstatic_lib.a.p
drwxr-xr-x. 2 clay clay 4096 08月 12 10:13 meson-info
drwxr-xr-x. 2 clay clay   26 08月 12 10:13 meson-logs
drwxr-xr-x. 2 clay clay 4096 08月 12 10:13 meson-private
-rw-r--r--. 1 clay clay  808 08月 12 21:05 .ninja_deps
-rw-r--r--. 1 clay clay  152 08月 12 21:05 .ninja_log

```
## 构建加载第三方静态库的可执行项目
### 创建项目
目录结构如下：
```bash
load_static_lib_project
├── meson.build
└── src
    ├── include
    │   └── static_lib.h
    ├── lib
    │   └── libstatic_lib.a
    └── main.c
```

`static_lib.h` 的文件内容：
```c
#ifndef _THIRD_LIB_
#define _THIRD_LIB_
 
void info_print();
 
#endif
```

`main.c` 的文件内容：
```c
#include <stdio.h>
#include "static_lib.h"
 
int main(int argc, char *argv[]) {
        info_print();
        return 0;
}
```

`meson.build` 的文件内容：
```bash
project('load_static_lib_project', 'c') # 指定项目名称和编程语言的类型
libs=meson.get_compiler('c').find_library('static_lib', dirs : join_paths(meson.source_root(),'src/lib')) # 指定静态库文件的名称和所在目录的路径，文件名称不需要加”lib” 前缀
executable('load_static_lib', 'src/main.c', dependencies : libs, include_directories : 'src/include')  # 指定可执行文件的文件名、入口源文件、静态库的头文件所在目录的路径;  
# 使用`dependency()`函数来引用已编译的库。该函数接受一个参数，即库的名称。
# 使用`executable()`函数来定义项目的可执行文件，并将已编译的库与之关联。
```
### 构建项目
```bash
# 进入项目目录
$ cd load_static_lib_project

# 生成构建目录，build是构建目录的名称，可以自定义
$ meson build   # 或者 meson setup build

# 进入构建目录
$ cd build

# 编译项目代码
$ ninja

# 运行可执行文件
$ ./load_static_lib
```
## 编译DPDK22.11范例

# 参考
```bash
# meson ninja 简介
https://blog.csdn.net/dongleilei_/article/details/120081627
```