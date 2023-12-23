```table-of-contents
```
# DNS报文结构
## 构成
DNS的请求和响应的基本单位是**DNS报文**（Messgae）。请求和响应的DNS报文结构是完全相同的，每个报文都由以下五段（Section）构成：
```c
+---------------------+
|        Header       | 报文头，固定12字节
+---------------------+
|       Question      | 查询的Question
+---------------------+
|        Answer       | Question部分对应的应答记录
+---------------------+
|      Authority      | 权威服务器资源记录
+---------------------+
|      Additional     | 附加信息资源记录
+---------------------+
```
## Header部分
DNS Header是每个DNS报文都必须拥有的一部分，它的长度固定为12个字节(每一行2个字节)，它拥有如下的结构：
```c
  0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      ID                       |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|QR|   Opcode  |AA|TC|RD|RA|   Z    |   RCODE   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    QDCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ANCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    NSCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    ARCOUNT                    |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
- **ID**
ID是一个16bit长标识符，可以理解为一个2字节的无符号数。该字段由发起查询请求的客户端生成。服务端在生成响应报文时会原样复制这个值，这意味着相同ID的请求和响应报文是一对请求。因此客户端可以根据这个字段得知某个响应报文对应的是哪次查询请求。

- ** QR**
QR是1bit flag位，值为0表示这是一次查询，1则表示该报文是响应报文。

- ** Opcode**
4bit长的查询类型字段，这个值由查询发起者设置，在响应时原样复制，我们主要使用Query查询，其它几种基本见不到。

|   |   |   |
|---|---|---|
|0|Query（最常用的查询）|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|1|IQuery (反向查询，现在已经不再使用了)|[[RFC3425](http://www.iana.org/go/rfc3425)]|
|2|Status|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|3|未指定||
|4|Notify|[[RFC1996](http://www.iana.org/go/rfc1996)]|
|5|Update|[[RFC2136](http://www.iana.org/go/rfc2136)]|
|6|DNS Stateful Operations (DSO)|[[RFC8490](http://www.iana.org/go/rfc8490)]|
|7-15|未指定|

- **AA(Authoritative Answer)权威应答标志位**
1bit权威应答标记，当响应报文由权威服务器发出时，该位置1，否则为0。

- **TC(TrunCation)截断标志位**
当使用UDP传输时，若响应数据超过DNS标准限制（超过512B），数据包便会发生截断，超出部分被丢弃，此时该flag位被置1。

当客户端发现TC位被置1的响应数据包时应该选择使用TCP重新发送查询。因为TCP DNS报文不受512字节限制。

- **RD(Recursion Desired)递归查询期望标志位**
客户端希望服务器对此次查询进行递归查询时将该位置1，否则置0。响应时RD位会复制到响应报文内。

- **RA(Recursion Available)递归查询可用标志位**
服务器根据自己是否支持递归查询对该位进行设置。1为支持递归查询，0为不支持递归查询。

- **Z 保留段**
这三个bit未在[RFC1035](https://tools.ietf.org/html/rfc1035)中指定用途，保留到以后升级时使用。

- **RCODE(Response Code)响应码**
这个字段在响应时进行设置：

|RCODE||Reference|
|---|---|---|
|0|没有错误。|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|1|Format error：格式错误，服务器不能理解请求的报文格式。|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|2|Server failure：服务器失败，因为服务器的原因导致没办法处理这个请求。|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|3|Name Error：名字错误，该值只对权威应答有意义，它表示请求的域名不存在。|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|4|Not Implemented：未实现，域名服务器不支持该查询类型。|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|5|Refused：拒绝服务，服务器由于设置的策略拒绝给出应答。比如，服务器不希望对个请求者给出应答时可以使用此响应码。|[[RFC1035](http://www.iana.org/go/rfc1035)]|

在[RFC1035](https://tools.ietf.org/html/rfc1035)中，6-15的RCODE未被指派。在后期IETF新指定了许多RCODE，同时在EDNS0内，该字段被扩展至了12bit，这意味着响应码的数量由16升至了4096。详细完整的值与含义可以访问[这里查看](https://www.iana.org/assignments/dns-parameters/dns-parameters.xhtml#dns-parameters-6)。

- **QDCOUNT，ANCOUNT，NSCOUNT，ARCOUNT**
这四个字段都是一个16bit无符号整数，分别表示后面四个数据段内条目的个数。

## Question段/Query段
question部分存放的是向服务器查询的域名数据。它由**QDCOUNT**个“**条目**”(Entry)组成。一般情况下它只有一条Entry。

每个Entry的格式是相同的，如下所示：
```c
      0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                                               |
    /                     QNAME                     /
    /                                               /
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QTYPE                     |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
    |                     QCLASS                    |
    +--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
## Answer，Authority，Additional段
# 参考
```c
DNS报文格式
https://yangwang.hk/?p=878

# 使用Wireshark进行DNS协议解析
https://mp.weixin.qq.com/s?__biz=MzIxNTExNTcxMg==&mid=2247483877&idx=1&sn=48d797977ec96a3d5117f9da9403ff84&chksm=979c734aa0ebfa5c80147f1dc3f58ed7869c8f5f1b57d249d6c747538b2a65d9459b00a8d560&scene=21#wechat_redirect

## 万字长文爆肝 DNS 协议
https://cloud.tencent.com/developer/article/1779032
```
