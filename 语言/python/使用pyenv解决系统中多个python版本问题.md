```table-of-contents
```
# pyenv
## 介绍
pyenv 主要用来对 Python 解释器进行管理，可以管理系统上的多个版本的 Python 解释器。 pyenv 可以改变全局的 Python 版本，在系统中安装多个版本的 Python， 设置目录级别的 Python 版本，还能创建和管理 `virtual python environments` 。所有的设置都是用户级别的操作，不需要 sudo 命令。

## 原理

主要原理就是将新的解释器路径放在 PATH 环境变量的前面，这样新的 python 程序就“覆盖”了老的 python 程序，达到了切换解释器的目的。

pyenv 实现的精髓之处在于，它并没有使用将不同的 `$PATH` 植入不同的 shell 这种高耦合的工作方式，而是简单地在 `$PATH` 的最前面插入了一个垫片路径（shims）： `~/.pyenv/shims:/usr/local/bin:/usr/bin:/bin` 。所有对 Python 可执行文件的查找都会首先被这个 shims 路径截获，从而使后方的系统路径失效。

## 作用
通过pyenv 可以在一个系统中安装多个python版本，并在指定的作用域（比如：当前路径，当前的shell，全局范围等）切换到指定的python版本。


## 安装与配置pyenv
### 安装

```bash
$ git clone git://github.com/yyuu/pyenv.git $HOME/.pyenv
后面的`$HOME/.pyenv`是你想安装在硬盘的地址
```
尽可能不要用root用户来安装pyenv.


### 配置

如果使用的是`bash`
```bash
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.bashrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.bashrc
$ echo 'eval "$(pyenv init -)"' >> ~/.bashrc

```

如果使用的是`zsh`
```bash
$ echo 'export PYENV_ROOT="$HOME/.pyenv"' >> ~/.zshrc
$ echo 'export PATH="$PYENV_ROOT/bin:$PATH"' >> ~/.zshrc
$ echo 'eval "$(pyenv init -)"' >> ~/.zshrc

```

重启当前shell，加载新的环境变量
```bash
$ exec $SHELL -l
```

## 操作pyenv
### 更新pyenv
```bash
$ cd ~/.pyenv 
$ git pull
```
### 查看
#### 查看当前使用的python版本
```bash
# pyenv version
# which python // python的位置
or
# python --version // python2的版本
# python3 --verison // python3的版本
```
#### 查看可供pyenv使用的`python`版本
```bash
$ pyenv versions

上面命令展示已经下载完成，可供切换的python版本。
```

#### 查看所以可以安装的版本
```bash
$ pyenv install --list
```

### 安装指定python版本
```bash

$ yum install -y libffi-devel ncurses-devel bzip2-devel xz-devel  readline-devel sqlite-devel
// 安装过程中可能需要这些库

$ pyenv install <python版本>

比如：
pyenv install 3.7.2

$ pyenv rehash 
安装新版本后，需要rehash一下
```

安装的版本会在`$HOME/.pyenv/versions`目录下。比如：
```
# ls /root/.pyenv/versions/
3.7.2
```

### 切换指定的python版本
#### 当前目录设置指定版本
**pyenv local本地设置（只影响当前目录）**：它会在当前目录创建一个.python-version文件，里面记录着版本内容。
**pyenv local命令只会对当前的文件夹和其子目录中的版本起作用** ，其他的目录不起作用。
```bash
$ pyenv local <python版本>
```

##### 范例
新建目录/data/test.
```bash
[root@ops-130 test]# pyenv versions
* system (set by /root/.pyenv/version)
  2.7.8
  3.3.3
# pyenv local修改python版本  
[root@ops-130 test]# pyenv local 3.3.3
[root@ops-130 test]# pyenv versions
  system
  2.7.8
* 3.3.3 (set by /data/test/.python-version)
[root@ops-130 test]# python -V
Python 3.3.3
# 在当前目录生成.python-version版本文件
[root@ops-130 test]# ls -a
.  ..  .python-version  test2
[root@ops-130 test]# cat .python-version 
3.3.3
```
创建一个子目录test2
```bash
[root@ops-130 test]# mkdir test2
[root@ops-130 test]# cd test2
[root@ops-130 test2]# pyenv versions
  system
  2.7.8
* 3.3.3 (set by /data/test/.python-version)
[root@ops-130 test2]# python -V
Python 3.3.3
```
发现子目录也随之一起改变了
再回到父目录查看：父目录不变。
```bash
[root@ops-130 test]# cd ..
[root@ops-130 data]# pyenv versions
* system (set by /root/.pyenv/version)
  2.7.8
  3.3.3
[root@ops-130 data]# python -V
Python 2.7.5
```


#### 当前shell设置指定版本
```bash
$ pyenv shell <python版本>
```
`pyenv shell` 只影响当前的shell的python版本。不影响其他的shell的的python版本。


##### 范例
终端 1 的操作：
```bash
[root@ops-130 ~]# pyenv versions
* system (set by /root/.pyenv/version)
  2.7.8
  3.3.3
# pyenv修改python版本  
[root@ops-130 ~]# pyenv shell 3.3.3
[root@ops-130 ~]# pyenv versions
  system
  2.7.8
* 3.3.3 (set by PYENV_VERSION environment variable)
[root@ops-130 ~]# python -V
Python 3.3.3
[root@ops-130 ~]#

可以看到，使用pyenv shell切换会话里的python版本后，
终端 1 的pyenv和python显示版本均为3.3.3
```

终端2 的操作：
```bash
[root@ops-130 ~]# cd /data/
[root@ops-130 data]# pyenv versions
* system (set by /root/.pyenv/version)
  2.7.8
  3.3.3
[root@ops-130 data]# python -V
Python 2.7.5
[root@ops-130 data]#

可以看到新打开的会话是Python 2.7.5，并没有受到影响，
所以shell只会影响到当前的会话，一旦这个会话结束，则一切失效
```


#### 全局设置指定版本
使用此命令，可以看到所有受到pyenv控制的窗口都受到了影响，即全局设置，会覆盖所有的目录和窗口。
所以**尽可能不要用root用户来安装pyenv，否则会影响到之前的系统**。
```bash
$ pyenv global <python版本>
```
原理是：
该命令执行后会在 $(pyenv root) 目录(默认为 ~/.pyenv )中创建一个名为 version 的文件(如果该文件已存在，则修改该文件的内容)，里面记录着系统全局的Python版本号。

#####  范例
```bash
[root@ops-130 ~]# pyenv global 2.7.8
[root@ops-130 ~]# pyenv versions
  system
* 2.7.8 (set by /root/.pyenv/version)
  3.3.3
# 会在~/.pyenv中创建version文件
[root@ops-130 ~]# ls /root/.pyenv/ 
bin    CHANGELOG.md  completions  Dockerfile  LICENSE   man      pyenv.d    shims  terminal_output.png  version
cache  COMMANDS.md   CONDUCT.md   libexec     Makefile  plugins  README.md  src    test                 versions
[root@ops-130 ~]# cat /root/.pyenv/version
2.7.8
```

切换到其他目录查看：
```bash
# opt也被改成了2.7.8
[root@ops-130 test]# cd /opt/
[root@ops-130 opt]# pyenv versions
  system
* 2.7.8 (set by /root/.pyenv/version)
  3.3.3
```

切换到之前local设置的目录：local并没有被global覆盖
```bash
[root@ops-130 ~]# cd /data/test/
[root@ops-130 test]# pyenv versions
  system
  2.7.8
* 3.3.3 (set by /data/test/.python-version)
```

#### 取消pyenv的版本设置
```bash
# 取消当前shell窗口的
pyenv shell --unset
# 取消当前目录的
pyenv local --unset
# 取消全局设置
pyenv global system
```

#### 优先级比较
寻找 python 的时候优先级：
```bash
shell > local > global
```
pyenv 会从当前目录开始向上逐级查找 `.python-version` 文件，直到根目录为止。若找不到，就用 global 版本。

```bash
pyenv global 2.7.3  # 设置全局的 Python 版本，通过将版本号写入 ~/.pyenv/version 文件的方式。
pyenv local 2.7.3 # 设置 Python 本地版本，通过将版本号写入当前目录下的 .python-version 文件的方式。通过这种方式设置的 Python 版本优先级较 global 高。

pyenv shell 2.7.3 # 设置面向 shell 的 Python 版本，通过设置当前 shell 的 PYENV_VERSION 环境变量的方式。这个版本的优先级比 local 和 global 都要高。
pyenv shell --unset # `--unset` 参数可以用于取消当前 shell 设定的版本。
pyenv rehash  # 创建垫片路径（为所有已安装的可执行文件创建 shims，如：~/.pyenv/versions/*/bin/*，因此，每当你增删了 Python 版本或带有可执行文件的包（如 pip）以后，都应该执行一次本命令）
```


#### 查看效果

设置之后可以在目录内外分别试下`which python`或`python --version`看看效果, 
如果没变化的话可以`$ python rehash`之后再试试

### 卸载指定的版本
```bash
$ pyenv uninstall <python版本>
```

## 应用场景
pyenv 的一个典型使用场景就是，比如一个老项目需要使用 Python 2.x ，而另一个新项目需要 Python 3.x 。

## pyenv 和 virtual python environments

pyenv 的一个典型使用场景就是，比如一个老项目需要使用 Python 2.x ，而另一个新项目需要 Python 3.x 。而 virtualenv 主要是用来管理相同版本 Python 不同项目的包的依赖不同的问题，就无法解决这个问题，这个时候就需要 pyenv

pyenv 通过修改系统环境变量来实现不同 Python 版本的切换。而 virtualenv 通过将 Python 包安装到一个目录来作为 Python 包虚拟环境，通过切换目录来实现不同包环境间的切换。

pyenv 实现的精髓之处在于，它并没有使用将不同的 `$PATH` 植入不同的 shell 这种高耦合的工作方式，而是简单地在 `$PATH` 的最前面插入了一个垫片路径（shims）： `~/.pyenv/shims:/usr/local/bin:/usr/bin:/bin` 。所有对 Python 可执行文件的查找都会首先被这个 shims 路径截获，从而使后方的系统路径失效。


# 参考
```bash
# Python多环境管理神器（pyenv）
https://www.cnblogs.com/doublexi/p/15786911.html

# 使用 pyenv 管理 Python 版本
https://www.wulicode.com/python/packages/pyenv.html

# pipenv：新一代Python项目环境与依赖管理工具
https://www.wulicode.com/python/packages/pipenv.html
```