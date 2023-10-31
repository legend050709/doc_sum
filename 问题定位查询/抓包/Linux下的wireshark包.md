```table-of-contents
```
# 介绍
Wireshark包实际上它带了一套非常有用的命令行工具集。其中包括 editcap 与 mergecap。editcap 是一个万能的 pcap 编辑器，它可以过滤并且能以多种方式来分割 pcap 文件。
mergecap 可以将多个 pcap 文件合并为一个。
# 使用方法
![](attachments/Pasted%20image%2020231027161820.png)
![](attachments/Pasted%20image%2020231027160629.png)
# 处理
## 安装
```c
yum install wireshark -y
```
## 过滤pcap
通过 editcap， 我们能以很多不同的规则来过滤 pcap 文件中的内容，并且将过滤结果保存到新文件中。
### 时间范围过滤
以“起止时间”来过滤 pcap 文件。 " - A < start-time > 和 " - B < end-time > 选项可以过滤出在这个时间段到达的数据包（如，从 2:30 ～ 2:35）。时间的格式为 “ YYYY-MM-DD HH:MM:SS"。
```c
editcap -A '2014-12-10 10:11:01' -B '2014-12-10 10:21:01' input.pcap output.pcap
```
### 指定范围的N个包
可以从某个文件中提取指定范围的 N 个包。下面的命令行从 input.pcap 文件中提取100个包（从 401 到 500）并将它们保存到 output.pcap 中：
```c
editcap input.pcap output.pcap 401-500
```
### 剔除重复数据包
#### 指定包个数范围内的重复包
```c
editcap -D 10 input.pcap output.pcap
```
#### 指定时间范围内的重复包
```c
editcap -w 0.5 input.pcap output.pcap
```
## 分割pcap
### 按照包个数分割
将一个 pcap 文件分割成数据包数目相同的多个文件。输出的每个文件有相同的包数量，以 < output-prefix >-NNNN的形式命名。
```c
editcap -c <packets-per-file> <input-pcap-file> <output-prefix>
```
### 以时间分割
```c
editcap -i <seconds-per-file> <input-pcap-file> <output-prefix>
```
## 合并pcap
如果想要将多个文件合并成一个，用 mergecap 就很方便。
当合并多个文件时，mergecap 默认将内部的数据包以时间先后来排序。
```c
mergecap -w output.pcap input.pcap input2.pcap [input3.pcap . . .]
```

如果要忽略时间戳，仅仅想以命令行中的顺序来合并文件，那么使用 -a 选项即可。
例如，下列命令会将 input.pcap 文件的内容写入到 output.pcap, 并且将 input2.pcap 的内容追加在后面。
```c
mergecap -a -w output.pcap input.pcap input2.pcap
```
##  比较包异同
pcap-diff 用于比较 pcap 文件的异同。
# 参考
```c
https://linux.cn/article-4762-1.html
```