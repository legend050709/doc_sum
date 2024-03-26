```table-of-contents
```
# 问题
设备先安装了操作系统，后插上网卡到设备，就会出现`/etc/sysconfig/network-scripts`目录下无该网卡对应配置文件的问题，但是`ifconfig`命令能看见系统给该网卡产生的名称。

例如系统安装后，插上新兆网卡，`ifconfig`直接结果：
![](attachments/Pasted%20image%2020240314193306.png)

但是`/etc/sysconfig/network-scripts`目录下只有其他网卡对应的配置文件。
![](attachments/Pasted%20image%2020240314193323.png)

# 影响
当前情形下，对`eth4`配置相应IP，使用`ip addr`或者`ifconfig`配置`ip`，可直接将`ip`配置到网卡上。  
但是因为无网卡`eth4`对应的网络配置文件，设备重启后将丢失配置的IP地址，导致业务中断。

# 解决方法
使用Linux自带的工具`nmtui（TUI：文本用户界面，Text-based User Interface； nm: network manager）`产生对应网卡的配置文件，默认放在`/etc/sysconfig/network-scripts`目录。
## `nmtui` 介绍
![](attachments/Pasted%20image%2020240314195338.png)

## 操作`nmtui`
(1) 输入命令行：`nmtui`，界面如下，选择` Edit a connection`
![](attachments/Pasted%20image%2020240314195428.png)

(2)  选择`Add`
![](attachments/Pasted%20image%2020240314195502.png)

(3) 选择`Ethernet`
![](attachments/Pasted%20image%2020240314195516.png)

(4)  输入该网卡的对应名称
这里的网卡名称填写`eth4`，`Device`也填`eth4`，选择`OK`
![](attachments/Pasted%20image%2020240314195544.png)

(5) 返回
可以看见多了`eth4`，选择右下角的`Back`
![](attachments/Pasted%20image%2020240314195609.png)

(6) `quit`
![](attachments/Pasted%20image%2020240314195631.png)

(7) 产生了`ifcfg-eth4`
![](attachments/Pasted%20image%2020240314195717.png)

## 证明ifcfg-eth4是网卡eth4对应的配置文件
查看`ifcfg-eth4`的`UUID`，通过命令`nmcli con show`查看所有网卡的`UUID`，可知是相匹配的。
![](attachments/Pasted%20image%2020240314195808.png)

> 注：` nmcil` 执行需要确保 `NetworkManager` 进程在执行。

# 其他
## TUI
当你开始使用 Linux 并关注关于 Linux 的网站和论坛时，你会经常遇到诸如 `GUI`、`CLI` 等术语，有时还会遇到 `TUI`。
### 图形用户界面GUI
GUI - 图形用户界面(`Graphical User Interface`)；
GUI 应用程序（或图形应用程序）基本上是指任何可以与你的鼠标、触摸板或触摸屏交互的东西。有了图标和其他视觉概念，你可以使用鼠标指针来访问功能。
![](attachments/Pasted%20image%2020240314194245.png)
### 命令行界面CLI
CLI - 命令行界面(`Command Line Interface`)；
CLI 是一个接受输入来执行某种功能的命令行程序。基本上，任何可以在终端中通过命令使用的应用程序都属于这一类。
![](attachments/Pasted%20image%2020240314194236.png)

### 终端用户界面TUI
TUI - 终端用户界面(Terminal User Interface)，也称为 基于文本的用户界面(Text-based User Interface)；
“基于文本”这个说法主要是因为你在屏幕上有一堆文本，而“终端用户界面”的说法是因为它们只在终端中使用。

TUI 基本上部分是 GUI，部分是 CLI。
你已经知道，早期的计算机使用 CLI。在实际的 GUI 出现之前，基于文本的用户界面在终端中提供了一种非常基本的图形交互。你会有更多的视觉效果，也可以使用鼠标和键盘与应用程序进行交互。
终端中的 nnn 文件浏览器，如下所示：
![](attachments/Pasted%20image%2020240314194356.png)

TUI 的应用虽然不是那么常见，但你还是有一些的。[基于终端的 Web 浏览器](https://link.zhihu.com/?target=https%3A//itsfoss.com/terminal-web-browsers/)是 TUI 程序的好例子。[基于终端的游戏](https://link.zhihu.com/?target=https%3A//itsfoss.com/best-command-line-games-linux/)也属于这一类。

如下所示，基于文本的终端`web`浏览器，`w3m`。
![](attachments/Pasted%20image%2020240314194638.png)
# 参考
```bash
# [nmtui解决network-scripts目录下无网卡对应配置文件问题](https://www.cnblogs.com/t-bar/p/11170094.html)

# Linux 黑话解释：什么是 Linux 中的 GUI、CLI 和 TUI？ | Linux 中国
https://zhuanlan.zhihu.com/p/282776001
```