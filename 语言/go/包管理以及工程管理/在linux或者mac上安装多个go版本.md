```table-of-contents
```

# 背景
由于测试环境多个go项目，开发使用的go版本不一样，就需要在同一个机器上安装多个golang的版本。

# go多版本管理工具g
## g 介绍
g是一个Linux、macOS、Windows下的命令行golang工具，可以提供一个便捷的多版本go环境的管理和切换

## g的特性
- 支持列出可供安装的go版本号
- 支持列出已安装的go版本号
- 支持在本地安装多个go版本
- 支持卸载已安装的go版本
- 支持在已安装的go版本之间自由切换

## g的安装
### 安装前清空手动安装的`GOROOT`、`GOBIN`等环境变量

由于之前安装了go，并设置了GOPATH等环境变量，那么将之前的环境变量的设置给清除了。
```bash
# 注释掉 GOROOT、gobin的PATH等环境变量
sudo vi ~/.bash_profile
source ~/.bash_profile
```

### 一键安装g工具

通过下面的命令来安装g。
```bash
curl -sSL https://raw.githubusercontent.com/voidint/g/master/install.sh | bash
```

安装后，查看g的结构，以及脚本。
如下所示，是通过g 安装了 go1.22.3之后的情况。
```bash
➜  /Users/aaa ll ~/.g
total 8
drwxr-xr-x  3 aaa  staff    96B 12 12 20:41 bin
drwxr-x---  4 aaa  staff   128B 12 12 20:45 downloads
-rw-r--r--  1 aaa  staff   164B 12 12 20:41 env
lrwxr-xr-x  1 aaa  staff    36B 12 12 20:45 go -> /Users/aaa/.g/versions/1.22.3
drwxr-x---  3 aaa  staff    96B 12 12 20:45 versions

➜  /Users/aaa ll ~/.g/bin
total 23472
-rwxr-xr-x  1 aaa  staff    11M  7  7 17:54 g

➜  /Users/aaa ll ~/.g/downloads
total 174456
-rw-r--r--  1 aaa  staff   4.5M 12 12 20:41 g1.7.0.darwin-amd64.tar.gz
-rw-r--r--  1 aaa  staff    67M 12 12 20:45 go1.22.3.darwin-amd64.tar.gz

➜  /Users/aaa ll ~/.g/versions
total 0
drwxr-xr-x  18 aaa  staff   576B 12 12 20:45 1.22.3

➜  /Users/aaa ll ~/.g/versions/1.22.3
total 64
-rw-r--r--    1 aaa  staff   1.3K 12 12 20:45 CONTRIBUTING.md
-rw-r--r--    1 aaa  staff   1.4K 12 12 20:45 LICENSE
-rw-r--r--    1 aaa  staff   1.3K 12 12 20:45 PATENTS
-rw-r--r--    1 aaa  staff   1.4K 12 12 20:45 README.md
-rw-r--r--    1 aaa  staff   426B 12 12 20:45 SECURITY.md
-rw-r--r--    1 aaa  staff    35B 12 12 20:45 VERSION
drwxr-xr-x   27 aaa  staff   864B 12 12 20:45 api
drwxr-xr-x    4 aaa  staff   128B 12 12 20:45 bin
-rw-r--r--    1 aaa  staff    52B 12 12 20:45 codereview.cfg
drwxr-xr-x    7 aaa  staff   224B 12 12 20:45 doc
-rw-r--r--    1 aaa  staff   505B 12 12 20:45 go.env
drwxr-xr-x    3 aaa  staff    96B 12 12 20:45 lib
drwxr-xr-x   10 aaa  staff   320B 12 12 20:45 misc
drwxr-xr-x    4 aaa  staff   128B 12 12 20:45 pkg
drwxr-xr-x   74 aaa  staff   2.3K 12 12 20:45 src
drwxr-xr-x  364 aaa  staff    11K 12 12 20:45 test

➜  /Users/aaa cat ~/.g/env
#!/bin/sh
# g shell setup
export GOROOT="${HOME}/.g/go"
export PATH="${HOME}/.g/bin:${GOROOT}/bin:${GOPATH}/bin:$PATH"
export G_MIRROR=https://golang.google.cn/dl/
# 如上所示，设置了GOROOT，PATH设置了g的路径，以及新版本的go的bin目录。
```

### 设置环境变量
然后，每次执行 "${HOME}/.g/env" 脚本来设置g的PATH，以及go的PATH。
```bash
# mac中注释掉 git的别名g。因为git的别名g和此中的g冲突。
vim ~/.oh-my-zsh/plugins/git/git.plugin.zsh 
#注释g # alias g='git'

# 如果是mac，则在 .zshrc 中添加下面的内容
if [ -f "${HOME}/.g/env" ]; then
    . "${HOME}/.g/env"
fi
```


## g的使用

![](attachments/Pasted%20image%2020241212211604.png)

### 列出可安装的 Go 版本
```bash
g ls-remote
```
### 列出已下载的go版本
```bash
g ls
```
### 安装特定版本的 Go
```bash
g install 1.22.3
```

### 切换到特定版本
```bash
g use 1.22.3
```

### 查看go版本
```bash
go version
```


# 参考 
```bash
#  golang多版本 管理工具 g
https://blog.csdn.net/Mint6/article/details/135159666
```