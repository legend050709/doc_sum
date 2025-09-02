```table-of-contents
```
# 背景

# 动态库的多版本管理

# 静态库的多版本管理
## 背景
在Linux环境下，静态库（通常以`.a`为后缀）不像动态库（`.so`）那样有内置的版本信息，也不支持类似动态库的**符号链接版本管理**。

## 方法
### 将版本信息嵌入库文件名中
如下所示，提供2个文件, 2个文件的 `md5` 值完全相同。
```bash
- libmylib_v1.2.3.a
- libmylib.a
```
实际提供给用户使用的永远都是 `xxx.a`文件。`xxx.vyyy.a`只是作为提供版本信息。
这样即使存在多个`xxx.vyyy.a`文件，只需要看`xxx.a`文件的`md5`值，和哪个相同，就可以知道当前的 `xxx.a`的版本。

### 从静态库中检索符号

#### strings检索
如果库在构建时将版本信息编译进了某个全局变量或字符串常量中，可以用以下命令搜索：
```bash
$ strings libmylib.a | grep 'my_version_string' 
mylib_version="1.2.3"
```

#### 自定义ELF段
编译时可以将版本信息放入自定义ELF段：
```c
__attribute__((section(".lib_version"))) const char ver[] = "1.2.3";
```

查看：
```bash
objdump -s -j .lib_version libmylib.a
```

### 创建版本信息文件
在构建静态库时，同时生成一个包含版本信息的文本文件（如`version.txt`）并随库一起发布：
```bash
$ cat version.txt
1.2.3
```





# 其他
## 将 `git tag`信息传递到`Linux C`程序中作为库的版本

# 参考
```bash

```