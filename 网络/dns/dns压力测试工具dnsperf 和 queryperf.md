```table-of-contents
```
# 概述
DNS的压力测试工具最常用（或者说最好找）的就是`dnsperf`和`queryperf`。其中`queryperf`是bind软件自带的，`dnsperf`软件是`Nominum`提供的一款开源压测软件。
# dnsperf
## 介绍
## 安装
dnsperf依赖的软件比较多，如果是源码安装那么要注意最好所在的环境安装bind软件。
如果使用yum安装则先执行`yum install -y epel-release`安装`epel`源，然后执行`yum install dnsperf`安装`dnsperf`。
## 使用
## 范例
### 范例一
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
![](attachments/Pasted%20image%2020240126103844.png)


下图是压测过程中DNS服务器的top查看截图，
可见1个CPU线程已经完全占用，但并没有使用其他的CPU。==原因是`dnsperf`压测的时候，它发出来的请求源IP是固定的，源端口也是固定==，那么`bind`软件接到这个请求后就只能`HASH`到固定的1个`CPU`上。
![](attachments/Pasted%20image%2020240126104012.png)

下面是同时启动2个`dnsperf`压测时DNS服务器的`top`查看截图，可见这个时候有2个CPU被完全占用，`named`的进程使用CPU是`200%`。
![](attachments/Pasted%20image%2020240126104200.png)

#### 注意
注意：因为`dnsperf`这个工具无法发随机源IP的DNS查询包，所以是无法很好的测试bind性能的，尤其是bind的多CPU多线程测试。`REUSEPORT`特性是基于查询包的二元组（SIP+SPort）来HASH到不同的CPU的，如果源IP固定、源端口固定，所以无法充分使用多CPU。所以如果要充分测试，还是建议能自己写一个能发随机源IP和源端口的发包工具，并且性能要高。


# queryperf
## 介绍
`queryperf`是bind9出品的一款测试dns服务器性能的工具，目前在`9.12.4`版本的bind源码中还存在，再往后的新版本就没看到有`queryperf`了。

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

在压测之前需要我们自己准备压测的测试数据，格式为`域名 查询类型`，如：
```text
tinychen.com A
tiny777.com A
tinychen.com MX
tiny777.com MX
tinychen777.com AAAA
```

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

## 范例


# 参考
```c
https://zhuanlan.zhihu.com/p/470002059
```