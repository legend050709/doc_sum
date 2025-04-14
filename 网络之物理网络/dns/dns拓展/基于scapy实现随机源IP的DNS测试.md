```table-of-contents
```
# 介绍
在DNS系统运维工作中，我们通常会希望能测试不同的源IP下DNS解析结果的应答情况，进而评估智能DNS的实现情况或者做DNS数据分析等。
# scapy测试脚本
本文介绍在`python2.7`环境下使用scapy模块实现一个能随机源IP地址的简单易用的`DNS`发包工具。
```python
#!/usr/bin/env python
#coding=utf-8
import time
import random
from scapy.all import *

dns_server = '192.168.138.146'

#定义DNS域名查询时请求的记录类型，根据DNS协议
query_type_dic = {'A':1,'AAAA':28,'PTR':12,'CNAME':05,'NS':02}
#发包实现函数
def attack(dns_server,client_ip,domain,query_type):
   a = IP(dst=dns_server,src=client_ip)
   b = UDP(dport=53)
   c = DNS(id=1,qr=0,opcode=0,tc=0,rd=1,qdcount=1,ancount=0,nscount=0,arcount=0)
   c.qd = DNSQR(qname=domain,qtype=query_type,qclass=1)
   p = a/b/c
   send(p)

#打开要拨测的域名文件
f = file('domain.txt','r')
list_domain = f.readlines()
f.close()

while True:
   #指定随机源IP，例如IP地址的前16位固定是10.10
   i = random.randrange(0,255,2)
   j = random.randrange(0,255,2)
   attack_ip = '10.10.%s.%s' % (i,j)
   
   attack_domain = list_domain[random.randrange(0,len(list_domain))].split()
   try:
       attack(dns_server,attack_ip,attack_domain[0],query_type_dic[attack_domain[1]])
   except:
       print 'send failed~',attack_domain
   time.sleep(0.2)

```
# 测试方法
用于拨测的域名清单文件`domain.txt`内容如下
```bash
[root@localhost random_source_ip]# cat domain.txt 
www.baidu.com A
www.taobao.com A
163.com MX
www.youku.com AAAA
www.qq.com AAAA
www.jd.com AAAA
8.8.8.8.in-addr.arpa PTR
114.114.114.114.in-addr.arpa PTR

```
# 总结
1. 此工具做功能测试及验证比较适合，但不适合性能测试。
2. 分享的目的是DNS运维的工作的经验，不是为了拿工具来攻击DNS系统。强烈反对这样的行为。
# 参考
```bash
# 基于scapy实现随机源IP的DNS发包工具
https://blog.csdn.net/u011288801/article/details/106663563
```