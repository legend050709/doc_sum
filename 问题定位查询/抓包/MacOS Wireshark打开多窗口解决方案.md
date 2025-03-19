```table-of-contents
```
# 背景
wireshark不能打开多个窗口，看多个pcap文件的时候，打开新的，会自动把旧的替换掉。
使用mac的 自动操作，可以打开多个窗口。

# 方法
## 方法一：通过命令行打开
```
# 打开多个初始Wireshark窗口
open -n /Applications/Wireshark.app

# 直接在多个窗口打开抓包文件
open -n -a /Applications/Wireshark.app file_name.pcap
```

## 借助自动操作打开
### 创建自动操作

![](attachments/Pasted%20image%2020250212123139.png)
点击「自动操作」的图标

![](attachments/Pasted%20image%2020250212123150.png)
点击「新建文稿」按钮，弹出「选择文稿类型」界面

![](attachments/Pasted%20image%2020250212123206.png)

![](attachments/Pasted%20image%2020250212123217.png)

![](attachments/Pasted%20image%2020250212123227.png)
文稿类型选取「应用程序」

![](attachments/Pasted%20image%2020250212123246.png)
![](attachments/Pasted%20image%2020250212123251.png)

 向下滑动找到「运行脚 Shell 脚本」并拖到右侧

![](attachments/Pasted%20image%2020250212123311.png)

更改 `传递输入`为 `作为自变量`; 脚本内容更改为
```text
open -n -a /Applications/Wireshark.app $1
```
修改后的内容如上图所示。

### 保存自动操作到应用程序

选择「文件」-> 「存储」
![](attachments/Pasted%20image%2020250212123411.png)

存储为填一个你容易找的名称，我这里面填的是 `WiresharkX` 位置选择「应用程序」 文件格式选择「应用程序」

![](attachments/Pasted%20image%2020250212123442.png)

保存之后在 `mac` 的启动台和应用程序里面就可以看到刚刚创建的 `WiresharkX`。

### Wireshark 打开多窗口测试

在应用程序或者启动台中找到 WiresharkX 图标多次点击，即可打开多个 Wireshark 窗口，这样就可以进行抓包进行对比分析了。

![](attachments/Pasted%20image%2020250212123536.png)

# 参考
```c
https://www.cnblogs.com/wangjq19920210/p/17058626.html

# 参考：【小技巧】MacOS Wireshark 打开多个窗口
https://zhuanlan.zhihu.com/p/676586313
```