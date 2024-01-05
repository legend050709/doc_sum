```table-of-contents
```
# DNS查询与响应
DNS工作在UDP和TCP上，都使用53端口。
在大多数情况下，DNS请求和响应都使用UDP数据包，此时UDP的payload便直接是DNS的报文。
当使用TCP时，整个DNS报文部分没有任何变化，但是由于TCP是一个字节流协议，你需要额外指定一个报文长度来进行报文划分。这个长度字段长2字节，在DNS报文之前发送。因此对于报文”Message“，使用TCP时，实际传输的数据是：
```bash
[2字节Message报文长度][Message具体内容]
```
同样的，这个长度字段使用网络序。
由于UDP只提供尽力而为的服务，请求报文和响应报文都可能在链路上丢失。因此为了保证查询的可靠性，**使用UDP进行DNS查询时必须设计重传机制**。
RFC1035建议，超时重传的时间应该尽可能的通过分析统计得出，建议重传时间为2-5秒。

# DNS查询与响应的报文结构
## 构成
DNS的请求和响应的基本单位是**DNS报文**（Messgae）。请求和响应的DNS报文结构是完全相同的，每个报文都由以下五段（Section）构成：
1、首部（Header）  
2、问题（Question）  
3、回答（Answer）  
4、权威（Authority）  
5、附加（Additional）
![](attachments/Pasted%20image%2020240104104202.png)
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
DNS Header是每个DNS报文都必须拥有的一部分，它的长度固定为**12B**。
每一类DNS报文都有一个首部部分。首部包括一些字段，用于规定其余部分是否存在，也是规定是查询还是响应，是标准查询还是其他类型的操作。
![](attachments/Pasted%20image%2020240104104248.png)

如下所示，每一行2个字节，它拥有如下的结构：
```c
  0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     Transaction ID            |
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
- **Transaction ID(16bit)**
ID是一个16bit长标识符。
该字段由发起查询请求的客户端生成。服务端在生成响应报文时会原样复制这个值，这意味着相同ID的请求和响应报文是一对请求。
一对请求和应答报文的会话标识字段是相同的，四元组和会话标识一起，可以将DNS应答报文和请求报文一一对应。

- **QR(1bit)**
QR(Query-Response)是1bit flag位，查询/响应的标志位。
值为0表示这是一次查询，1则表示该报文是响应报文。

- **opcode(4bit)**
4bit长的查询类型字段，这个值由查询发起者设置，在响应时原样复制，我们主要使用Query查询，其它几种基本见不到。
0表示标准查询，1表示反向查询，2表示服务器状态请求。

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

- **AA(Authoritative Answer)(1bit)**
权威应答标志位。1bit权威应答标记，当响应报文由权威服务器发出时，该位置1，否则为0。

- **TC(TrunCation)(1bit)**
截断标志位。当使用UDP传输时，若响应数据超过DNS标准限制（超过512B），数据包便会发生截断，只返回前 512 个字节，超出部分被丢弃，此时该flag位被置1。
> 注：当客户端发现TC位被置1的响应数据包时应该选择使用TCP重新发送查询。因为TCP DNS报文不受512字节限制。

- **RD(Recursion Desired)（1bit）**
期望递归。该字段能在一个查询中设置，并在响应中返回。客户端希望服务器对此次查询进行递归查询时将该位置1，否则置0。响应时RD位会复制到响应报文内。

- **RA(Recursion Available)（1bit）**
可用递归。服务器根据自己是否支持递归查询对该位进行设置。1为支持递归查询，0为不支持递归查询。

- **Z （zero）保留段（3bit）**
保留字段，在所有的请求和应答报文中，它的值必须为 0。这三个bit未在[RFC1035](https://tools.ietf.org/html/rfc1035)中指定用途，保留到以后升级时使用。

- **RCODE(Response Code)(4bit)**
响应码。表示响应的差错状态。
0表示没有差错，1表示格式错误，32表示服务器错误，3表示域参照问题，4表示查询类型不支持，5表示被禁止，其它被保留。
![](attachments/Pasted%20image%2020240104105307.png)


- **QDCOUNT**
问题数。

- **ANCOUNT**
Answer RRs。回答资源记录数
> RRs 即 Resource Records，资源记录数量。

- **NSCOUNT**
Authority RRs。权威资源记录数。


- **ARCOUNT**
Additional RRs。附加资源记录数。


在头部之后，为正文的四大区域，分成两类：
1、Queries区域，包括数量不等的查询块
2、资源记录（RR）区域，即Answers、Authorities、Additional records三个区域。每个区域均包括数量不等的资源记录块。

## Question段/Query段
![](attachments/Pasted%20image%2020240104105939.png)
question部分存放的是向服务器查询的域名数据。它由**QDCOUNT**个条目(Entry)组成。一般情况下它只有一条Entry。

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

比如：
![](attachments/Pasted%20image%2020240104110040.png)
### DNS标准名称表示法
DNS使用一种标准格式对域名进行编码。它由一系列的**label**（和域名中用.分割的label不同）构成。
每个lebel首字节的高两位用于表示label的类型。这意味着一共可直接分配四种label类型。RFC1035中分配了四个里面的两个，分别是：`00`表示的普通label，`11`表示的压缩label。
#### 普通label(00xxxxxx)
当label首字节的高两位为0时，表示这是一个普通label。此时使用该字节剩下的6bit（也就是xxxxxx部分）表示该label后续的长度。由于标识长度的部分只有6bit，因此它的取值范围就成了0x00-0x3f（0-63），换句话说，域名的一个label最长为63个字节。

后面的部分和域名的那个label完全相同。例如`www.example.com`中第一个label `www` 使用普通label编码时会得到二进制序列：`**[0x03]['w']['w']['w']**`;
由于所有域名最后都会有一个长度为0的根域名，因此该编码下便会有个值为`0x00`的label来标识域名末尾。
`www.example.com`的完整编码是“`[3]www[7]example[3]com[0]`“，画个图的话就是：
![](attachments/Pasted%20image%2020231229203626.png)

#### 压缩label(11xxxxxx)
**背景**
由于一个DNS报文内往往拥有多个域名，大多数情况下这些域名有一部分是完全相同的。比如13个根域名服务器地址的唯一的区别是三级域名由a编号至m，后面的15个字节的label完全相同（`.root-servers.net`）。
如果所有域名都按照上述方法原样表示，那么会因为冗余数据浪费许多空间。为此，标准规定了一种针对域名的压缩表示法，即使用一个**压缩指针**来对域名数据进行复用。生成报文时压缩并不是强制规定必须实现的。

**压缩指针offset**
label首字节的高两位为11时表示这个label是压缩表示的，即为一个压缩指针。换句话说，当你发现某个label首字节的值大于等于0xc0(十进制的192)时，说明这是一个指针。
压缩指针长两字节，指针的值是第一个字节的低六位和第二个字节，一共14位。
```bash
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
| 1  1|                OFFSET                   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
**OFFSET**便是指针，这个OFFSET表示label的真实数据位于整个数据包的offset处。和数组下标类似，数据包的第一个字节offset为0。

下面是一个真实DNS数据包看起来的样子：
![](attachments/Pasted%20image%2020231229204110.png)
假设依次在数据包的`0x0C`，`0x2E`和`0x4C`(此处的`0x0C`、`0x2E`和`0x4C`是相对于DNS Header的位置)处依次出现了三条域名：`example.com`，`www.example.com`，`www.example.com`。由于后两条域名的`example.com`部分与第一条相同，因此可以使用压缩指针进行压缩。

对于第二条域名，第一个label `www`并未在以前域名中出现，因此第一个label需要直接给出，从第二个label起，example.com已经在数据包0x0C处有了一份，因此构造指针`0xC00C`(11000000 00001100)，最高两bit表示这是一个指针，后14bit指定了接下来的label位于整个报文的偏移量（第一个字节偏移量是0），也就是0C。

第三条域名同理，不过相同的部分变成了整个域名，因此直接提供指针即可。压缩是可以嵌套的，比如这里压缩指针`0xC02E`指向的是第二条域名，而第二条域名本身也使用了压缩。
> 注：使用压缩指针之后，就不用在指针最后加一个0x00表示名称结束了，因为顺着指针找下去总能找到表示末尾的0。

**注意**
压缩指针的含义是，当前域名的label序列从此label开始，与指针指向的label序列是相同的。很显然压缩结果不是唯一的，比如第三条域名也可以压缩成`[0x03]www[0xc0][0x0c]`。

#### 邮件名
有时候需要在一个字段内表示一个电子邮件地址，我们知道邮件地址的格式是`<name>@<domain-name>`，为了能方便复用，DNS内将@字符替换为一个”.”，如此便能沿用上面域名的表示方法。例如`admin@example.com`使用名称表示法后结果等同于`admin.example.com`。

### QNAME
查询名部分长度不定，一般为要查询的域名(也会有IP的时候，即反向查询)。
> 长度不定，因此有可能出现奇数个字节，但不进行补齐。

qname 由`labels`序列构成的域名。QNAME的格式使用DNS标准名称表示法。


### QTYPE
域名的资源类型，长度是两个字节。这意味着最多可以有65536种不同的资源。
FRC1035内指定了编号为1-16的16种资源类型，而当前则一共定义了近100种资源类型，这其中常用的类型并不是很多，大约只有以下十几种。

|类型|值|含义|REFERENCE|
|---|---|---|---|
|A|1|主机地址|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|NS|2|域名服务器记录|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|CNAME|5|域名别名|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|SOA|6|权威记录起始|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|PTR|12|域名指针，用于IP解析域名|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|MX|15|邮件交换|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|TXT|16|文本记录|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|AAAA|28|IPV6主机地址|[[RFC3596](http://www.iana.org/go/rfc3596)]|
|SRV|33|服务位置资源记录|[[RFC2782](http://www.iana.org/go/rfc2782)]|
|DS|43|委托签发者，用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3658](http://www.iana.org/go/rfc3658)]|
|RRSIG|46|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|
|NSEC|47|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|
|DNSKEY|48|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|

### QCLASS
两字节长的请求的类型，互联网请求时值为“IN”，对应值为`0x0001`。
这年头应该也见不到其他的CLASS类型。

## Answer，Authority，Additional段
资源记录（RR）区域，即Answers、Authorities、Additional records三个区域，每个区域均包括数量不等的查询块，每块格式如下：
![](attachments/Pasted%20image%2020240104110347.png)
![](attachments/Pasted%20image%2020240104120721.png)

**Answer**，**Authority**和**Additional**三个段的格式是完全相同的，都是由零至多条Resource Record（资源记录）构成。这些资源记录因为不同的用途而被分开存放。

Answer对应查询请求中的Question，Question中的请求查询结果会在Answer中给出，如果一个响应报文的Answer为空，说明这次查询没有**直接**获得结果。

Authority包含的是权威服务器信息，如果某次查询结果里Answer部分为空，那么你可能需要根据Authority内的内容继续发起查询请求

Additional这个名字看起来就很附加，前面三个段放不了的信息往里头无脑塞就完事儿了。这个段会在EDNS中被使用，也可以在这里包含Authority中域名服务器的具体IP地址，这样便可以减少一次查询域名服务器地址的请求。


范例如下所示：
![](attachments/Pasted%20image%2020240104110435.png)
### RR记录格式
资源记录是DNS系统中非常重要的一部分，它拥有一个变长的结构，具体格式如下：
```bash
  0  1  2  3  4  5  6  7  8  9  A  B  C  D  E  F
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                                               |
/                                               /
/                      NAME                     /
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      TYPE                     |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     CLASS                     |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                      TTL                      |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    RDLENGTH                   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--|
/                     RDATA                     /
/                                               /
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
- **NAME**
它指定该条记录对应的是哪个域名，格式使用DNS标准名称表示法。

- **TYPE(2B)**
资源记录的类型；

- **CLASS(2B)**
对应Question的QCLASS，指定请求的类型，常用值为`IN`，值为`0x001`。

- **TTL(Time To Live)(4B)**
资源的有效期，表示你可以将该条RR缓存TLL秒，TTL为0表示该RR不能被缓存(0代表只能被传输，但是不能被缓存)。
TTL是一个4字节有符号数，但是只使用它大于等于0的部分。
例如，如果一条记录的 TTL 为 86400 秒，则对该记录的更改最多需要 24 小时才会生效。
请注意，更改记录的 TTL 会影响之后的所有更改需要多久才会生效。要使后续的更改更快生效（例如，如果您想快速还原一项更改），则可以设置较短的 TTL 值，如 300 秒（5 分钟）。正确配置记录后，我们建议将 TTL 值设置为 86400，即让整个互联网中的服务器每 24 小时检查一次记录的更新情况。


- **RDLENGTH（2B）**
一个两字节非负整数，用于指定RDATA部分的长度（字节数）。

- **RDATA**
是一个长度和结构都可变的字段，它的具体结构取决于TYPE字段指定的资源类型。

### RR记录类型
为了使DNS协议能够应用于更多的场景，DNS协议设计了许多种不同的资源类型来满足不同场景的需求。Type字段在DNS报文中长度为2字节，这意味着最多可以有65536种不同的资源。
FRC1035内指定了编号为1-16的16种资源类型，而当前则一共定义了近100种资源类型，这其中常用的类型并不是很多，大约只有以下十几种。

|类型|值|含义|REFERENCE|
|---|---|---|---|
|A|1|主机地址|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|NS|2|域名服务器记录|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|CNAME|5|域名别名|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|SOA|6|权威记录起始|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|PTR|12|域名指针，用于IP解析域名|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|MX|15|邮件交换|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|TXT|16|文本记录|[[RFC1035](http://www.iana.org/go/rfc1035)]|
|AAAA|28|IPV6主机地址|[[RFC3596](http://www.iana.org/go/rfc3596)]|
|SRV|33|服务位置资源记录|[[RFC2782](http://www.iana.org/go/rfc2782)]|
|DS|43|委托签发者，用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3658](http://www.iana.org/go/rfc3658)]|
|RRSIG|46|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|
|NSEC|47|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|
|DNSKEY|48|用于DNSSEC|[[RFC4034](http://www.iana.org/go/rfc4034)][[RFC3755](http://www.iana.org/go/rfc3755)]|

#### A记录
A就是Address的首字母，它用于解析一个域名对应的IPv4地址，这是生活中最常使用的记录，没有之一。

它长度固定为4字节，直接以二进制形式表示一个IPv4地址。例如IP `1.2.4.8`会被表示为：
```bash
[0x01][0x02][0x04][0x08]
```

范例如下所示：
![](attachments/Pasted%20image%2020240104121039.png)

#### NS
域名服务器记录，用来指定该域名由哪个DNS服务器来进行解析。它的结果是一条新的域名，不过是权威服务器的。
NS的格式和RR里NAME字段的格式相同，它使用DNS标准名称表示法返回一条新域名。

#### CNAME
CNAME是域名别名记录，它用于将多个域名指向同一个主机。它的解析结果通常是另一个域名。
CNAME的格式和上面NS记录格式相同，它使用DNS标准名称表示法返回一条新域名。
CNAME记录经常在设置CDN时被使用。例如**example.com**使用了**cdn.example.net**公司的CDN服务（假设IP=**1.1.1.1**）。你当然可以直接将example.com直接使用A记录解析到1.1.1.1，但是由于CDN不是由你直接管理的，这个ip地址可能会发生变化。如果该公司提供了一个域名cdn.example.net，并且对该域名设置了解析，你就只需要将你的域名example.com通过CNAME解析到cdn.example.net，告诉解析者，example.com是cdn.example.net的别名，IP从他那儿获取即可。

#### SOA
SOA基本上是最复杂的资源记录类型了，没有之一，它的格式如下：
```bash
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
/                     PNAME                     /
/                                               /
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
/                     RNAME                     /
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    SERIAL                     |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    REFRESH                    |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                     RETRY                     |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    EXPIRE                     |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                    MINIMUM                    |
|                                               |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
SOA即Start Of Zone，“区域开始记录”，这个记录主要用于权威DNS服务器进行主从同步。

- **PNAME** ：（Primary Name Server）该区域的主名称服务器
- **RNAME** ：（Responsible authority's mailbox）负责此区域的管理员的电子邮件地址。`@` 用 `.` 来表示，也就是说` admin.example.com` 等同于`admin@example.com`。

- **SERIAL** 区域序列号，32bit无符号整型，从服务器发现这个值更新的时候会启动一次同步。
- **REFRESH** 从服务器从主服务器上获取SOA记录的周期
- **RETRY** 重试时间，当从服务器获取SOA失败时会在该字段指定的时间内重新尝试获取，这个值应当小于REFRESH
- **EXPIRE** 过期时间，如果从主服务器一直未响应，则在该时间后不再响应该区域的请求。这个值应该大于REFRESH+RETRY。
- **MINIMUM** 该区域内所有记录TTL的下限。

所有的时间单位都是秒，除前两个字段使用DNS标准名称表示法以外，其它字段长度都是32bit。SOA记录的TTL都是0，也就是说这个记录不能被缓存。

范例如下：
![](attachments/Pasted%20image%2020240104120828.png)

#### PTR
ptr用于将ip地址反向解析出域名。它和.arpa域名联合使用。例如对114.114.114.114做反向解析可以获得结果：
```bash
;; ANSWER SECTION:
114.114.114.114.in-addr.arpa. 556 IN PTR public1.114dns.com.
```
#### AAAA
AAAA一看就和A有关系，它被用于IPv6地址的解析。AAAA是4个A，而一个IPv6地址的二进制长度也正好是IPv4地址的4倍（IPv6 16Byte，IPv4 4Byte）。 它的格式和A记录一模一样，只不过长度扩展了四倍。因此这个记录的RDATA长度是**固定的16字节**。地址FE80::1的AAAA RDATA部分格式为：
```bash
[0xFE][0x80][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x00][0x01]
```
实际上在早期还有一个叫A6的IPv6地址资源类型，不同点是A6是变长的，但是A6已经被废弃不再使用，IPv6地址全部使用AAAA记录进行解析。
#### MX
邮件交换记录，用于设置电子邮件发送地址。通常会设置多条不同优先级的MX记录。
```bash
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
|                  PREFERENCE                   |
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
/                   EXCHANGE                    /
/                                               /
+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+--+
```
- **PREFERENCE** 是一个16bit数，它表示该条记录的优先级，值越小优先级越高。
- **EXCHANGE** 一个域名，格式为DNS标准名称表示法。
#### TXT
直接就是一段文本，通常会用作域名验证用途。例如TXT记录“abcdefg”格式为：
```bash
['a']['b']['c']['d']['e']['f']['g']
```
## 范例
一次DNS请求的过程：
![](attachments/Pasted%20image%2020240104111102.png)
包括请求和响应，二者具备相同的ID。

**请求**
![](attachments/Pasted%20image%2020240104111219.png)
DNS请求中含有一个被请求的域名：dailyupdate.wangwang.taobao.com。

**响应**
![](attachments/Pasted%20image%2020240104111346.png)
使用Wireshark得到的分析如下：
![](attachments/Pasted%20image%2020240104111358.png)

在这里，就包含了域名部分重复的例子：
![](attachments/Pasted%20image%2020240104111501.png)
图中红框所示即为使用0xC028代替之前出现的`com`段，但是`dailyupdate.wangwang.taobao.com`由于在该`CNAME`的前一部分，则没有被代替。

# 参考
```c
DNS报文格式
https://yangwang.hk/?p=878

# 使用Wireshark进行DNS协议解析
https://mp.weixin.qq.com/s?__biz=MzIxNTExNTcxMg==&mid=2247483877&idx=1&sn=48d797977ec96a3d5117f9da9403ff84&chksm=979c734aa0ebfa5c80147f1dc3f58ed7869c8f5f1b57d249d6c747538b2a65d9459b00a8d560&scene=21#wechat_redirect

## 万字长文爆肝 DNS 协议
https://cloud.tencent.com/developer/article/1779032
```
