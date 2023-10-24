```table-of-contents
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

## 四层头过滤
### tcp重传
- wireshark 过滤查看
TCP重传十分影响网络性能，往往通过抓包之后，wireshark里打开pcap文件，然后过滤框里输入过滤条件tcp.analysis.retransmission，就能过滤出所有重传的数据包，然后可以通过Statistics里的Summary查看占比.


# 参考