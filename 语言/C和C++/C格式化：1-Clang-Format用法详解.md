```table-of-contents
```
# 介绍

# 使用
## 使用流程
### 生成配置文件 `.clang-format`
```bash
//生成基于 LLVM 的配置文件
clang-format -style=LLVM -dump-config > .clang-format
//-style=LLVM|Google|Chromium|Mozilla|Webkit
```
### 修改配置文件
一般而言需要修改`.clang-format`中的缩进行数，每行最大字数等

### 格式化.c或者.h文件
将`.clang-format`配置文件放入待修改文件平级或上级文件夹内；
在`.clang-format`同级目录下通过`clang-format`命令行来进行格式化

```bash
clang-format -i <filename> -style=file 
//-i表示直接在源文件内进行修改，不加则将修改后的文件内容输出到终端。 
//<filename> 替换为文件路径
```

注：`clang-format`不支持多文件处理，如果需要的话需要写脚本来执行，例如:
```bash
#!/bin/sh
for file in `find . -name '*.[ch]'`
do
    clang-format -i $file -style=file
done
```

## 格式化整个目录下的指定类型的文件
### python脚本方式
一个利用 `.clang-format` 文件来 Clang-Format 整个目录的简单工具. 该工具支持递归目录进行格式化的功能，支持的文件类型为： `['.h', '.cpp', '.cu', '.cuh', '.hpp', '.cc', '.c']`.

#### 脚本
```python
# -*- coding: utf-8 -*-

import os
import argparse
import subprocess

parser = argparse.ArgumentParser()
parser.add_argument('-b',
                    '--bin',
                    required=True,
                    help='path of executable clang-format')
parser.add_argument('-dir',
                    '--directory',
                    required=True,
                    help='the target directory to be formatted')

types = ['.h', '.cpp', '.cu', '.cuh', '.hpp', '.cc', '.c']
processes = []


def format_file(cmd, path):
    cmd = cmd + path
    processes.append(subprocess.Popen(cmd, shell=True))


def format_dir(cmd, root):
    dirs = os.listdir(root)
    if len(dirs) == 0:
        return 0

    for dir in dirs:
        path = os.path.join(root, dir)
        if os.path.isdir(path):
            format_dir(cmd, path)
        else:
            for suffix in types:
                if path.endswith(suffix):
                    format_file(cmd, path)


def main(cmd, dir):
    if not cmd or cmd.find('clang-format') < 0:
        print('Invalid clang-format path!')
        return -1

    cmd += ' -style=file -i -fallback-style=none '

    if os.path.isdir(dir):
        format_dir(cmd, dir)

    for p in processes:
        p.wait()


if __name__ == "__main__":
    args = parser.parse_args()
    main(args.bin, args.directory)

```
#### 流程
- （1）安装 `clang-format` 和 `Python`.

```shell
clang-format --version
```

- （2）使用 `clang_format_dir.py` 脚本格式化代码.
    - Step 1: 拷贝 `clang_format_dir.py` 到你的工作目录 `working_dir`.
    - Step 2: 拷贝你自定义的 `.clang-format` 配置文件到你的工作目录 `working_dir`.
    - Step 3: 执行脚本 `clang_format_dir.py`


#### `clang_format_dir.py` 使用说明
`clang_format_dir.py` 有两个必要参数:
- `-b [CLANG_FORMAT_EXE]`: 用于声明可执行的 `clang-format` 文件路径`CLANG_FORMAT_EXE`.
- `-dir [TARGET_DIR]`: 用于声明想要格式化代码的目录 `TARGET_DIR`.

#### 范例
- 当前案例的工作目录如下:
```bash
---- working_dir
|
---- src
    |
    -- hello_world.cc
    |
    -- hello_world.h
|
---- util
    |
    -- util.cc
    |
    -- util.h
|
-- BUILD
|
-- WORKSPACE
```

- 若我们想要格式化目录 `src` 下的所有代码, 执行:
```bash
python3 clang_format_dir.py -b $(which clang-format) -dir src
```

- 若我们想要格式化当前整个工作目录下 `.` 的所有代码文件, 执行:
```bash
python3 clang_format_dir.py -b $(which clang-format) -dir .
```


# mac下配置clang-format
## 配置
```bash
rm -rf ./clang-format/ && mkdir -p ./clang-format
curl 'http://releases.llvm.org/8.0.0/clang+llvm-8.0.0-x86_64-apple-darwin.tar.xz' -o './clang-format/clang-format.tar.xz'
tar xvfJ clang-format/clang-format.tar.xz -C ./clang-format
rm -f /usr/local/bin/clang-format
sudo mv $(pwd)/$(find clang-format | grep bin/clang-format$) /usr/local/bin/clang-format
clang-format --help
```

或者
```bash
brew install clang-format

安装完，一般在：
/Library/Developer/CommandLineTools/usr/bin/clang-format

设置PATH
```
# git 联合 clang-format

# 参考
```bash
# Clang-Format用法详解
https://juejin.cn/post/7252500978556649528

```