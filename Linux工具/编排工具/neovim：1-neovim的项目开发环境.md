```table-of-contents
```
# 背景
在自己的`mac`上，使用 `vscode` IDE工具 查看 `Linux` 内核代码，会比较慢，比如查找，以及函数跳转存在明显的卡顿。这是因为 `Linux` 内核代码很是庞大，而 一般的`PC`(比如`Mac`笔记本)的内存，以及CPU又是有限的，而不是说`vscode` IDE工具存在较大的性能问题。

既然是硬件的配置存在问题，那么可以考虑直接在公司的开发机（`CPU`，以及内存的配置比较高）上直接查看内核代码。直接在`Linux`设备上查看代码，缺乏好的编排工具，直接通过`Vim`，又比较繁琐。那么 `neovim` 编排工具就出现了。

# Neovim(Nvim)
## 为什么选择 Neovim
## Nvim 配置基础知识
### Lua 语言

# LazyVim 
## 介绍
`LazyVim` 是由 `lazy.nvim` 提供支持的 `Neovim` 配置，即：`LazyVim`是  `Neovim` 的配置，可以轻松自定义和扩展相关的配置。
通过`LazyVim`就不需要再在从头开始进行配置，它预先已经配置了大量的插件，可以做到开箱即用的效果，能够瞬间让`Neovim`拥有其他`IDE`的目录、补全、跳转等功能；这里需要再重复解释一下，==`LazyVim`并不是`Neovim`的替代品，它只是按照「约定大于习惯」的原则把一些常用、好用的插件、配置预置到了配置文件里==。
所以，要想使用`LazyVim`，首先需要安装`Neovim`，然后再安装`LazyVim`，当打开`Neovim`时，它会自动加载配置和插件，迅速完成`Neovim`的配置。


# 参考
```bash

# 为什么放弃 Vim 而选择 Neovim? 【lazyvim的介绍】
https://xie.infoq.cn/article/264b821dd959274a013bbe8ce?utm_campaign=geektime_search&utm_content=geektime_search&utm_medium=geektime_search&utm_source=geektime_search&utm_term=geektime_search


# 从 VSCode 迁移到 Neovim 的体验【neovim的一些相关的配置】
https://xie.infoq.cn/article/17c179d138d1f5046c4761955?utm_campaign=geektime_search&utm_content=geektime_search&utm_medium=geektime_search&utm_source=geektime_search&utm_term=geektime_search

# Neovim 配置全面解析（上）
https://zhuanlan.zhihu.com/p/688749817

```