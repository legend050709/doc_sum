```table-of-contents
```
# 基础
## 语言服务器协议（Language Server Protocol/LSP）
语言服务器协议（LSP）定义了编辑器或IDE与语言服务器之间使用的协议，语言服务器提供了自动完成、跳转定义、查找引用等语言功能。

### 为什么要提出语言服务器协议并增加语言服务器呢？

很久以前各IDE各自为营，例如Eclipse CDT（主要在Eclipse中提供C/C++支持）是由Java编写，如果一个使用TypeScript编写的IDE（例如：Visual Studio Code）想提供C/C++的支持就需要自己使用TypeScript重新写一套。

为了改善这样的局面，微软提出了LSP，将语言支持和编辑器之间增加一层抽象，编辑器/IDE只要实现LSP协议就拥有了复杂的语言支持功能，而对于语言服务器同样只需要关注实现LSP协议，而不限定使用何种语言。

## Clangd
Clangd是一个C++的语言服务器，他为C++语言提供了代码完成、编译错误、转到定义等功能。

下面是一个带有Clangd插件的Vs code,演示了代码补全功能：

![](attachments/Pasted%20image%2020260716144654.png)

### VScode安装clangd插件

可以直接在VScode中选择**View->Extensions**打开插件管理，然后搜索"**clangd**"单击安装

官方推荐`Make sure the Microsoft C/C++ extension is **not** installed`这里先忽略，在稍后配置文件中禁用。

安装后需要重载窗口，**Ctrl+Shift+P**并输入**Reload Window**

### compile_commands.json（编译数据库）

c++代码中，我们需要关注**头文件搜索路径**、**编译时打开了哪些宏**、**这些宏在编译时的赋值**，有这些输入才能准确地知道当前代码的编译环境，才能准确跳转到对应的头文件，才能正确显示这些宏开关。

c++中这类配置文件被称为**编译数据库**，即`compile_commands.json`。


`compile_commands.json`该json中的每一项详细描述了一个源文件(.c或者.cpp)如何被编译，包括头文件的搜索路径、搜索顺序、宏是否打开。通常由编译系统自动生成。

#### 普通工程如何生成compile_commands.json？

在C/C++项目开发中，`compile_commands.json`文件是代码分析工具（如Clangd、VSCode C/C++插件）的核心依赖文件，它记录了每个源文件的完整编译命令。本文将系统介绍两种主流生成方法：CMake原生支持与Bear工具拦截。

主要的已知**编译命令捕获框架及工具**:
```bash
(1) strace/ptrace
适用于 linux，strace/ptrace 是 linux 原生提供的接口和命令，可以捕获所有的命令。目前已知有多款工具都是基于 strace/ptrace 实现。

(2) bear:
bear 可以直接捕获并生成 clang 的 compile_commands.json，基于 linux 的 LD_PRELOAD 实现，因此也是只适用于 linux。
协议不是很友好，没办法直接用。

(3) 构建工具本身能力
比如 cmake、bazel、ninja 等，可以直接导出 clang 的编译数据库，也算是性能提升的途径，但是并不完善，导出来的数据，有时候有问题。
```



##### CMake生成compile_commands.json的方法（推荐）

优势分析:
- **原生支持**：无需额外工具，构建系统自动维护文件更新
- **路径准确**：自动处理相对路径转换问题
- **性能高效**：与构建过程深度集成，无额外开销


详细步骤：
**(1)启用编译数据库导出**  
在`CMakeLists.txt`文件顶部添加以下配置：
```bash
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)  # 关键配置项
```

**(2)重新生成构建系统**
```bash
mkdir -p build && cd build
cmake .. -G Ninja  # 推荐使用Ninja生成器（可选）
```

**(3)执行构建生成文件**
```bash
cmake --build .  # 或直接运行 ninja/make
```

**(4)符号链接优化（可选）**
```bash
ln -s build/compile_commands.json .  # 方便IDE直接读取
```


##### Bear工具生成方案
Bear是一个专门为clang工具链设计的编译数据库生成工具。它能够**自动捕获构建过程中的编译命令**，生成标准的JSON格式编译数据库文件。Bear让代码静态分析变得简单高效。这个功能对于代码分析、重构和IDE集成至关重要。
> 注：bear 其主要作为是将项目中参与的编译文件和关联头文件生成到 `compile_commands.json` 文件，该文件可以确保在阅读修改代码的时候更好的进行跳转。

Bear的主要功能是在构建过程中自动生成`compile_commands.json`文件。==这个JSON编译数据库记录了每个编译单元的处理信息，为clang-tidy、cppcheck等静态分析工具提供必要的构建上下文==。

**在编译源码的时候，默认情况下是在工程文件夹的顶层执行 make 指令进行编译，此时需要使用 bear 来生成 compile_command.json 文件，只需要使用 bear make 即可。既可以完成编译，同时会生成 compile_command.json 文件**，生成的文件存放到上述指定路径下即可，上面是顶层文件夹下。


**核心优势：**
- 自动化生成编译数据库
- 支持多种构建系统（Make、CMake、Autotools等）
- 生成标准compile_commands.json文件
- 简化静态分析工具配置，简化clang工具链的集成过程


###### 适用场景
- 非CMake项目（如Makefile、自定义脚本）
- 需要拦截现有构建流程的情况

###### 操作流程

（1）工具安装

|系统|安装命令|
|---|---|
|Ubuntu/Debian|`sudo apt install bear`|
|macOS|`brew install bear`|

```bash
源码安装：

git clone https://gitcode.com/gh_mirrors/be/Bear
cd Bear
mkdir build && cd build
cmake -DENABLE_UNIT_TESTS=OFF -DENABLE_FUNC_TESTS=OFF ..
make -j$(nproc)
sudo make install


检查：
$ bear --version
bear 3.0.18
```


(2)拦截构建过程:

使用Bear非常简单，只需要在构建命令前加上`bear --` 或者 `bear`：
```bash
bear版本 > 2.4
$ bear -- make -j8

bear版本 < 2.4
$ bear make -j8

下面使用高于2.4版本的bear举例：

bear -- make  # 替换为实际构建命令（如 ninja、g++等）
bear -- make -j8


# 或者
bear -- cmake --build .

```

执行后，Bear会在当前目录生成`compile_commands.json`文件，这个文件包含了所有编译命令的详细信息。
> 注意：==正式编译之前需要先清理一下编译文件，否则生成的 `compile_commands.json` 文件可能不完整==。
```bash
$ make clean && bear -- make
```



(3)路径验证:
检查文件中的`directory`和`file`字段是否为绝对路径，必要时使用`sed`命令批量转换：
```bash
sed -i 's#"directory": ".#"directory": "/full/path/#g' compile_commands.json
```


##### `compile_commands.json`文件未生成处理
(1) 检查`CMakeLists.txt`配置位置是否正确（必须在`project()`声明前）
(2) 确认构建目录有写入权限：`chmod +w build/`
(3) 对于Bear工具，验证构建命令是否完整执行：`echo $?`检查退出状态

使用jq工具批量修改路径（需先安装`jq`）：
```bash
jq '.[].directory |= sub("^\./"; "/full/path/")' compile_commands.json > tmp.json && mv tmp.json compile_commands.json
```

#### Makefile && clangd && compile_commands.json && bear

**(1) Makefile**: 定义了你的项目如何编译。它知道每个源文件需要哪些编译器标志（如包含路径 -I, 宏定义 -D 等）。
**(2) clangd**: 是一个语言服务器，它需要知道这些编译器标志才能准确地分析你的代码并提供智能感知。
**(3) compile_commands.json**: 是一个 JSON 文件，它列出了项目中每个源文件的确切编译命令。clangd 使用这个文件来获取所需的编译器标志。
**(4) bear (Build EAR)**: 是一个工具，它可以“监听”你的构建过程（例如执行 make），并自动生成 compile_commands.json 文件。



#### compile_commands.json文件结构解析
典型的`compile_commands.json`包含以下关键字段：
```json
[
  {
    "directory": "/path/to/build",
    "file": "/path/to/source.c",
    "command": "gcc -I../include -O2 -c source.c"
  }
]
```

- **directory**：构建目录绝对路径
- **file**：源文件绝对路径
- **command**：完整编译命令（含所有参数）




#### vscode使用compile_commands.json进行索引
大多数现代代码编辑器 和 IDE 都能自动检测项目根目录下的 `compile_commands.json` 文件并使用 `clangd`。
你可能需要在 VS Code 的设置中指定 `clangd` 的路径，如果它不在系统 `PATH` 中或者你想使用特定版本的 `clangd`。

在拥有`compile_commands.json`文件后，可以通过`--compile-commands-dir`选项来指定数据库地址。
vscode使用的是LSP(language server protocol)架构，后端使用的是`clangd server`，前端使用`clangd`插件，另外还需要vscode配置用于连接前后端。

![](attachments/Pasted%20image%2020260716170849.png)


#### 开始编码

一旦 clangd 和 compile_commands.json 配置妥当，你的编辑器 就应该能提供以下功能：

(1) 精确的错误/警告提示 (diagnostics)
(2) 代码补全 (code completion)
(3) 跳转到定义/声明 (go to definition/declaration)
(4) 查找引用 (find references)
(5) 悬停提示类型信息 (hover for type information)
(6) 重构功能 (renaming, etc.)


#### 重要注意事项和故障排除

**(1) 更新 compile_commands.json**: 
如果你更改了 Makefile 中的编译选项、添加/删除了源文件，或者修改了包含路径，你需要重新运行 bear -- make ... 来更新 compile_commands.json。

**(2)make clean 的重要性**: 
如果不 make clean，bear 可能只会记录下实际被重新编译的文件的命令。对于初次生成或重大更改后，make clean 非常重要。

**(3)编译器包装器**: 如果你使用像 ccache 这样的编译器包装器，bear 通常能够正确处理。如果遇到问题，可以尝试暂时禁用它们来生成 compile_commands.json。

**(4)路径问题**: 
compile_commands.json 中的路径应该是相对于 "directory" 字段的，或者使用绝对路径。bear 通常能正确处理。

**(5)clangd 找不到标准库头文件**: 偶尔会发生这种情况，特别是如果你的 clangd 和用于编译项目的编译器来自不同的安装或版本。确保你的编译器（如 gcc）安装正确且包含标准库头文件。
有时 clangd 需要一些提示，可以通过在项目根目录创建 .clangd 配置文件并指定 --query-driver 或 -isystem 选项来帮助它找到系统头文件，但这通常是更高级的故障排除。

**(6)检查 clangd 日志**: 
如果遇到问题，大多数编辑器允许你查看 clangd 的日志输出，这可以提供诊断问题的线索。




# C++工程配置
## `.vscode`目录
### settings.json 文件
在目标工程目录下新建`.vscode`目录，在`.vscode`目录下新建`settings.json`文件，注意这个配置文件只对当前工程生效，`settings.json`中输入以下内容：



# 参考
```bash
# 在macOS上用VSCode写C++代码 5 开启VSCode的大门
https://yang-xijie.github.io/LECTURE/vscode-cpp/5_%E5%BC%80%E5%90%AFVSCode%E7%9A%84%E5%A4%A7%E9%97%A8/

# 5分钟掌握cmake(20): VSCode 里的 C++ 跳转：cmake-file-api 介绍
https://zhuanlan.zhihu.com/p/707532724

# macOS 下让 VSCode Clangd 识别 C++11 语法
https://zhuanlan.zhihu.com/p/670005031

# vscode配置c++代码跳转demo
https://juejin.cn/post/7370993837303414822
```