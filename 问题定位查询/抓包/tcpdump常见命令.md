```table-of-contents
```
# 是什么
- 通俗的说，tcpdump是一个抓包工具，用于抓取互联网上传输的数据包。
- 学术的说，tcpdump是一种嗅探器（sniffer），利用以太网的特性，通过将网卡适配器（NIC）置于混杂模式（promiscuous）来获取传输在网络中的信息包。
要用tcpdump抓包，请记住，一定要切换到root账户下，因为只有root才有权限将网卡变更为“混杂模式”
所谓混杂模式，也就是嗅探（Sniffering），就是把目的地址不是本机地址的网络报文也抓取下来。


# 选项

- `-n`： 表示不要解析域名，直接显示 ip。
- `-nn`： 不要解析域名和端口
- `-X`：告诉tcpdump命令，需要把协议头和包内容都原原本本的显示出来（tcpdump会以16进制和ASCII的形式显示），这在进行协议分析时是绝对的利器。
- `-XX`： 同 `-X`，但同时显示以太网头部。
- `-S`： 显示绝对的序列号（sequence number），而不是相对编号。
- `-s` 长度，可以只抓取每个报文的一定长度
- `s0` : tcpdump 默认只会截取前 96 字节的内容，要想截取所有的报文内容，可以使用 -s number， number 就是你要截取的报文字节数，如果是 0 的话，表示截取报文全部内容。
- `-i`： 选择要捕获的接口；`-i any` 监听所有的网卡
- `-v, -vv, -vvv`：显示更多的详细信息
- `-c number`:  截取 number 个报文，然后结束
- `-A`： 表示使用 `ASCII` 字符串打印报文的全部数据，这样可以使读取更加简单，方便使用 `grep` 等工具解析输出内容。`-A` 和 `-X` 这两个参数不能一起使用。

- `-p` : 不让网络接口进入混杂模式。
>默认情况下使用 tcpdump 抓包时，会让网络接口进入混杂模式。一般计算机网卡都工作在非混杂模式下，此时网卡只接受来自网络端口的目的地址指向自己的数据。当网卡工作在混杂模式下时，网卡将来自接口的所有数据都捕获并交给相应的驱动程序。如果设备接入的交换机开启了混杂模式，使用 -p 选项可以有效地过滤噪声。

- `-e` : 显示数据链路层信息。
> 默认情况下 tcpdump 不会显示数据链路层信息，使用 -e 选项可以显示源和目的 MAC 地址，以及 VLAN tag 信息。例如：

```bash
$ tcpdump -n -e -c 5 not ip6

tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on br-lan, link-type EN10MB (Ethernet), capture size 262144 bytes
18:27:53.619865 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 1162: 192.168.100.20.51410 > 180.176.26.193.58695: Flags [.], seq 2045333376:2045334484, ack 3398690514, win 751, length 1108
18:27:53.626490 00:e2:69:23:d3:3b > 24:5e:be:0c:17:af, ethertype IPv4 (0x0800), length 68: 220.173.179.66.36017 > 192.168.100.20.51410: UDP, length 26
18:27:53.626893 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 1444: 192.168.100.20.51410 > 220.173.179.66.36017: UDP, length 1402
18:27:53.628837 00:e2:69:23:d3:3b > 24:5e:be:0c:17:af, ethertype IPv4 (0x0800), length 1324: 46.97.169.182.6881 > 192.168.100.20.59145: Flags [P.], seq 3058450381:3058451651, ack 14349180, win 502, length 1270
18:27:53.629096 24:5e:be:0c:17:af > 00:e2:69:23:d3:3b, ethertype IPv4 (0x0800), length 54: 192.168.100.20.59145 > 192.168.100.1.12345: Flags [.], ack 3058451651, win 6350, length 0
5 packets captured
```

- `-l`:  默认情况下是**全缓冲**的，`-l` 可以将tcpdump的输出变为“**行缓冲**”方式。
如果想实时将抓取到的数据通过管道传递给其他工具来处理，需要使用 `-l` 选项来开启行缓冲模式。
![](attachments/Pasted%20image%2020231221120824.png)
```bash
$ tcpdump -i eth0 -s0 -l port 80 | grep 'Server:'
```
需求： “对于tcpdump输出的内容，提取每一行的第一个域，即”时间域”，并输出出来，为后续统计所用” ----> `# tcpdump -i ens33 -l |awk '{print $1}'` ----> 如果不加-l选项，那么只有全缓冲区满，才会输出一次，这样不仅会导致输出是间隔不顺畅的，而且当你ctrl-c时，很可能会断到一行的半截，损坏统计数据的完整性。

- `-P`：指定要抓取的包的方向。是流入还是流出的包。可以给定的值为"in"、"out"和"inout"，默认为"inout"。

![](attachments/Pasted%20image%2020240429143830.png)

# 过滤器
网络报文是很多的，很多时候我们在主机上抓包，会抓到很多我们并不关心的无用包，然后要从这些包里面去找我们需要的信息，无疑是一件费时费力的事情，tcpdump 提供了灵活的语法可以精确获取我们关心的数据，这些语法说得专业点就是过滤器。

## 分类
过滤器简单可分为三类：协议（proto）、传输方向（dir）和类型（type）。

一般的 **表达式格式** 为：

![](attachments/Pasted%20image%2020240429144211.png)

- 关于 proto：可选有 ip, arp, rarp, tcp, udp, icmp, ether 等，默认是所有协议的包
- 关于 dir：可选有 src, dst, src or dst, src and dst，默认为 src or dst
- 关于 type：可选有 host, net, port, portrange（端口范围，比如 21-42），默认为 host

## 过滤规则查看
通过查看 `man pcap-filter` 可以看到 tcpdump的常用过滤规则。如下所示：
![](attachments/Pasted%20image%2020231023103324.png)

# 过滤查询
## ipv4头相关过滤
### 基础知识
```ruby
0                   1                   2                   3
0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version|  IHL  |Type of Service|          Total Length         |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Identification        |Flags|      Fragment Offset    |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|  Time to Live |    Protocol   |         Header Checksum       |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                       Source Address                          |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Destination Address                        |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                    Options                    |    Padding    | <-- optional
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                            DATA ...                           |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```
```c
/*IP头定义，共20个字节*/
typedef struct _IP_HEADER
{
    char m_cVersionAndHeaderLen;     　　//版本信息(前4位)，头长度(后4位)
    char m_cTypeOfService;      　　　　　 // 服务类型8位
    short m_sTotalLenOfPacket;    　　　　//数据包长度
    short m_sPacketID;      　　　　　　　 //数据包标识
    short m_sSliceinfo;      　　　　　　　  //分片使用
    char m_cTTL;        　　　　　　　　　　//存活时间
    char m_cTypeOfProtocol;    　　　　　 //协议类型
    short m_sCheckSum;      　　　　　　 //校验和
    unsigned int m_uiSourIp;     　　　　　//源ip
    unsigned int m_uiDestIp;     　　　　　//目的ip
} __attribute__((packed))IP_HEADER, *PIP_HEADER ;
```

如何从包头过滤信息，如下所示：
```shell
proto[x:y]          : 过滤从x字节开始的y字节数。比如ip[2:2]过滤出3、4字节（第一字节从0开始排）
proto[x:y] & z = 0  : proto[x:y]和z的与操作为0
proto[x:y] & z !=0  : proto[x:y]和z的与操作不为0
proto[x:y] & z = z  : proto[x:y]和z的与操作为z
proto[x:y] = z      : proto[x:y]等于z
```
### 含有ip option的包
`一般`的IP头是20字节，但IP头有选项设置，不能直接从偏移21字节处读取数据。IP头有个长度字段可以知道头长度是否大于20字节。
通常第一个字节的二进制值是：01000101，

分成两个部分：

> **0100 = 4** 表示IP版本
> 
> **0101 = 5** 表示IP头32 bit的块数，
> 
> 5 x 32 bits = 160 bits or 20 bytes

如果第一字节第二部分的值大于5，那么表示头有IP选项。

```shell
tcpdump -i eth1 'ip[0] & 15 > 5'
or
tcpdump -i eth1 'ip[0] & 0x0f > 5'　
```
### 抓取分片包
当发送端的MTU大于到目的路径链路上的MTU时就会被分片。

```c
id: 16bit
flags: 3Bit
frag offset: 13Bit
```
**Bit 0**: 保留，必须是0  
**Bit 1**: (DF) 0 = 可能分片, 1 = 不分片  
**Bit 2**: (MF) 0 = 最后的分片, 1 = 还有分片

**Fragment Offset**字段只有在分片的时候才使用。

- 抓取非分片的包
要抓带DF位标记的不分片的包，第七字节的值应该是：
0100 0000 = 64
```shell
tcpdump -i eth1 'ip[6] = 64'
```

- 抓取前面的分片
匹配MF，分片包。 即MF=1. DF=0 此时抓取的是分片包且不是最后一个分片。
```shell
tcpdump -i eth1 'ip[6] = 32'
```

- 匹配最后分片
此时抓取的是存在frag_offeset
```shell
tcpdump -i eth1 '((ip[6:2] > 0) and (not ip[6] = 64))'
```

- 抓取所有的分片
```shell
tcpdump -nni eth01 '(ip[6] = 32)' or '((ip[6:2] > 0) and (not ip[6] = 64))'
```
#### 测试分片
```shell
ping -M want -s 3000 192.168.1.1
```
![](attachments/Pasted%20image%2020231019192421.png)

### 根据包长度过滤
```shell
- 大于600字节
tcpdump -i eth1 'ip[2:2] > 600'
```
## ipv6过滤
抓取 IPv6 流量：
```bash
tcpdump -nni ethx ip6
```
## 四层头过滤
### tcp重传过滤
**wireshark 过滤查看**
TCP重传十分影响网络性能，往往通过抓包之后，wireshark里打开pcap文件，然后过滤框里输入过滤条件`tcp.analysis.retransmission`，就能过滤出所有重传的数据包，然后可以通过Statistics里的Summary查看占比.


**其他方法**
通过 wireshark 进行分析重传，如果pcap文件很大，那么就会导致 使用 wireshark 异常的慢。可以通过其他的方式来找到重传的数据包，并且一目了然。
```bash
比如：查找 server对外发出的fin-ack重传的包；

# 先将所有输出到 text 文件中
tcpdump -r vs_103.102.202.153_80_p1.pcap -nn > aa.out


# 对text文件进行操作，对指定的输出念排序，比如基于 server发出的 fin 包进行 ip:port 对进行排序。
cat aa.out | grep -v 103.102.202.153 | egrep  "Flags \[F\.\]" | grep "\.80 >" | sort -k3 -k4 > bb.out

cat bb.out | awk '{print $3 " >  " $5}' | sort | uniq -c > dd.out

从 dd.out 中 ，可以轻易的得到 重传的信息。

awk 某一列元素出现的个数：
cat 11.out | awk '{ count[$5]++} END{for(i in count) if(count[i]>=3)print i " " count[i]}'

or

cat logfile | awk '{print $3}' | uniq -c
```

## 包内容进行过滤
通过 `-A` 选项，可以以 ASCII 码的形式，展示包内容。或者 `-XX` 也可以展示ASCII的形式展示包内容。
进而也可以对包内容以字符串形式进行查询。
![](attachments/Pasted%20image%2020231221115557.png)

### 范例
**对指定的包内容进行处理**：
```bash
tcpdump -r vip6_4666_2.pcap -nn ip -X | grep "0x0010:" | awk '{print $9}' | sort | uniq -c | sort -n
```

**截取 http 请求的时候可以用 **
```bash
sudo tcpdump -nni ethx -SA port 80
```

**从 HTTP 请求头中提取 HTTP 用户代理**：
```bash
$ tcpdump -nn -A -s1500 -l | grep "User-Agent:"
```
通过 `egrep` 可以同时提取用户代理和主机名（或其他头文件）：
```bash
$ tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'
```

**提取 HTTP 请求的主机名和路径**：
```bash
$ tcpdump -s 0 -v -n -l | egrep -i "POST /|GET /|Host:"

tcpdump: listening on enp7s0, link-type EN10MB (Ethernet), capture size 262144 bytes
    POST /wp-login.php HTTP/1.1
    Host: dev.example.com
    GET /wp-login.php HTTP/1.1
    Host: dev.example.com
    GET /favicon.ico HTTP/1.1
    Host: dev.example.com
    GET / HTTP/1.1
    Host: dev.example.com
```

### wireshark 包内容过滤
比如，自定义的四层之上的协议。
`tcp.payload contains "request header"` 来直接从 payload 里面过滤请求。


## 字段偏移过滤
### 用法
tcpdump 支持我们根据数据包的标志位进行过滤
```bash
proto [ expr:size ]
```
- `proto`：可以是熟知的协议之一（如ip，arp，tcp，udp，icmp，ipv6）
- `expr`：可以是数值，也可以是一个表达式，表示与指定的协议头开始处的字节偏移量。
- `size`：是可选的，表示从字节偏移量开始取的字节数量。

```bash
`tcp[n]`：表示 tcp 报文里 第 n 个字节
`tcp[n:c]`：表示 tcp 报文里从第n个字节开始取 c 个字节，tcp[12:1] 表示从报文的第12个字节（因为有第0个字节，所以这里的12其实表示的是13）
```
tcpflags 可以理解为是一个别名常量，相当于 13，它代表着与指定的协议头开头相关的字节偏移量，也就是标志位，所以 `tcp[tcpflags]` 等价于 `tcp[13]`。

tcp-fin, tcp-syn, tcp-rst, tcp-push, tcp-ack, tcp-urg 这些同样可以理解为别名常量，分别代表 1，2，4，8，16，32，64。


下面以最常见的 syn包为例，演示一下如何用 tcpdump 抓取到 syn 包，而其他的类型的包也是同样的道理。
- 第一种写法：使用数字表示偏移量
```bash
$ tcpdump -i eth0 "tcp[13] & 2 != 0"

$ tcpdump -i eth0 'tcp[13] == 2 or tcp[13] == 16'
```

- 第二种写法：使用别名常量表示偏移量
```bash
$ tcpdump -i eth0 "tcp[tcpflags] & tcp-syn != 0"

$ tcpdump -i eth0 'tcp[tcpflags] == tcp-syn or tcp[tcpflags] == tcp-ack'
```

- 第三种写法：使用混合写法
```bash
$ tcpdump -i eth0 "tcp[tcpflags] & 2 != 0" 

or 

$ tcpdump -i eth0 "tcp[13] & tcp-syn != 0"
```
### 范例
**抓取 HTTP GET 流量**：
```bash
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

**可以抓取 HTTP POST 请求流量**：
```bash
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

**使用偏移量的方法，抓取TLS 握手阶段的 Client Hello 报文**
```bash
tcpdump -w file.pcap 'dst port 443 && tcp[20]==22 && tcp[25]==1'
```
分析：
```text
dst port 443：这个最简单，就是抓取从客户端发过来的访问 HTTPS 的报文。

tcp[20]=22：这是提取了 TCP 的第 21 个字节（因为初始序号是从 0 开始的），由于 TCP 头部占 20 字节，TLS 又是 TCP 的载荷，那么 TLS 的第 1 个字节就是 TCP 的第 21 个字节，也就是 TCP[20]，这个位置的值如果是 22（十进制），那么就表明这个是 TLS 握手报文。

tcp[25]=1：同理，这是 TCP 头部的第 26 个字节，如果它等于 1，那么就表明这个是 Client Hello 类型的 TLS 握手报文。
```

用偏移量方法，写一个 tcpdump 抓取 TCP SYN 包的过滤表达式
```bash
tcpdump 'tcp[13]&2 != 0'
```

如果要指定只抓取 SYN 包而不抓取 SYN+ACK，可以用下面的表达式
```bash
tcpdump 'tcp[13]|2 = 2'
```

过滤出 TCP RST 报文（TCPDUMP 预定义）
```bash
tcpdump -w file.pcap 'tcp[tcpflags]&(tcp-rst) != 0'  
# 偏移量的写法  
tcpdump -w file.pcap 'tcp[13]&4 != 0'
```

## 过滤后转存
有时候，我们想从抓包文件中过滤出想要的报文，并转存到另一个文件中。比如想从一个抓包文件中找到 TCP RST 报文，并把这些 RST 报文保存到新文件。那么就可以这么做：
```bash
tcpdump -r file.pcap 'tcp[tcpflags] & (tcp-rst) != 0' -w rst.pcap
```

# 其他方法
## 任意口抓包
```bash
tcpdump -nni any xxxx
```

![](attachments/Pasted%20image%2020240429200503.png)

如上所示：正常情况下，抓包是不会显示In/out的。只有抓包口为 any，并且 `-e ` 打印mac时，会 显示 in、out 标识。

> 注： 可以通过 `-Q in/out/inout`的方向过滤。
```bash
比如：只抓取 eth02 发出去的包。

tcpdump -nni eth02 -Q out
```

## 保存包到文件的同时显示当前抓到的包

```
(1) 笨方法一：
tcpdump 过滤条件 -w  aaa.out
tcpdump -r  aaa.out -nn

(2) 笨方法二：

tcpdump 过滤条件 -l | tee arp.pcap
注：此中保存的 arp.pcap 是文本文件，而不是二进制文件。

```

合理的方法：
```bash
tcpdump 过滤条件 -U -w - | tee arp.pcap | tcpdump -nn -r -

注：此时保存的 arp.pcap 是二进制文件，而不是文本文件。
后续可以通过：tcpdump -r arp.pcap -nn 读取的。
如果是文本文件，则只能 cat arp.pcap 读取。

比如：
tcpdump -nni eth03 host not 192.20.29.1 and host not 192.20.29.2 and net 192.20.29 -v -U -w - | tee arp2.pcap | tcpdump -nn -r -
```

说明：
```bash
-U 文件及时写入，而不是缓存一些后再写;  
-w 把抓到的内容写到文件中，其中后面跟的文件是一个"-", 表示标准输入/输出。

然后用 tee 把标准输出保存成文件，然后继续用 tcpdump -r 把标准输入解析显示出来。
```
![](attachments/Pasted%20image%2020240322140656.png)

## 抓取n个包
## 抓取指定时间的包
```c
1》抓取30s的包
timeout 30 tcpdump -nni ethx ...

2》抓取报文后隔指定的时间保存一次
tcpdump -nni eth3 -s0 -G 60 -Z root -w %Y_%m%d_%H%M_%S.pcap
-G选项 后面接时间 单位为秒 本例中的时间为60秒
-s0 表示数据包不进行截断。
-Z user
```

## pcap包的分割和合并
主要是使用了Linux下的 wireshark  包中的 editcap 与 mergecap 工具。
具体参考：**Linux下的wireshark工具包**

## 指定抓包长度
我们给tcpdump 加上 -s 参数，指定抓取的每个报文的最大长度，就节省抓包文件的大小。
-s 这个长度参数，它的使用场景其实就包括了延长抓包时间，因为减少了抓包的大小。

一般来说，帧头是 14 字节，IP 头是 20 字节，TCP 头是 20~40 字节。如果你明确地知道这次抓包的重点是传输层，那么理论上，对于每一个报文，你只要抓取到传输层头部即可，也就是前 14+20+40 字节（即前 74 字节）：
```bash
tcpdump -s 74 -w file.pcap
```

```bash
如果包长很大，那么可能存在截断 。
-s0:  即不限制抓包长度

$ tcpdump -i eth0 -s0 -l port 80 | grep 'Server:'
```

## 显示tcp的相对以及绝对序列号
```
`-S`： 显示绝对的序列号（sequence number），而不是相对编号。
```

## 查询一段时间来源最多的IP地址
**背景**
比如：dns服务器，某一段时间压力比较大，但是又没有打开日志，想知道哪些client的服务器造成了dns查询的压力。
> 注：由于dns的查询主要是一问一答的2个包。所以，比如容易知道dns查询的压力。
> 如果是TCP流量，则最好是基于TCP的SYN包进行过滤，然后基于SIP的个数进行排序。


**解决**
```bash
tcpdump -r xxxxx.pcap -nnn -t | cut -f 1,2,3,4 -d '.' | sort | uniq -c | sort -nr | head -n 20


说明：
-t: 表示不输出时间。
cut -f 1,2,3,4 -d '.' : 以 `.` 为分隔符，打印出每行的前四列。即 IP 地址。
```

# tcptrace
## 背景
有时候我们并不方便用 Wireshark 打开抓包文件做分析，比如抓包的机器不允许向外传文件，也就是可能只能在这台机器上做分析。
我们可以用 tcpdump -r 的方式，打开原始抓包文件看看：
```bash
$ tcpdump -r test.pcap | head -10
reading from file test.pcap, link-type EN10MB (Ethernet)
03:55:10.769412 IP victorebpf.51952 > 180.101.49.12.https: Flags [S], seq 3448
03:55:10.779061 IP 180.101.49.12.https > victorebpf.51952: Flags [S.], seq 156
03:55:10.779111 IP victorebpf.51952 > 180.101.49.12.https: Flags [.], ack 1, w
03:55:10.784134 IP victorebpf.51952 > 180.101.49.12.https: Flags [P.], seq 1:5
03:55:10.784297 IP 180.101.49.12.https > victorebpf.51952: Flags [.], ack 518,
03:55:10.795094 IP 180.101.49.12.https > victorebpf.51952: Flags [P.], seq 1:1
03:55:10.795118 IP victorebpf.51952 > 180.101.49.12.https: Flags [.], ack 1502
03:55:10.795327 IP 180.101.49.12.https > victorebpf.51952: Flags [P.], seq 150
03:55:10.795356 IP victorebpf.51952 > 180.101.49.12.https: Flags [.], ack 3881
03:55:10.802868 IP 180.101.49.12.https > victorebpf.51952: Flags [P.], seq 388
```
报文都是按时间线原样展示的，缺乏逻辑关系，是不是难以组织起有效的分析？比如，要搞清楚里面有几条 TCP 连接都不太容易。这时候怎么办呢？


# 参考
```c
# TCP 实战抓包分析
https://xiaolincoding.com/network/3_tcp/tcp_tcpdump.html#%E6%98%BE%E5%BD%A2-%E4%B8%8D%E5%8F%AF%E8%A7%81-%E7%9A%84%E7%BD%91%E7%BB%9C%E5%8C%85

# 全网最全 tcpdump 抓包指南
https://iswbm.com/70.html
```