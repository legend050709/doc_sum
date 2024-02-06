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

- sr1()方法
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

## 原理
# why
# 范例
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