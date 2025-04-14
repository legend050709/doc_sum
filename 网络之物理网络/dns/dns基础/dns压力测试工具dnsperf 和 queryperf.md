```table-of-contents
```
# 概述
DNS的压力测试工具最常用（或者说最好找）的就是`dnsperf`和`queryperf`。其中`queryperf`是bind软件自带的，`dnsperf`软件是`Nominum`提供的一款开源压测软件。
# dnsperf
## 介绍
dnsperf 是一个 dns的性能测试工具，一般用来测试 dns 权威服务器的性能，也可以用来测试 dns缓存服务器的性能。
使用 dnsperf 测试时，其最好和目标的dns服务器运行在不同的机器上。

dns 也可以测试 动态更新的性能数据。

## 安装
dnsperf依赖的软件比较多，如果是源码安装那么要注意最好所在的环境安装bind软件。
如果使用yum安装则先执行`yum install -y epel-release`安装`epel`源，然后执行`yum -y install dnsperf`安装`dnsperf`。
## 使用

![](attachments/Pasted%20image%2020240417150421.png)

### 参数说明
**参数说明**：
```bash
-s:  指定 dns 服务器的 ip；
-d:  指定要查询的域名的文件，每一行一个查询记录，查询名称在前，然后空格，然后查询类型。
如：
数据文件中以 "#" 开头的行被认为是注释行，会被dnsperf忽略。
www.example.com A
www.test.com A
其中有效数据由两列组成，第一列是查询域名，第二列是查询的资源类型，dnsperf支持的资源类型如下：
`A`，`NS`，`MD`，`MF`，`CNAME`，`SOA`，`MB`，`MG`，`MR`，`NULL`，`WKS`，`PTR`，`HINFO`，`MINFO`，`MX`，`TXT`，`AAAA`，`SRV`，`NAPTR`，`A6`，`ASFR`，`MAILB`，`MAILA`，`ANY`

-t： 用来指定每个请求的超时时间，以s为单位。默认值是3000ms
-Q： 用来指定最大的QPS。
-c： 用来指定并发探测的client数，默认值是1. 如果指定多个，就是模拟多个clients，即请求是从多个socket发出去的。

-l： 用来指定本次压测的时间，默认值是无穷大。
-e： 本选项通过EDNS0，在OPT资源记录中运用edns-client-subnet来指定真实的client ip. 

-T:   默认情况下，dnsperf使用一个发送线程，一个接收线程。指定了-T，则可以设置N对发送接收线程对。

-n：  最多运行输入文件这么多次。如果没有设置时间限制，则文件将被读取正是这个次数；如果设置了时间限制，文件的读取次数可能会减少。

-y:   向所有发送的数据包添加 TSIG 记录, 使用指定的 TSIG 密钥算法、名称和密钥.


-u:  指示 dnsperf 发送 DNS 动态更新消息，而不是查询。在这种情况下，输入文件的格式是不同的。

-S:  统计间隔，每N秒打印一次qps。
```
### 常见操作
#### dns查询的测试
**dns查询的测试**：
```bash
dnsperf -s 10.3.3.3 -d ./dns.test -n 1000000 -S 1 -Q 100000  -T 16 -c 1000 -t 1
```

**结果说明**：

![](../attachments/Pasted%20image%2020240126103844.png)

标准输出中:
```bash
queies sent是指本次探测发送的总请求数，
queries completed是指本次探测收到响应的请求数，
complete percentage是指本次探测的成功率(queies_completed/queries_sent)，
elapsed time是指本次探测的时间，
queries per second是指本次探测的QPS。
```


#### dns动态更新的测试
**dns动态更新的测试**：


## 范例
### 测试dns查询的性能
#### 编写待解析域名的文件
1. 编写一个文本文件例如`domain.txt`内容如下：
```
www.example.com A
www.test.com A
```
#### dnsperf 执行压测命令
执行命令就开始压测了。
```bash
dnsperf -d domain.txt -s 192.168.2.10 -l 120

-d 用来指定要压测的域名列表文件。  
-s 用来指定要压测的DNS服务器IP地址。  
-l 120用来指定压测的时间，单位是秒
```

#### 性能分析
下图是压测结果，一个`dnsperf`进程压测**缓存应答(缓存解析性能)**，在关闭bind日志的情况下是7.8万 QPS左右。

![](../attachments/Pasted%20image%2020240126103844.png)


下图是压测过程中DNS服务器的top查看截图，
可见1个CPU线程已经完全占用，但并没有使用其他的CPU。
==原因是`dnsperf`压测的时候，它发出来的请求源IP是固定的，源端口也是固定==，那么`bind`软件接到这个请求后就只能`HASH`到固定的1个`CPU`上。
![](../attachments/Pasted%20image%2020240126104012.png)

下面是同时启动2个`dnsperf`压测时DNS服务器的`top`查看截图，可见这个时候有2个CPU被完全占用，`named`的进程使用CPU是`200%`。
![](../attachments/Pasted%20image%2020240126104200.png)

### 测试动态更新的性能

## 优缺点
### 优点

### 缺点
#### 不适合测试链接外网的缓存服务器的性能
dnsperf 适合测试权威dns服务器，不适合测试链接外网的缓存服务器的测试。
dnsperf 使用“自定进度（self-pacing）”方法，该方法基于这样的假设：

您只需向服务器发送一小段连续的查询来填充网络缓冲区，不必考虑什么时候收到响应，然后发送一个新的查询，就可以保持服务器 100% 繁忙。
这种方法非常适合按顺序处理查询且一次处理一个查询的权威服务器。


#### 无法发随机源IP的DNS查询包
因为`dnsperf`这个工具**无法发随机源IP的DNS查询包**，所以是无法很好的测试bind的view性能的。
```bash
注：如果无法指定sip，那么可以通过-y 指定 tsig的方式，也可以匹配不同的 view。
```

`REUSEPORT`特性是基于查询包的二元组（SIP+SPort）来HASH到不同的CPU的，如果源IP固定、源端口固定，所以无法充分使用多CPU。

所以如果要充分测试，还是建议能自己写一个能发随机源IP和源端口的发包工具，并且性能要高。

## 注意
使用 dnsperf 测试的时候，测试设备 和 dns服务器之间的 网络延迟小是很重要的。
最好就是选择测试client和dns测试服务器在同一个机房，ping延迟小于1ms是比较好的。

![](attachments/Pasted%20image%2020240419142944.png)
![](attachments/Pasted%20image%2020240419143037.png)


如上所示，相同的测试服务器，相同的测试脚本。一个是同机房，一个是跨机房的测试。测试的性能差距还是比较大的。


# resperf
## 介绍
resperf - 测试缓存 DNS 服务器的解析性能（resolution performance）。
resperf 是 dnsperf 的配套工具。 dnsperf 主要是为对权威服务器进行基准测试而设计的，它不适用于与实时互联网通信的缓存服务器。

与 dnsperf 的“自定进度（self-pacing）”方法不同，resperf 的工作原理是以受控且稳定增加的速率发送 DNS 查询。默认情况下，resperf会发送流量60秒，线性增加流量 每秒从 0 到 100,000 次查询。

在测试期间，resperf 侦听来自服务器的响应并跟踪响应率、故障率和延迟。停止发送流量后，它还将继续监听 40 秒，以便服务器有时间响应最后发送的查询。



# queryperf
## 介绍
`queryperf`是bind9出品的一款测试dns服务器性能的工具，目前在`9.12.4`版本的bind源码中还存在，再往后的新版本就没看到有`queryperf`了。



使用 queryperf 作性能测试时，最好测试多次，取平均值。
可以修改配置文件的部分参数测试，如，开启递归，开启查询日志等功能作测试。

## 安装
```text
 [root@coredns1 home]# wget https://ftp.isc.org/isc/bind9/9.12.4/bind-9.12.4.tar.gz
 [root@coredns1 home]# tar -zxvf bind-9.12.4.tar.gz
 [root@coredns1 home]# cd bind-9.12.4/contrib/queryperf
 [root@coredns1 queryperf]# ./configure
 [root@coredns1 queryperf]# make
 [root@coredns1 queryperf]# file queryperf
 queryperf: ELF 64-bit LSB executable, x86-64, version 1 (SYSV), dynamically linked (uses shared libs), for GNU/Linux 2.6.32, not stripped
```

## 使用
```bash
[root@coredns1 home]# queryperf -h

DNS Query Performance Testing Tool
Version: $Id: queryperf.c,v 1.12 2007/09/05 07:36:04 marka Exp $


Usage: queryperf [-d datafile] [-s server_addr] [-p port] [-q num_queries]
                 [-b bufsize] [-t timeout] [-n] [-l limit] [-f family] [-1]
                 [-i interval] [-r arraysize] [-u unit] [-H histfile]
                 [-T qps] [-e] [-D] [-R] [-c] [-v] [-h]
  -d specifies the input data file (default: stdin)
  -s sets the server to query (default: 127.0.0.1)
  -p sets the port on which to query the server (default: 53)
  -q specifies the maximum number of queries outstanding (default: 20)
  -t specifies the timeout for query completion in seconds (default: 5)
  -n causes configuration changes to be ignored
  -l specifies how a limit for how long to run tests in seconds (no default)
  -1 run through input only once (default: multiple iff limit given)
  -b set input/output buffer size in kilobytes (default: 32 k)
  -i specifies interval of intermediate outputs in seconds (default: 0=none)
  -f specify address family of DNS transport, inet or inet6 (default: any)
  -r set RTT statistics array size (default: 50000)
  -u set RTT statistics time unit in usec (default: 100)
  -H specifies RTT histogram data file (default: none)
  -T specify the target qps (default: 0=unspecified)
  -e enable EDNS 0
  -D set the DNSSEC OK bit (implies EDNS)
  -R disable recursion
  -c print the number of packets with each rcode
  -v verbose: report the RCODE of each response on stdout
  -h print this usage
```

### 参数说明
**参数说明**:

```bash
 ## queryperf [-d datafile] [-s server_addr] [-p port] [-q num_queries]
 -d: 后面接上一个文件，文件的内容是用户对DNS的请求，一行为一条请求，所以为了测试，我们可以在里面写上几千几万条。
 -s: DNS服务器地址
 -p: DNS服务器端口
 -q: 指定查询的输出的最大数量
 
```

在压测之前需要我们自己准备压测的测试数据，格式为`域名 查询类型`，如：
```text
tinychen.com A
tiny777.com A
tinychen.com MX
tiny777.com MX
tinychen777.com AAAA
```

### 常见操作

常用的操作命令有：
```text
# 对192.168.1.1进行压测，查询域名为文件query.domain.list的内容
queryperf -s 192.168.1.1 -d query.domain.list

# 对192.168.1.1的5353端口进行压测，查询域名为文件query.domain.list的内容
queryperf -s 192.168.1.1 -p 5353 -d query.domain.list

# 对192.168.1.1的5353端口进行压测，查询域名为文件query.domain.list的内容，压测压力为1000qps
queryperf -s 192.168.1.1 -p 5353 -d query.domain.list -T 1000

# 对192.168.1.1的5353端口进行压测，查询域名为文件query.domain.list的内容，压测压力为1000qps，每次查询超时时间为3s
queryperf -s 192.168.1.1 -p 5353 -d query.domain.list -T 1000 -t 3
```

**结果说明**:

![](attachments/Pasted%20image%2020240417154816.png)


## 范例




## 其他

# dnstop
## 背景

当我们分析DNS 服务器日志时，希望了解哪些用户在使用DNS 服务器，同时也希望对DNS 查询做一个统计。一般情况下，可以使用命令“tcpdump –i eth0 port 53”来查看DNS查询包，当然也可以把输出重定向到文件，然后使用rndc stats（bind9）来获取。但这种方法对于初学者而言操作复杂，也不直观。下面介绍的这款工具dnstop，使用起来就非常方便。

## 介绍

虽然log文件可以给我们很多有用的信息，但dnstop可以像top一样帮助我们解决问题，这款软件处了可以即时的对DNS请求做分析，还可以使用tcpdump的输出文件作为它的输入文件来使用。
dnstop 能够即时的输出DNS服务器接受查询的各种类型，并且它还能查找谁在查询你的DNS服务器以及在查询什么类型， 具体什么内容的域名。

## 安装
```bash
yum install -y dnstop
```

## 使用
```bash
 dnstop [-46apsQR] [-b expression] [-i address] [-f filter] [-r interval] [device] [savefile]
 
```
![](attachments/Pasted%20image%2020240417161922.png)

### 参数说明

 **交互参数说明**
![](attachments/Pasted%20image%2020240417162238.png)

在dnstop运行时,可以输入如下按键,获取特定内容
```bash
 s - Sources list
 d - Destinations list
 t - Query types
 o - Opcodes
 r - Rcodes
 1 - 1st level Query Names      ! - with Sources
 2 - 2nd level Query Names	@ - with Sources
 3 - 3rd level Query Names	# - with Sources
 4 - 4th level Query Names	$ - with Sources
 5 - 5th level Query Names	% - with Sources
 6 - 6th level Query Names	^ - with Sources
 7 - 7th level Query Names	& - with Sources
 8 - 8th level Query Names	* - with Sources
 9 - 9th level Query Names	( - with Sources
^R - Reset counters
^X - Exit
 ? - this
```

## 范例

### 指定网卡查看请求
```bash
dnstop -4 -Q -R eth0
```

![](attachments/Pasted%20image%2020240417161829.png)


```bash
dnstop eth01 -l 5 # 监控eth0网卡的dns请求和数据，并最多显示5级域名
然后：? 帮助
然后: %
```
![](attachments/Pasted%20image%2020240417162947.png)

### tcpdump文件作为输入
```bash
dnstop tcpdump.out   //将tcpdump的输出文件tcpdump.out做为dnstop的输入文件

or

dnstop tcpdump.out -l 4
然后输入5 或者 %

```

**查看响应的结果**
如果是抓包的方式，来查看响应结果，响应结果很多，如何查看各个结果的统计，或者异常结果。那么通过这种方式，很容易查看响应的结果。

![](attachments/Pasted%20image%2020240417163925.png)


# 参考
```c
https://zhuanlan.zhihu.com/p/470002059
```