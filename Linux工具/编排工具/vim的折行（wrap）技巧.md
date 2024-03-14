```table-of-contents
```
# 背景
在默认情况下，Vim会自动折行––将超出屏幕范围的文本打断并显示在下一行。这样会显得输出很杂乱。如下所示：
![](attachments/Pasted%20image%2020230810133624.png)
# 方法
更加倾向于方法2，因为如果文件很大，则vim打开一个大文件，需要将文件内容加载到内存，占用大量内存，可能导致机器的其他进程的内存申请失败。

## 方法一: vim的 set nowrap不换行显示
```c
1> vim <(cat /proc/interrupts)
注：<( 中间没有空格
2> :set nowrap
设置不换行
3> :N
定位到第N行
4> 0 or $
定位到当前行的行首和行尾
```

说明：
```text
"vim <(cat /proc/interrupts)"这个命令是将"/proc/interrupts"文件的内容通过管道传递给"cat"命令，再将"cat"命令的输出结果通过重定向符"<"传递给"vim"命令。这样，"vim"命令就可以打开一个临时文件，该文件的内容就是"/proc/interrupts"文件的内容。

这种方式的作用是可以在不改变"/proc/interrupts"文件本身的情况下，使用"vim"等编辑器对其进行查看和编辑。同时，由于是在内存中创建临时文件，因此也不会对磁盘空间产生影响。

除了上述方式之外，还有其他一些实现方法。例如，您可以使用以下命令将"/proc/interrupts"文件的内容保存到一个临时文件中：
cat /proc/interrupts > /tmp/interrupts
然后，您可以使用"vim"等编辑器打开该临时文件进行查看和编辑。需要注意的是，使用这种方式会在磁盘上创建一个临时文件，因此需要确保磁盘空间充足; 同时vim打开文件时，需会加载文件到内存。
```

![](attachments/Pasted%20image%2020230810133703.png)
## 方法二: less -S 不换行显示
cat输出时，一行往往过长而换行显示，就会显得很乱。
加上“**-S**”参数可以强制不换行显示，及屏幕的一行只显示文件的一行，当屏幕不够时可以按左右键来左右移动显示内容。
```c
-S or --chop-long-lines:不换行显示
        Causes lines longer than the screen width to be chopped (truncated) rather than wrapped.  That is, the portion of a long line that does not fit in the screen width is not shown.  The default is to wrap long lines; that is, display the remainder on the next line.


-N or --LINE-NUMBERS：显示行号
    Causes a line number to be displayed at the beginning of each line in the display.

-n or --line-numbers:抑制行号
    Suppresses line numbers.  The default (to use line numbers) may cause less to run more slowly in some cases, especially with a very large input file.  Suppressing line numbers with the -n option will avoid this problem.  Using line numbers means: the line number will be displayed in  the  ver‐bose prompt and in the = command, and the v command will pass the current line number to the editor (see also the discussion of LESSEDIT in PROMPTS below)

-m or --long-prompt
    Causes less to prompt verbosely (like more), with the percent into the file.  By default, less prompts with a colon.
```

```c
cat /proc/interrupts | less -S
如果希望加上行号则：cat /proc/interrupts | less -S -Nm
加上“**-N**”参数可以显示每行的行号，“**-m**”可以显示类似more命令的百分比，方便查看文件时知道看到哪个位置了。
```