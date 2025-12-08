```table-of-contents
```
# oh-my-zsh设置
## 安装 oh-my-zsh
```bash
sh -c "$(curl -fsSL https://raw.github.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```
看到如下提示说明安装成功了
![[attachments/Pasted image 20250207212303.png]]

## 设置 Zsh 为默认 Shell
在终端中运行以下命令来将 Zsh 设置为默认 Shell：
```bash
chsh -s /bin/zsh
```
**配置 Oh-My-Zsh**： Oh-My-Zsh 默认安装后会生成一个配置文件 .zshrc，你可以编辑这个文件来定制你的 Zsh 配置和主题。可以通过编辑 ~/.zshrc 文件来添加插件、更改主题等。 **记得更改之后，要执行`source ~/.zshrc` 使配置生效。**


## 插件
### 自动补全(zsh-autosuggestions)
安装 zsh 命令提示补全插件以后的样子，输几个字母就能出现自己想要的（之前自己输入过的命令），直接一个右向箭头，就直接补全了完整命令，好不快捷。

![[attachments/Pasted image 20250207212153.png]]

#### 安装 zsh-autosuggestions 插件
使用包管理工具 git 克隆 `zsh-autosuggestions` 仓库到 Zsh 插件目录（通常是 `~/.oh-my-zsh/custom/plugins/`）：
```bash
git clone https://github.com/zsh-users/zsh-autosuggestions ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions

注：
	~/.oh-my-zsh/plugins 下 是自带的内置插件。
	~/.oh-my-zsh/custom/plugins 是自己安装的三方插件，当然也可以放入到 ~/.oh-my-zsh/plugins 下。
```

#### 激活插件
编辑 `~/.zshrc` 文件，在插件部分添加 `zsh-autosuggestions`：
```bash
plugins=(zsh-autosuggestions) 
// 或者 把 
plugins=(git) 改成 
plugins=( git zsh-autosuggestions )
```
![[attachments/Pasted image 20250207212608.png]]

然后在 `~/.zshrc` 文件 尾部中添加，如下所示：
> 注：新手可能会发现自动建议的颜色与终端背景颜色过于接近，导致建议不明显。可以通过`ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE`设置。

```bash
source ~/.oh-my-zsh/custom/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
ZSH_AUTOSUGGEST_HIGHLIGHT_STYLE="fg=#ff00ff,bg=cyan,bold,underline"
```

#### 使更改生效
```bash
source ~/.zshrc
```

到这里 zsh 命令提示补全就安装完成了，可以享用命令提示补全工具带来的便利了，去试一试吧。
![[attachments/Pasted image 20250207212643.png]]

### 语法高亮()

### 内置插件
#### copyfile
copyfile，是一个内置的插件，无需额外安装，用于拷贝文件内容到剪贴板，命令格式 `copyfile <文件路径>`。
假设，现有一个文件 test.txt。
```bash
# cat test.txt 
Hello oh my zsh
```

一个测试命令，`copyfile test.txt`，即可将 `test.txt` 文件中的内容拷贝到剪贴板中。无需鼠标选中复制粘贴。

![[attachments/Pasted image 20250208104451.png]]

#### z或者zoxide
**z 或者 zoxide**：基于历史访问目录的快速跳转；我们无需输入全路径，即可完成目录切换。
我觉得大部分人在使用 Linux 都被 cd 跳转目录跳转烦恼过。z 就是这个烦恼的救星。

首先，我直接输入 z，紧跟 tab 键，会看到如下的效果。它会直接将访问过的目录都列出来。
![[attachments/Pasted image 20250208104915.png]]

这些由 tab 产生的自动补全目录都是历史访问过的目录。因为，在没有输入任何内容的情况下，我们输入 tab 的，它列出最近访问过的目录。

如果我们输入形如 `z substring`，即提供子字符串，它们将所有匹配 `substring` 的目录都列举出来。
例如，我们输入 `z blog`，紧跟 tab 键，会直接列出访问过包含 `blog` 的目录。
如果输入内容只有一个关联的目录名，我们输入 `z tmux`，因为匹配 `tmux` 的目录只有一个，将会被直接选中。我们输入 `z tmux`，直接 `Enter` 确认，即可进入到目录。


##### 生效
打开 `.zshrc` 完成配置：
```bash
plugins=(git  z copyfile zsh-syntax-highlighting zsh-autosuggestions)
```
执行 `source ~/.zshrc` 生效配置。

# ssh 连接
## ssh会话复用
### 背景
现在很多公司登录服务器都需要先登录到一个跳板机(relay)然后再登录到目标机器，每次输入密码（一般还是动态的）很麻烦。这样的话导致每次新打开一个iterm窗口，都需要输入密码才可以登录跳板机。
一般的教程推荐使用 expext 解决这个问题，这里介绍一个更简单、直接的办法。


### 设置

#### 连接复用

首先，要先解决登录到跳板机的连接复用问题，这个输入 ssh 本身的范畴，一般教程都是说在 ~/.ssh/config 增加下面的配置：
```bash
Host *
	ControlMaster auto
	ControlPath ~/.ssh/master-%r@%h:%p
```

但这样有个问题，iTerm2 第一次登录的 tab 关闭后，就失效了，再登录就又要密码了。其实再增加一个配置即可。
```bash
ControlPersist yes
ServerAliveInterval 60
```
之前的设置只是实现了连接复用，但 tab 关闭 ssh 进程结束后，连接也被销毁了，`ControlPersist` 项的意思是进程结束后连接还保持，再有相同的（ControlPath配置）连接，还能继续使用此连接。

#### 小结
小结，如下所示：
```bash
# cat ~/.ssh/config
Host *
    PubkeyAcceptedKeyTypes +ssh-rsa
    ControlPersist 24h
    ServerAliveInterval 60
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
```

```bash
说明：
	`Host *`: `*` 表示所有的机器。
	`ServerAliveInterval 60` : 是每60S发送一个no-op包，这样服务器就不会关闭.
```

## 连接空闲一段时间断开的问题
### 背景
有时候会发现，有些机器一会儿没操作就会被断开 ssh。


### 分析

一方面可以设置 `ServerAliveInterval`，但作用不大，因为实际连接服务器的是 relay 跳板机。

iTerm2 其实没有可以显式设置这个功能的地方，有些同学可能找到个这个配置。

![[attachments/Pasted image 20250209175250.png]]

可能错误使用的比较多，iTerm2 还特意提示了，开启这个功能可能存在其他的影响，比如view打开一个文件， 无缘无故添加了ASCII码。

那到底有没有办法呢，还是有的。其实这个特性是为了安全，是 `TMOUT` 环境变量控制的，`export` 设置一个比较大的值（比如12h，保证第二天来了不断就行）。这个可以设置在自己常用机器的环境变量里，或者定义一个 `Snippet` 方便使用（后面还有介绍）。

```bash
在 ~/.bash_profile 文件中，添加：
export TMOUT=86400
```

### 配置
```bash
#  cat ~/.ssh/config
Host *
	PubkeyAcceptedKeyTypes +ssh-rsa
    ControlMaster auto
    ControlPath ~/.ssh/master-%r@%h:%p
    ControlPersist 12h
	SendEnv LANG LC_*
	ServerAliveCountMax 10
	ServerAliveInterval 10
	IPQoS=throughput
```

### 总结
如果是连接到 `relay`上， 再通过`relay`连接服务器，那么 设置 如下，应该也会给真实的服务器发送这样的`ASCII`码，可能也是有问题。

![[attachments/Pasted image 20250209175250.png]]

如果不是直接连接`relay`，而是直接连接的是服务器，那么其实是不建议使用上面的方式。而是通过 编辑`~/.ssh/config`，以及设置 `export TMOUT=86400` 来看看是否存在效果「经过测试，看着效果也不好」。


# 安装 powerlevel10k zsh 主题
powerlevel10k 是一个高度可定制的 Zsh 终端主题，它提供了丰富的功能和配置选项，以增强你的命令行体验。以下是 powerlevel10k 的一些主要特性：

**美观的界面**：powerlevel10k 拥有一个现代且美观的界面，支持多种颜色和字体样式。
**即时反馈**：它提供了即时的命令执行反馈，包括命令执行时间、退出状态等。
**Git 集成**：powerlevel10k 深度集成了 Git，可以显示当前分支、状态、上次提交信息等。
**自定义配置**：用户可以根据自己的喜好和需求，通过配置文件来定制主题的外观和功能。
**插件系统**：powerlevel10k 拥有一个插件系统，允许用户添加或移除特定的功能模块。
等等…


安装之后的效果如下所示：
![](attachments/Pasted%20image%2020250213194928.png)

## 安装 Meslo Nerd Font 字体
作者推荐使用 Meslo Nerd Font 这个字体，Nerd Font 相信大家都不陌生了，就是图标字体]，可以在终端上展示各种好看的图标。其实我们之前已经安装过 Nerd Font 类型的字体了，如果你不想安装可以跳过。

官网说了如果是 **iterm2 可以在安装 p10k 的时候进行安装**，没有必要手动去下载那一堆字体文件进行安装。

![](attachments/Pasted%20image%2020250213195048.png)

所以我们待会安装即可。

## 安装 p10k 主题
```bash
cd ~/.oh-my-zsh/themes
git clone https://github.com/romkatv/powerlevel10k.git
```

打开 zsh 的配置文件 `vim ~/.zshrc`将主题修改为 `p10k`
```bash
ZSH_THEME="powerlevel10k/powerlevel10k"
```

然后记得刷新配置 `source ~/.zshrc`，或者重启终端也行。

## 逐帧配置 p10k

执行 `source ~/.zshrc`之后会进行 p10k 的配置引导程序里。  

### 安装字体
第一个页面会提示你是否安装 Meslo Nerd Font 字体，输入 y 即可，安装成功就是下面这样。

![](attachments/Pasted%20image%2020250213195344.png)

然后 Command + Q 退出 iterm2，重新打开。  
关闭之后我们记得配置一下字体，选择带有 Nerd Font 后缀的字体。自行检查一下字体是否设置成功。

![](attachments/Pasted%20image%2020250213195456.png)

（1）可以看到图标的话，说明你的字体安装成功了，支持展示图标！

![](attachments/Pasted%20image%2020250213195512.png)

（2）一把锁

![](attachments/Pasted%20image%2020250213195527.png)


（3）向上的箭头

![](attachments/Pasted%20image%2020250213195545.png)

（4）交叉但不重叠

![](attachments/Pasted%20image%2020250213195604.png)


### 其他配置

（5）选择终端提示符，想要炫酷的效果就选 2（对应图上的 3），我这里就选 1（对应图上的 2）了。

![](attachments/Pasted%20image%2020250213195644.png)

（6）Unicode 还是 ASCII，我这里选 1 了。

![](attachments/Pasted%20image%2020250213195702.png)

（7）选择终端提示符的主题，我这里选 3 了。

![](attachments/Pasted%20image%2020250213195721.png)

（8）展示时间，选 2

![](attachments/Pasted%20image%2020250213195739.png)

（9）分隔符，选 2

![](attachments/Pasted%20image%2020250213195756.png)

（10）提示头部，选圆角 5 舒服一点。

![](attachments/Pasted%20image%2020250213195833.png)

（11） 提示尾部，圆角 5

![](attachments/Pasted%20image%2020250213195850.png)

（12）命令在一行输入还是两行输入，两行 2。

![](attachments/Pasted%20image%2020250213195901.png)

（13）头尾连接符，虚线 2

![](attachments/Pasted%20image%2020250213195929.png)

（14）提示框架，全连 4 就是左边也连着，右边也连着。

![](attachments/Pasted%20image%2020250213195948.png)

（15）两个提示符之间的间距，2 别那么挤。

![](attachments/Pasted%20image%2020250213200002.png)


（16）**选择图标，一定要 2 呀，要不然我们安装那么多字体干什么**

![](attachments/Pasted%20image%2020250213200028.png)

（17）1 信息展示精简一点

![](attachments/Pasted%20image%2020250213200041.png)

（18）y，完成的命令就归档在一起，很好的设计。

![](attachments/Pasted%20image%2020250213200108.png)

（19）命令行模式选择 1，给你详细的提示。

![](attachments/Pasted%20image%2020250213200137.png)

（20）y，把我们的这些配置写到配置文件中。
可以发现 `~/.zshrc`文件的最后多了一行`p10k`的配置。

![](attachments/Pasted%20image%2020250213200153.png)


# iterm2设置
## 自定义快捷键
iTerm2 本身也提供了很强大的 Profiles 功能，能一键登录目标机器（还可以设置快捷键）。

![[attachments/Pasted image 20250210103646.png]]

有些自己的常用机器，还可以复制一下这份基础的 relay 配置。
![[attachments/Pasted image 20250210103732.png]]

再加一行（Send text at start处）。
![[attachments/Pasted image 20250210103743.png]]

还可以把机器打个标签，然后页面上就能分组，方便选择。
![[attachments/Pasted image 20250210103804.png]]


## 显示操作时间轴
`View -> Show Timestamps`，就能再右边显示一个操作的时间轴，这样就能很方便的看见之前的命令是什么时候执行的，有些时候排查问题特别方便。但这个会遮挡右边的显示，我们可以定义一个 action，方便切换开关。

![[attachments/Pasted image 20250209175538.png]]


## 关闭iterm2中对于行数的限制
iTerm2 默认的行数限制，超过 1000 的部分就被隐藏不显示了。
![](attachments/Pasted%20image%2020250211211242.png)

## 设置光标按照单词快速移动
### 背景
iTerm2之后，发现`option+←`和`option+→`这两组快捷键并不能实现光标按照单词快速移动。

### 设置
打开 `Preferences`，
![[attachments/Pasted image 20250207181845.png]]

进入 `Profiles -> Keys -> Key Mappings`。选择 `Natural Text Editing` 或者 `Terminal.app Compatiability`.  `Terminal.app Compatiability` 应该更好点。
![[attachments/Pasted image 20250207181915.png]]

![[attachments/Pasted image 20250207185721.png]]

## 即时回放

使用`Command + Opt + b` 打开即时回放，按`Esc`退出。即时回放可以记录终端输出的状态，让你“穿越时间”查看终端内容。默认每个会话最多储存4MB的内容，可以在设置中更改（`Preferences -> Genernal -> Instant Replay`）。


## 单击选中命令块

可以在之前输入输出构成的命令块中搜索和过滤关键字，查看结果更加清楚。

![](attachments/Pasted%20image%2020250213202317.png)

如何关闭？  `General -> Selection`，把默认勾选去掉即可。

![](attachments/Pasted%20image%2020250213202336.png)


## 终端提示符标记

我们可以清楚的看到，每一行前面有一个**蓝色小标记**。

![](attachments/Pasted%20image%2020250213202416.png)

这是 iterm2 终端为 Shell自带的一些集成特性优化，但是和我们 zsh 主题的标记冲突了，所以需要把它关掉。`Profiles -> Terminal`把默认勾选去掉即可。

![](attachments/Pasted%20image%2020250213202440.png)



# 光标控制

```bash
ctrl + a: 到行首
ctrl + e: 行末
ctrl + f/b: 前进后退，相当于左右方向键，但是显然比移开手按方向键更快
ctrl + p: 上一条命令，相当于方向键上
ctrl + r: 搜索命令历史，这个大家都应该很熟悉了
ctrl + d: 删除当前字符
ctrl + h: 删除之前的字符
ctrl + w: 删除光标前的单词
ctrl + k: 删除到文本末尾
ctrl + t: 交换光标处文本
⌘ + —/+/0: 调整字体大小
⌘ + r:清屏，其实是滚到新的一屏，并没有清空。ctrl + l 也可以做到。
```


# 窗口操作
## 一个窗口分割

垂直分割: `Command + D`

- 水平分割: `Shift + Command + D`
- 前一个面板: `Command + [ 或 Option + Command + 左右方向键`
- 后一个面板: `Command + ]`
- 切换到上/下/左/右面板: `Option + Command + 上下左右方向键`
- 关闭panel：`⌘ + w`
- 最大化Tab中的pane，隐藏本Tab中的其他pane：`⌘+ shift +enter` , 再次还原

##  新建Tab标签页

- 新建标签页: `Command + T`
- 关闭标签页: `Command + W`
- 前一个标签页: `Command + 左方向键，Shift + Command + [`
- 后一个标签页: `Command + 右方向键，Shitf + Command + ]`
- 进入标签页1，2，3…: `Command + 标签页编号`
- Expose 标签页: `Option + Command + E`（将标签页打撒到全屏，并可以全局搜索所有的标签页）

窗口太多，可以使用 ⌘ + / 快速定位到光标所在位置

## 多个窗口操作

- 新建窗口：`command + N`
- 关闭窗口： `command + w`
- 前一个窗口：command + `
- 后一个窗口：Shitf + command + `
- 进入窗口 1,2,3：option + command + 编号


# zsh别名
打开 `.zshrc`配置文件，可以将常用命令配置一个别名，下次只要输入别名就可以执行对应的命令，可以大大提高我们的工作效率。

```bash
alias c=clear
alias e=exit
alias i=idea
alias v=vim
alias vim=nvim
alias t=touch
alias m=mkdir
alias gc="git clone"
```

# 问题记录
## 窗口中ctrl+C等命令失效，显示对应的文本

![](attachments/Pasted%20image%2020251107142352.png)

# 参考
```bash
# Iterm2 修改默认Key Mappings
https://juejin.cn/post/7090094297436913677

# Mac下Iterm2使用及快捷键
https://yu66.vip/doc/mac/005-Mac%E4%B8%8BIterm2%E4%BD%BF%E7%94%A8%E5%8F%8A%E5%BF%AB%E6%8D%B7%E9%94%AE.html

# 【Mac 从 0 到 1 保姆级配置教程 05】 - 全网最详细 20+ 张图逐帧安装 powerlevel10k zsh 主题
https://blog.csdn.net/Xiang__Qian/article/details/139871268
```