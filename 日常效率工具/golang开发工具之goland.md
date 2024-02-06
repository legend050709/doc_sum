```table-of-contents
```
# goland的下载与激活
## 方式一
参考：[mac goland2023.3.4 快速激活到2099年](https://www.modb.pro/db/638074)
### 流程
**激活包下载**：  

网盘地址（2023/5/29 更新）：[百度网盘激活包下载地址](https://pan.baidu.com/s/1y8rBgngeQ7hXieaoFAf7lw)          
提取码：4ltz
激活包下载完之后，请 **解压** 并保持文件目录结构不变如下！（**注意：****激活包的文件路径中最好不要包含中文和特殊字符串**）

**执行脚本**：
请确保 **goland** 软件处于关闭状态！这时候打开一个 **terminal（命令行控制端）**，并将工作目录切换到解压出来的脚本文件所在路径（此处请注意，破解文件夹的路径最好不要出现中文！！！）然后执行命令：**sh ./golandmacActive.sh** 得到如下提示即可！


**激活goland**: 
再次打开 **goland** ,新安装的用户会自动弹出注册页面，已经在使用中（临时激活码或其他方式进入**goland**）的找到顶部导航栏 **help -> register** 按钮打开注册页面，将破解包里面的 **激活码.txt** 复制进去即可（这里请各位大佬注意！下图中左下角已经登陆的朋友请点击 log out），然后点击 蓝色 **Activate** 按钮即可

### 其他
#### zip包的密码
![](attachments/Pasted%20image%2020240206111254.png)

## 方式二
goland 需要激活码才可以激活。下面是goland的激活。
参考:[bilibili: GoLand 激活破解激活码2023最新教程](https://www.bilibili.com/read/cv26005032/)
## 总结
方式一，测试好用。方式二，不好用。
# goland的配置
# 使用技巧
## 快捷键
![](attachments/Pasted%20image%2020231208114902.png)
![](attachments/Pasted%20image%2020231208114851.png)
![](attachments/Pasted%20image%2020231208114911.png)
![](attachments/Pasted%20image%2020231208114918.png)

## 同时打开多个窗口
![](attachments/Pasted%20image%2020231208115344.png)
![](attachments/Pasted%20image%2020231208115350.png)
## Search Everywhere（随处搜索）
`Search Everywhere`（随处搜索）是一项多工具操作，可以帮助您查找任何文字内容！ 源代码中的任何条目、数据库、操作、用户界面元素、插件、设置、Git 分支、提交、标记、消息等。
![](attachments/Pasted%20image%2020231208112127.png)
> 在mac中选中字段，然后执行 `Shift+Shift`即可。

要缩小搜索范围，请按 _⇥/Tab_ 在选项卡之间导航，或点击窗口工具栏上的筛选器图标并选择适当的选项。
![](attachments/Pasted%20image%2020231208112204.png)
您可以在 _Find Tool Window_（查找工具窗口）的一个单独的选项卡中打开当前搜索结果并运行另一个查询。 只需点击 _Search Everywhere_（随处搜索）右侧的 _Open in Find Window_（在查找窗口中打开）图标即可。

## 关闭git管理
**关闭git管理**  
打开Settings | Version Control | Directory Mappings 选中目标文件夹，点击  
-号即可

## Project 目录自动选择打开的文件
看源码，我们是需要知道当前打开的文件所处的目录的，这样对整个代码流程理解是有帮助的。Goland 默认是不在 Project 目录选择打开的文件的。

自动选择打开的文件：Project目录设置中勾上 Always Select Opened File
![](attachments/Pasted%20image%2020231208114657.png)

## 自动生成单元测试文件代码
自动生成单元测试文件代码：输入 ⌃ + Enter;︎ 选择 Test for function 或者 Tests for file 会自动生成 xxx_test.go 文件及测试代码

## 新增文件自动添加注释
新增文件自动添加注释：Editor → File and Code Templates → Go File


## 自动生成函数注释
自动生成函数注释：Perferances → Plugin 中搜索 Goanno 插件，安装完成之后，在函数上方使用快捷键 ⌃⌘/ 便能自动生成函数注释

## .idea 文件夹
.idea 文件夹是存储 IntelliJ IDEA 项目的配置信息，主要内容有 IntelliJ IDEA 项目本身的一些编译配置、数据源，类库，项目字符编码，历史记录，版本控制信息等。

goland打开项目后，会自动在本地生成一个.idea目录。但是这个 .idea 文件夹可能会影响git 的提交。

> 一般用 Git 做版本控制的时候会把.idea 文件夹排除，因为这个文件下保存的都是个人本地 idea 编译器的配置，这样可以有效避免版本冲突。


**解决**
.gitignore 文件创建并更新。在 gitignore 中忽略.idea 文件夹.
![](attachments/Pasted%20image%2020231208154051.png)
```c
### IntelliJ IDEA ###
.idea
*.iws
*.iml
*.ipr
```
**配置语法遵循如下：**
- 以斜杠`/`开头表示目录；
- 以星号`*`通配多个字符；
- 以问号`?`通配单个字符
- 以方括号`[]`包含单个字符的匹配列表；
- 以叹号`!`表示不忽略(跟踪)匹配到的文件或目录；
    
此外，`git` 对于`.gitignore`配置文件是按行从上到下进行规则匹配的，意味着如果前面的规则匹配的范围更大，则后面的规则将不会生效；
        

# 参考
```c
https://www.bilibili.com/read/cv26005032/
```