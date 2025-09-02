```table-of-contents
```
# 查看gdb版本
```bash
gdb -v
```

![](attachments/Pasted%20image%2020250730111240.png)

```bash
./gdb-11.2 -h
  ...
  --data-directory=DIR, -D
		     Set GDB's data-directory to DIR.
```

# 低版本gdb的问题范例
## 低版本

# 高版本的gdb 的安装
## 源码安装
### 下载源码
```bash
（1）源码目录
https://ftp.gnu.org/gnu/gdb/

(2) 下载指定版本的gdb源码
wget https://ftp.gnu.org/gnu/gdb/gdb-14.2.tar.xz --no-check-certificate
xz -d gdb-14.2.tar.xz
tar xf gdb-14.2.tar
```

### 安装编译依赖
在编译GDB之前，我们需要确保系统中安装了必要的编译工具和库。一般来说，GDB的编译依赖包括`gcc`、`make`、`bison`、`flex`、`texinfo`等。
```bash
yum install bison flex texinfo gcc make gmp gmp-devel mpfr mpfr-devel -y
```

### 配置编译环境
在开始编译之前，需要对GDB的编译环境进行配置。通过`./configure`脚本来配置编译选项。可以使用`--prefix`选项指定安装目录，使用`--with-python`选项启用`Python`支持等。

```bash
cd gdb-14.2
./configure --prefix=$HOME/.local/gdb-14.2 --with-python

```

#### `--with-python` 参数
`--with-python` 参数是 `./configure` 脚本的一个选项，用于告诉 GDB 编译过程启用 Python 解释器嵌入支持。
简单来说，就是让编译后的 GDB 内部集成 Python 解释器，使得 GDB 可以运行 Python 脚本。



### 编译GDB源码

### 安装GDB

# 参考
```bash

```