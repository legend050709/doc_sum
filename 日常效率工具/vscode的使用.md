```table-of-contents
```

# vscode 三方插件
如下所示，日常主要是C语言的开发，安装了下面的插件。
![[attachments/Pasted image 20250207184806.png]]

# vscode的配置
## settings.json的方式进行配置vscode
vscode菜单： `Code -> Preferences -> Settings`(快捷键command + ，)，如下方式，打开配置文件`settings.json`。

![[attachments/Pasted image 20250207183058.png]]

### 编辑代码后自动保存

![[attachments/Pasted image 20250207183546.png]]

### 代码保存后自动格式化
在`settings.json`文件里面添加如下：

```json
// "editor.formatOnType": true,
// 写这一个就可以
"editor.formatOnSave": true
```

# vscode阅读C
## 插件安装
### `C/C++ Extension Pack`插件
安装完了`C/C++ Extension Pack`之后，可以实现跳转。
`command + 鼠标左键`，实现跳转到函数定义的地方或者函数调用的地方。
`ctrl + -`实现回退到跳转前的位置。

# vscode阅读C++
在VSCode中实现C++代码跳转，主要通过clangd插件或微软C/C++插件完成。推荐优先使用`clangd+compile_commands.json`方案以获得最佳体验，以下是具体配置方法和对比分析。

## clangd插件方案（推荐）

vscode 自带的 C/C++ 插件(ms-vscode.cpptools)提供了很多功能，比如 IntelliSense(代码补全、跳转)、调试等，其中的 IntelliSense 功能问题非常多，官方的维护也不太积极，所以可以选用更稳定的 clangd 插件。

clangd是LLVM项目提供的`C/C++/Objective-C语言服务器`，基于Clang编译器实现。
clangd 和 c/c++ 插件都是用来函数跳转的，c/c++ 适合小项目，clangd 适合大项目。

优势: 跳转速度快，使用编译时的符号链接导出，能精确跳转到某个文件
劣势: 更新cmake后，还需要重新导出操作符号（通过脚本来实现），好定位到头文件

### 功能介绍

clangd的强大之处在于，其基于真实的编译过程来分析代码。比如在一个大型 C++ 项目中，当你定义了一个复杂的模板类，Clangd 能够准确理解模板的实例化过程、参数类型等，而不是像一些简单的代码分析工具，仅仅通过文本匹配来提供有限的功能。 它还能在你输入代码的同时，实时检查语法错误、类型不匹配等问题，并给出详细的诊断信息，就像是一个严格又耐心的老师，时刻帮你把关代码质量。

clangd是一款在`vscode/trae/cursor`等编辑器中用来开发`c/c++`的==插件==，clangd插件支持在`c/c++`项目中实现如下功能：

**（1）代码补全（Code Completion）**

- 智能上下文感知：深入理解代码的语法结构、类型系统以及项目中的各种符号。比如，当你在一个复杂的 C++ 项目中输入一个类名后再输入点号，Clangd 能瞬间罗列出该类的所有成员函数和变量，即便是模板类也不在话下，它能准确识别模板参数，给出正确的补全内容。
- 片段补全（Snippets）：为你节省大量的敲代码时间。当你输入 “for”，它会自动补全一个完整的 for 循环模板，包括初始化、条件判断和自增部分，你只需根据实际需求修改关键部分即可。
- 优先级排序：它还会根据符号的使用频率进行优先级排序，高频使用的符号会优先显示，让你在众多补全选项中能快速找到常用内容 。

**（2）代码诊断（Diagnostics）**

- 实时错误检查：语法错误、类型不匹配、未定义符号等。
- 集成 Clang-Tidy：静态代码分析（如内存泄漏、代码风格违规）。

**（3）代码跳转**

- 跳转定义（Go to Definition）：快Clangd 的跳转定义功能就像是给代码世界绘制了一张精准的地图，让你轻松穿梭其中。当你想查看某个变量或函数的定义时，只需将光标放在符号上，按下快捷键（如 F12），Clangd 便能瞬间带你跳转到其定义处，哪怕定义在不同的文件中，也能快速定位 。
- 查找引用（Find References）：查找引用功能也同样强大，它能列出代码库中所有使用该符号的位置，让你清晰了解符号的调用关系，无论是追踪代码逻辑，还是进行代码修改，都能做到心中有数 。。
- 符号大纲（Outline）：符号大纲功能还能展示整个文件的结构，类、函数、宏等一目了然，方便你快速把握文件的整体架构 。

**（4）代码重构（Refactoring）**

- 重命名符号（Rename Symbol）：Clangd 的重命名符号功能堪称神器，当你需要修改一个变量名或函数名时，只需选中符号，按下重命名快捷键（如 F2），Clangd 会自动在整个项目中搜索并替换所有相关引用，跨文件的批量修改也能轻松完成，确保代码的一致性，大大减少手动修改的工作量和出错概率 。
- 提取函数/变量：如果你发现一段代码逻辑复杂，想将其提取为一个独立的函数或变量，Clangd 也能助你一臂之力。它会根据代码的语义和上下文，给出合理的提取建议，帮助你优化代码结构，提高代码的可读性和可维护性 。


**（5）文档提示（Hover Information）**

- 类型/参数说明：悬停显示变量类型、函数签名、注释文档。在阅读和编写代码时，对于一些复杂的符号，我们常常需要了解其详细信息 。Clangd 的悬停提示功能就像一个贴心的小秘书，当你将鼠标悬停在变量、函数等符号上时，它会立即显示出变量的类型、函数的签名，以及可能存在的文档注释，让你无需翻阅大量代码就能快速理解符号的含义和用法 。

- 宏展开预览：直接显示宏定义展开后的代码。对于宏定义，Clangd 更是提供了宏展开预览功能。你可以直接查看宏定义展开后的代码，直观了解宏在代码中的实际作用，避免因宏的复杂性而产生理解误区 。


### 环境准备
#### 禁用冲突插件
关闭VSCode的C/C++插件（Ctrl+Shift+X搜索`C/C++`禁用）

```bash
// setting.json
{
    "C_Cpp.intelliSenseEngine": "Disabled"
}
```
#### 安装clangd
```bash
# Ubuntu示例 
sudo apt install clangd-12  # 或从GitHub下载最新版`
```

#### compile_commands.json
Clangd 要发挥出最佳效果，离不开一个关键文件 ——`compile_commands.json`，它包含了项目中每个源文件的详细编译命令，如使用的编译器、包含路径、宏定义、编译旗标等 。Clangd 通过这个文件了解项目的编译方式，从而进行精确的代码分析。

##### 生成compile_commands.json

不同的编译方式，导出符号的方法也不同;

要求：
（1） 修改配置文件后需要重启clangd服务器
（2）确保compile_commands.json在项目根目录

```bash
1. 重启服务: control + shift + p, 然后 Clangd: Restart language server
```

###### CMake项目

对于使用 CMake 作为构建系统的项目，生成 compile_commands.json 文件很简单。你只需在 CMakeLists.txt 文件中添加 “set (CMAKE_EXPORT_COMPILE_COMMANDS ON)” ，或者在执行构建命令时带上参数 “cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=ON ..” ，重新执行 cmake 命令后，在 build 目录下就会生成 compile_commands.json 文件 。为了方便 Clangd 识别，你可以将这个文件软链接或复制到项目根目录 。


在CMakeLists.txt中添加：
```bash
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

比如：
cd build
cmake -DCMAKE_EXPORT_COMPILE_COMMANDS=1 ..
mv compile_commands.json ../
```

###### Makefile项目

如果项目使用 Makefile 作为构建系统，你可以借助 compiledb 工具来生成 compile_commands.json 文件。首先确保你已经安装了 Python，然后在命令行中输入 “pip install compiledb” 安装 compiledb 工具 。生成编译数据库时，使用 “compiledb -n make” 命令仅生成 json 配置文件，不执行编译；若要执行编译并生成 json 配置文件，则使用 “compiledb make” 命令 。执行完成后，就能在项目目录中看到生成的 compile_commands.json 文件了 。

使用bear工具生成：
```bash
bear -- make  # 首次执行前先make clean
```


### VSCode配置
#### 安装clangd插件
扩展商店搜索`clangd`安装
#### 配置settings.json
配置settings.json：（Ctrl+Shift+P → `Preferences: Open Workspace Settings`）
```bash
{
  "clangd.arguments": ["--compile-commands-dir=${workspaceFolder}"],
  "clangd.path": "/usr/bin/clangd"  # 实际路径
}
```

说明：
```bash
{
    "clangd.arguments": [
      // 在后台持续索引代码，提高后续跳转和补全的速度
      // 特别适合大型项目，可以在你编码时建立索引
      "--background-index",

      // 指定compile_commands.json的查找目录
      // ${workspaceFolder}是VSCode的工作区目录变量
      "--compile-commands-dir=${workspaceFolder}",

      // 代码补全时显示详细信息
      // 会显示函数的完整签名、参数类型、返回值等
      "--completion-style=detailed",

      // 自动插入头文件的策略，使用"include what you use"规则
      // 只包含直接使用的头文件，减少不必要的包含
      "--header-insertion=iwyu",

      // 预编译头文件存储在内存中
      // 提高性能，但会占用更多内存
      "--pch-storage=memory",

    ]
}
```

其他常用参数：
```bash
"--all-scopes-completion"    // 显示所有范围的补全项
"--log=verbose"             // 详细日志，调试时有用
"--enable-config"           // 启用配置文件
"--query-driver=/usr/bin/g++"  // 指定编译器路径
"--clang-tidy"             // 启用clang-tidy检查

```


### 跳转操作
- **定义跳转**：F12或Ctrl+左键点击符号
- **引用查找**：Shift+F12

- **快速跳转**：‘command’ + 鼠标左键单击函数
- **返回跳转前**：‘control’ + ‘-’

## 微软C/C++插件方案
### 基础配置

![](attachments/Pasted%20image%2020260605165904.png)

- **安装插件**：扩展商店搜索`C/C++`安装
- **配置c_cpp_properties.json**（项目根目录.vscode/下）：
```bash
{
  "configurations": [
    {
      "name": "Linux",
      "includePath": ["${workspaceFolder}/**", "/usr/include"],
      "compilerPath": "/usr/bin/clang",
      "cStandard": "c17",
      "cppStandard": "c++14",
      "intelliSenseMode": "linux-clang-x64"
    }
  ],
  "version": 4
}
```

### 高级配置（可选）

- **指定compile_commands.json**（提升重载函数跳转准确性）

```bash
{
  "configurations": [
    {
      "compileCommands": "${workspaceFolder}/compile_commands.json"
    }
  ]
}
```

### 跳转操作

- **定义跳转**：F12或Ctrl+左键
- **引用查找**：右键符号 → `Find All References`

## 方案对比

|**特性**|**clangd方案**|**微软C/C++插件方案**|
|---|---|---|
|**依赖文件**|必须compile_commands.json|可选compile_commands.json|
|**跳转准确性**|⭐⭐⭐⭐⭐（支持重载/模板实例化）|⭐⭐⭐（重载函数可能跳转错误）|
|**大型项目性能**|⭐⭐⭐⭐（增量索引）|⭐⭐（全量分析可能卡顿）|
|**配置复杂度**|⭐⭐⭐（需生成compile_commands.json）|⭐⭐（基础配置简单）|

## 常见问题解决
### 跳转失效

- 检查`compile_commands.json`路径是否正确
- 执行`clangd --version`确认安装成功
- 查看VSCode输出面板（Ctrl+Shift+U）选择`clangd`日志

### 多项目工作区

- 在`.vscode/settings.json`中配置`"C_Cpp.default.includePath"`隔离不同项目符号索引

## 扩展建议

- **主题优化**：推荐安装`C/C++ Theme`插件提升语法高亮效果
- **调试配置**：结合`CMake Tools`+`CodeLLDB`实现大型项目调试
- **性能监控**：通过`C_Cpp.intelliSenseCacheSize`调整缓存大小优化大型项目体验

建议根据项目规模选择方案：**中小型项目可用微软插件快速启动，大型复杂项目务必使用clangd方案**。配置完成后可通过`Ctrl+P`输入`@`符号测试全局符号跳转功能。




# vscode阅读golang
## mac上安装go开发包
1. 打开浏览器,访问`Go`语言的官方网站:[https://golang.org/dl/](https://golang.org/dl/)
2. 下载适用于`macOS`的安装包（一般是以.pkg结尾的文件）。
3. 双击下载的`.pkg`文件,按照提示进行安装。
4. 安装完成后,在终端中运行以下命令来验证Go是否安装成功:
   `go version`
   如果输出类似以下信息,说明Go语言已成功安装:
   `go version go1.19 darwin/amd64`
5. 另外,您可以通过以下命令设置Go的工作目录:
```bash
   mkdir ~/go
   echo 'export GOPATH=$HOME/go' >> ~/.bash_profile
   echo 'export PATH="/usr/local/go/bin:$PATH" ' >> ~/.bash_profile
   echo 'export GOROOT=/usr/local/go ' >> ~/.bash_profile
   source ~/.bash_profile
```

# 参考
```bash
# Mac下vscode编辑器设置
https://yu66.vip/doc/mac/007-Mac%E4%B8%8Bvscode%E7%BC%96%E8%BE%91%E5%99%A8%E8%AE%BE%E7%BD%AE.html

# VSCode中C++代码跳转的两种高效方案：clangd与微软插件对比
https://comate.baidu.com/zh/page/7ewmht2gc3s#7

# (保姆级教程)Trae中使用clangd插件实现c++代码函数列表、变量补全、代码跳转等功能
https://developer.volcengine.com/articles/7535310756422615086
```

