
# 常用Ctrl快捷键
## ctrl +c && ctrl + z
首先，最常用的是Ctrl+C和Ctrl+Z：
Ctrl+C是***向前台程序发送SIGINT信号***，相当于kill -2 【进程】，所以有时候进程卡死了用Ctrl+C是无法的退出的。
这时候可以使用Ctrl+Z切出来，Ctrl+Z的作用是将***程序挂起在后台***。
举两个例子：
![](attachments/Pasted%20image%2020230508124246.png)

![](attachments/Pasted%20image%2020230508124334.png)

## jobs、fg、bg命令

```c
#查看终端后台程序 jobs 
#查看终端后台程序及程序进程号，经常配合kill命令使用 jobs -l 
#恢复后台挂起程序并保持后台运行，不加参数默认选择序号为1的后台程序 bg 
#恢复指定序号的挂起程序并保持后台运行，序号通过jobs查询，比如 bg 1 
#恢复后台程序并转到前台运行，不加参数默认选择序号1的后台程序 fg 
#恢复指定序号的后台程序并转到前天运行，需要通过jobs查询，比如 fg 1
```

# 拓展
Ctrl+r ：在命令历史中搜索命令（非常好用，相当于history命令的快速检索）（记忆方法：reverse）

Ctrl+l ：清理终端输出（非常好用，等效于clear命令但是更快捷）

Ctrl+s ：暂停终端输出(_**终端的锁定屏幕的快捷键**_)（记忆方法：screenlock或者stop）
>注：不管在不在Vi/Vim中，只要在终端中按下Ctrl+S都会产生“死机”的假象，所以以后碰上终端卡死的情况，不妨试试看Ctrl+Q，兴许是终端被锁定导致的。其实Ctrl+S/Ctrl+Q的作用和键盘上万年吃灰的ScrLK（ScreenLock）键是一样的，Ctrl+S锁定终端后，~~直接按下ScreenLock也可以实现解锁~~(勘误：和系统环境相关，实测在Ubuntu下可以，在Deepin下无效)，所以可以这么记忆：

Ctrl+q ：恢复终端输出 （配合Ctrl+s使用）（记忆方法：quit screenlock）

Ctrl+d : 发送EOF指令，相当于在终端中输入exit后回车，用于退出终端 （记忆方法：done）

Ctrl+a ：输入命令时将光标转跳到命令的最前面，相当于按键盘上的home键 （记忆方法：a是字母表第一个）

Ctrl+e ：与Ctrl+a相对应，将光标转跳到最后面，相当于按键盘上的end键 （记忆方法： end）

Ctrl+w ：直接删除一个单词（已空格为界限）（记忆方法：word）

Ctrl+u ：输入命令时可以一键删除光标前的输入内容

Ctrl+k ：类似Ctrl+u，输入命令时一键删除光标后的所有输入内容

更多快捷键可以通过下面的指令查询：
![](attachments/Pasted%20image%2020230508124920.png)


# 参考
```c
https://www.gooneyryan.com/archives/983
```