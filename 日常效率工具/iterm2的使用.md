```table-of-contents
```
# ssh 会话保持
## 背景
最近使用通过relay登录跳板机才能登录使用线上各个服务器,这样的话导致每次新打开一个iterm窗口，都需要输入密码才可以登录跳板机。

## 设置
如下所示，设置ssh，进行了会话保持，这样只要第一次登录，后续进行会话保持，后续打开窗口进行登录，就不需要输入密码。 


```bash
# cat ~/.ssh/config
Host *
    PubkeyAcceptedKeyTypes +ssh-rsa
    ServerAliveInterval 60
    ControlMaster auto
    ControlPath ~/.ssh/%r@%h:%p
```
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



# 设置光标按照单词快速移动
## 背景
iTerm2之后，发现`option+←`和`option+→`这两组快捷键并不能实现光标按照单词快速移动。

## 设置
打开 `Preferences`，
![[attachments/Pasted image 20250207181845.png]]

进入 `Profiles -> Keys -> Key Mappings`。选择 `Natural Text Editing` 或者 `Terminal.app Compatiability`.  `Terminal.app Compatiability` 应该更好点。
![[attachments/Pasted image 20250207181915.png]]

![[attachments/Pasted image 20250207185721.png]]

# 即时回放

使用`Command + Opt + b` 打开即时回放，按`Esc`退出。即时回放可以记录终端输出的状态，让你“穿越时间”查看终端内容。默认每个会话最多储存4MB的内容，可以在设置中更改（`Preferences -> Genernal -> Instant Replay`）。

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


# 参考
```bash
# Iterm2 修改默认Key Mappings
https://juejin.cn/post/7090094297436913677

# Mac下Iterm2使用及快捷键
https://yu66.vip/doc/mac/005-Mac%E4%B8%8BIterm2%E4%BD%BF%E7%94%A8%E5%8F%8A%E5%BF%AB%E6%8D%B7%E9%94%AE.html
```