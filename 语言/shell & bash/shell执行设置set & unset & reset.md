# set 
## 作用一：显示系统中已经存在的shell变量、环境变量、shell函数

- set 显示本地定义的shell 函数
![](attachments/Pasted%20image%2020230627145547.png)

如下所示：set 命令的开头会显示环境变量，后面显示的是本地定义的shell变量, 以及函数。
![](attachments/Pasted%20image%2020230627144614.png)
```c
# man -f set
set (n)              - Read and write variables
set (1p)             - set or unset options and positional parameters
```
![](attachments/Pasted%20image%2020230627152016.png)


## 作用二：用来设置shell脚本的执行方式
使用set更改shell特性时，符号"+"和"-"的作用分别是关闭和打开指定的模式。

### 参数
```c
- -e 　若指令返回值不等于0（即发生异常），则立即退出shell。
- -n 　只读取指令，而不实际执行。可用于检查shell脚本中语法错误
- -u 　当执行时使用到未定义过的变量，则显示错误信息。
- -v 　显示shell所读取的输入值。
- -x 　执行指令后，会先显示该指令及所下的参数。
- +<参数> 　取消某个set曾启动的参数。
```


### 范例
- set -e
`-e`参数表示只要shell脚本中发生错误，即命令返回值不等于0，则停止执行并退出shell。`set -e`在shell脚本中经常使用。默认情况下，shell脚本碰到错误会报错，但会继续执行后面的命令。
注：`set +e`表示关闭-e选项，`set -e`表示重新打开-e选项。

- set -u
`-u`参数表示shell脚本执行时如果遇到不存在的变量会报错并停止执行。默认不加`-u`参数的情况下，shell脚本遇到不存在的变量不会报错，会继续执行。


## 注意
set命令不能够定义新的shell变量。如果要定义新的变量，可以使用declare命令以`变量名=值`的格式进行定义即可。

```bash
使用declare命令定义一个新的环境变量"mylove"，并且将其值设置为"Visual C++"，输入如下命令：
declare mylove='Visual C++'   #定义新环境变量

再使用set命令将新定义的变量输出为环境变量，输入如下命令：
set -a mylove                 #设置为环境变量

执行该命令后，将会新添加对应的环境变量。用户可以使用env命令和grep命令分别显示和搜索环境变量"mylove"，输入命令如下：

env | grep mylove             #显示环境变量值

此时，该命令执行后，将输出查询到的环境变量值。
```
# unset
```c
# man -f unset
unset (n)            - Delete variables
unset (1p)           - unset values and attributes of variables and functions
```
![](attachments/Pasted%20image%2020230627150551.png)

# reset
reset命令可以进行终端初始化。reset命令可以进行终端初始化。
比如，不小心把二进位档用 cat 指令进到终端机，常会有终端机不再回应键盘输入，或是回应一些奇怪字元的问题。此时就可以用 reset 将终端机回复至原始状态。
