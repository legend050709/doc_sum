```table-of-contents
```
# 介绍
ar 用于创建、修改和提取归档文件（archive files），通常被称为静态库（static library）。

静态库是编译后的程序代码集合，它包含一组函数或其他对象文件。通过将多个目标文件合并到一个静态库中，可以将其作为单个实体进行管理和分发，并且可以减小可执行文件的大小和编译时间。

# 使用
## `ar rcs`


## `ar -t` 和 `ar -x`
因为静态库是多个`.o`文件的归档，那么就可以查看静态库下是哪些`.o`文件的 归档(`ar -t xxx.a`)。
查看了 `.o`文件之后，那么就从`.a`中提取出某个`.o`文件（`ar -x xxx.a xxx.o`）;
提取了`.o`文件之后之后，也可以从这个`.o`文件中查看，其都有哪些函数符号(`objdump -t xxx.o`)。



# 应用
## 将.o文件打包为静态库
## 将多个静态库打包为一个静态库

### 原理
从多个其他的静态库文件中提取出来`.o`文件，然后最终基于所有的`.o`文件归档，生成一个静态库。

### 使用方法
#### `mri` 脚本
#### `ar -M < mri脚本` 


## 查看静态库中的目标文件

列出静态库中的目标文件:
```bash
ar -t libyourlib.a
```

## 查看静态库中是否存在某个函数符号

### 分步查看
```bash
# 步骤1：列出静态库中的目标文件
ar -t libyourlib.a

# 步骤2：解压单个目标文件
ar -x libyourlib.a module1.o

# 步骤3：查看目标文件符号
nm -C module1.o
```

### 快速搜索
```bash
nm -C libyourlib.a | grep -w "your_symbol"

或者 
objdump -t libyourlib.a | grep "xxx"
```

# 其他
## ranlib
在 Makefile 中使用 `ranlib` 主要是为了给静态库（`.a` 文件）生成符号索引表（Symbol Table），以方便链接器快速查找库中的符号。
### 为什么需要 `ranlib`？

**静态库索引的作用**： 当使用 `ar` 命令打包目标文件（`.o`）为静态库时，默认不会自动生成符号索引。链接器在链接库时需要遍历所有目标文件查找符号，而索引表可加速这一过程。

### 在 Makefile 中的使用方式
#### 显式调用 `ranlib`
```bash
# 示例：编译静态库并生成索引
libexample.a: obj1.o obj2.o
	ar rc $@ $^   # 打包目标文件到静态库
	ranlib $@     # 生成符号索引
```
#### 使用 `ar` 的 `-s` 参数（替代 `ranlib`）
```bash
# 使用 ar 的 -s 参数自动生成索引（GNU ar 兼容）
libexample.a: obj1.o obj2.o
	ar rcs $@ $^  # rcs = replace/create + 生成索引
```

### 小结
**（1）何时需要 `ranlib`**：
- 每次修改库内容（增删 `.o` 文件）后，需重新生成索引。
- 如果使用 `ar rc`（无 `-s`），则必须显式调用 `ranlib`。
- `ar rcs` 可替代 `ranlib`


# 参考
```bash

```