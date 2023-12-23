```table-of-contents
```
# 选项

- `-n`： 表示不要解析域名，直接显示 ip。
- `-nn`： 不要解析域名和端口
- `-X`： 表示同时使用十六进制和 `ASCII` 字符串打印报文的全部数据。
- `-XX`： 同 `-X`，但同时显示以太网头部。
- `-S`： 显示绝对的序列号（sequence number），而不是相对编号。
- `-i`： 选择要捕获的接口；`-i any` 监听所有的网卡
- `-v, -vv, -vvv`：显示更多的详细信息
- `-c number`:  截取 number 个报文，然后结束
- `-A`： 表示使用 `ASCII` 字符串打印报文的全部数据，这样可以使读取更加简单，方便使用 `grep` 等工具解析输出内容。`-A` 和 `-X` 这两个参数不能一起使用。
- `s0` : tcpdump 默认只会截取前 96 字节的内容，要想截取所有的报文内容，可以使用 -s number， number 就是你要截取的报文字节数，如果是 0 的话，表示截取报文全部内容。
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

- `-l`:  如果想实时将抓取到的数据通过管道传递给其他工具来处理，需要使用 `-l` 选项来开启行缓冲模式。
![](attachments/Pasted%20image%2020231221120824.png)
```bash
$ tcpdump -i eth0 -s0 -l port 80 | grep 'Server:'
```


# 过滤规则查看
通过查看 `man pcap-filter` 可以看到 tcpdump的常用过滤规则。如下所示：
![](attachments/Pasted%20image%2020231023103324.png)
# 常用方法
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
具体参考：Linux下的wireshark包

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
### tcp重传
- wireshark 过滤查看
TCP重传十分影响网络性能，往往通过抓包之后，wireshark里打开pcap文件，然后过滤框里输入过滤条件tcp.analysis.retransmission，就能过滤出所有重传的数据包，然后可以通过Statistics里的Summary查看占比.

## 包内容进行过滤
通过 `-A` 选项，可以以 ASCII 码的形式，展示包内容。或者 `-XX` 也可以展示ASCII的形式展示包内容。
进而也可以对包内容以字符串形式进行查询。
![](attachments/Pasted%20image%2020231221115557.png)

截取 http 请求的时候可以用 
```bash
sudo tcpdump -nni ethx -SA port 80
```

从 HTTP 请求头中提取 HTTP 用户代理：
```bash
$ tcpdump -nn -A -s1500 -l | grep "User-Agent:"
```
通过 `egrep` 可以同时提取用户代理和主机名（或其他头文件）：
```bash
$ tcpdump -nn -A -s1500 -l | egrep -i 'User-Agent:|Host:'
```

提取 HTTP 请求的主机名和路径：
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

## 字段偏移过滤
抓取 HTTP GET 流量：
```bash
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x47455420'
```

可以抓取 HTTP POST 请求流量：
```bash
$ tcpdump -s 0 -A -vv 'tcp[((tcp[12:1] & 0xf0) >> 2):4] = 0x504f5354'
```

# 参考
```c

```