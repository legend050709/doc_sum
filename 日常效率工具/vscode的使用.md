```table-of-contents
```

# vscode 三方插件
如下所示，日常主要是C语言的开发，安装了下面的插件。
![[attachments/Pasted image 20250207184806.png]]

# vscode的配置
## settings.json的方式进行配置vscode
vscode菜单： `Code -> Preferences -> Settings`(快捷键command + ，)，如下方式，打开配置文件`settings.json`。

![[attachments/Pasted image 20250207183058.png]]

### 编辑代码后自动保存

![[attachments/Pasted image 20250207183546.png]]

### 代码保存后自动格式化
在`settings.json`文件里面添加如下：

```json
// "editor.formatOnType": true,
// 写这一个就可以
"editor.formatOnSave": true
```

# vscode阅读C
## 插件安装
### `C/C++ Extension Pack`插件
安装完了`C/C++ Extension Pack`之后，可以实现跳转。
`command + 鼠标左键`，实现跳转到函数定义的地方或者函数调用的地方。
`ctrl + -`实现回退到跳转前的位置。

# vscode阅读golang
## mac上安装go开发包
1. 打开浏览器,访问`Go`语言的官方网站:[https://golang.org/dl/](https://golang.org/dl/)
2. 下载适用于`macOS`的安装包（一般是以.pkg结尾的文件）。
3. 双击下载的`.pkg`文件,按照提示进行安装。
4. 安装完成后,在终端中运行以下命令来验证Go是否安装成功:
   `go version`
   如果输出类似以下信息,说明Go语言已成功安装:
   `go version go1.19 darwin/amd64`
5. 另外,您可以通过以下命令设置Go的工作目录:
```bash
   mkdir ~/go
   echo 'export GOPATH=$HOME/go' >> ~/.bash_profile
   echo 'export PATH="/usr/local/go/bin:$PATH" ' >> ~/.bash_profile
   echo 'export GOROOT=/usr/local/go ' >> ~/.bash_profile
   source ~/.bash_profile
```

# 参考
```bash
# Mac下vscode编辑器设置
https://yu66.vip/doc/mac/007-Mac%E4%B8%8Bvscode%E7%BC%96%E8%BE%91%E5%99%A8%E8%AE%BE%E7%BD%AE.html

```

