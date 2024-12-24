```table-of-contents
```
# virtualenv
## 背景
Python 一个不友好的地方，就是版本管理。例如有的项目使用 Python 2.x，有的项目使用 Python 3.x，而二者之间就有很多不兼容，并且一些库只支持 Python 2.x，不支持 Python 3.x。另外，同一个库，有的项目需要用到高版本，有的项目需要用到低版本，这也会造成版本管理上的不兼容。

## 介绍
virtualenv 所要解决的是同一个库不同版本共存的兼容问题。例如项目 A 需要用到 requests 的 1.0 版本，项目 B 需要用到 requests 的 2.0 版本。如果不使用工具的话，一台机器只能安装其中一个版本，无法满足两个项目的需求。

virtualenv 的解决方案是为每个项目创建一个独立的虚拟环境，在每个虚拟环境中安装的库，对其他虚拟环境完全无影响。所以就可以在一台机器的不同虚拟环境中分别安装同一个库的不同版本。

# virtualenv 和 pyenv 
## 对比
### pyenv
pyenv 不是用来管理同一个python库的多个版本，而是用来管理一台机器上的多个 Python 版本。主要解决开发中有的项目需要用Python 2.x，有的项目需要用Python 3.x的场景。

### virtualenv
virtualenv 是一个用来创建完全隔离的 Python 虚拟环境的工具，可以为每个项目工程创建一套独立的 Python 环境，从而可以解决不同工程对 不同的Python 版本，或者不同的库的依赖问题。

假如有 A 和 B 两个工程，A 工程代码要跑起来需要 requests（python库） 1.18.4，而 B 工程跑起来需要 requests（python库） 2.18.4，这样在一个系统中就无法满足两个工程同时运行问题了。最好的解决办法是用 virtualenv 给每个工程创建一个完全隔离的 Python 虚拟环境，给每个虚拟环境安装相应版本的包，让程序使用对应的虚拟环境运行即可。这样既不影响系统 Python 环境，也能保证任何版本的 Python 程序可以在同一系统中运行。

## 结合使用
使用 `pyenv` 安装任何版本的 Python，然后用 `virtualenv` 创建虚拟环境时指定需要的 Python 版本路径，这样就可以创建任何版本的虚拟环境，这样的实践真是极好的！

# virtualenv 和 pyenv的结合：Pyenv-virtualenv
## 介绍
`pyenv-virtualenv`是`pyenv`的一个插件，作用如同`virtualenv`一样，是用来管理虚拟环境的，配合`pyenv`主体使用可做到`python`的版本管理及虚拟环境的管理。

# 参考
```bash
# Python 两大环境管理神器：pyenv 和 virtualenv
https://blog.csdn.net/qq_38831220/article/details/117713365
```