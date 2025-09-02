```table-of-contents
```

# 介绍
install和cp类似，都可以将文件/目录拷贝到指定的地点。但它不仅可以复制文件，还可以设置目标文件的权限、属主、属组，并可以创建目标目录。
install通常用于程序的makefile（在RPM的spec里面也经常用到），使用它来将程序拷贝到目标（安装）目录。

# 使用方法
## 语法
```bash
SYNOPSIS
       install [OPTION]... [-T] SOURCE DEST
       install [OPTION]... SOURCE... DIRECTORY
       install [OPTION]... -t DIRECTORY SOURCE...
       install [OPTION]... -d DIRECTORY...
```
## 选项

![](attachments/Pasted%20image%2020250702143649.png)

## 使用范例
### 安装一个自定义程序到系统路径
你写了一个简单的 C 程序 `hello.c`，编译后生成 `hello`，你想把它安装到 `/usr/local/bin/` 并让所有用户运行。

```bash
gcc -o hello hello.c
sudo install -m 755 hello /usr/local/bin/
```

 成功后你可以直接输入 `hello` 执行该程序（前提是 `/usr/local/bin` 在环境变量中）。

### 创建目录并复制多个文件进去

```bash
sudo install -d /opt/myapp/{bin,lib,conf}
sudo install -m 644 config.ini /opt/myapp/conf/
sudo install -m 755 app_binary /opt/myapp/bin/
```

这种方式常用于构建安装脚本，结构清晰、安全可控。


### 保留原文件属性进行安装（推荐）

```bash
install -p original_file /backup/
```

这会保留原始文件的时间戳和权限信息。


# 参考
```bash
# Linux install 命令详解
https://www.cnblogs.com/hcgk/p/18945386
```