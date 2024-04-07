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

# editcap工具
## 过滤pcap
通过 editcap， 我们能以很多不同的规则来过滤 pcap 文件中的内容，并且将过滤结果保存到新文件中。
### 时间范围过滤
以“起止时间”来过滤 pcap 文件。 " - A < start-time > 和 " - B < end-time > 选项可以过滤出在这个时间段到达的数据包（如，从 2:30 ～ 2:35）。时间的格式为 “ YYYY-MM-DD HH:MM:SS"。
A: after;  B: before；

```c
editcap -A '2014-12-10 10:11:01' -B '2014-12-10 10:21:01' input.pcap output.pcap

editcap -B "2021-08-15 21:35:00" test.pcapng test1.pcapng
从 test.pcapng 中读取指定时间之前的数据包，然后保存为 test1.pcapng

editcap -A "2021-08-15 21:35:00" test.pcapng test1.pcapng
从 test.pcapng 中读取指定时间之后的数据包，然后保存为 test1.pcapng
```

### 指定范围的N个包
可以从某个文件中提取指定范围的 N 个包。下面的命令行从 input.pcap 文件中提取100个包（从 401 到 500）并将它们保存到 output.pcap 中：
```c
editcap input.pcap output.pcap 401-500

$ editcap -r test.pcapng test1.pcapng 1-10
保留 test.pcapng 中 1#-10# 的数据包，然后保存为 test1.pcapng


$ editcap -r test.pcapng test1.pcapng 10
保留 test.pcapng 中 10# 的数据包，然后保存为 test1.pcapng
```


### 剔除重复数据包
#### 指定包个数范围内的重复包
使用 "-D < dup-window >" （dup-window可以看成是对比的窗口大小，仅与此范围内的包进行对比）选项可以剔除重复包。每个包都依次与它之前的 < dup-window > -1 个包对比长度与MD5值，如果有匹配的则丢弃。

```c
editcap -D 2000 input.pcap output.pcap

比如：输出结果为：
3516968 packets seen, 126672 packets skipped with duplicate window of 2000 packets.
遍历了 3516968 个包, 在 2000 窗口内重复的包有 126672 个，并丢弃。
```
#### 指定时间范围内的重复包
```c
editcap -w 0.5 input.pcap output.pcap
```
### 查看重复包


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

## 截断数据包
```c
editcap -s 60 test.pcapng test1.pcapng
按 60 字节长度截断数据包。
```


# pcap-diff工具
## 安装

```bash
python脚本的pcap-diff工具：
https://github.com/isginf/pcap-diff
```
##  `pcap-diff` 比较包异同
`pcap-diff` 用于比较 `pcap` 文件的异同。

### 利用pcap-diff 和  editpcap 得到重传的数据包

# 参考
```c
https://linux.cn/article-4762-1.html
```