```table-of-contents
```
# EDNS
## EDNS定义
扩展DNS机制EDNS(Extension Mechanisms for DNS)：在遵循已有的DNS消息格式的基础上增加一些字段，来支持更多的DNS请求业务。
第一组扩展由Internet工程任务组于1999年发布，名称为RFC2671，也称为EDNS0，由RFC6891在2013年进行了更新，将缩写略微更改为EDNS（0）。

需要注意的是，像DNS服务器这样一个大型且广泛应用的系统软件，新增加扩展协议的时候一定要考虑到向后兼容性(backward compatibility)，即你增加了你这个特性的消息传输给未支持该特性的服务器时，后者依然能正确处理。

## 为什么要有 EDNS
RFC2671中指出EDNS被提出来的几个理由：
- 1）DNS协议头部的第二个16bit中都已经被用的差不多了，需要添加新的返回类型(RCODE)和标记(FLAGS)来支持其他需求；
- 2）只为标示domain类型的标签分配了两位，现在已经用掉了两位（00标示字符串类型，11表示压缩类型），后面如果有更多的标签类型则无法支持；
- 3）当初DNS协议中设计的用UDP包传输时包大小限制为512字节，现在很多主机已经具备重组大数据包的能力，所以要有一种机制来允许DNS请求方通知DNS服务器让其返回大包；
- 4) DNSSEC机制和`edns-client-subnet`机制等都需要有EDNS的支持。

## 实现
怎样在DNS消息协议的基础上再增加一些字段呢？为了保持向后兼容性，更改已有的DNS协议格式是不可能的，所以只能在DNS协议的数据部分中做文章。

所以，EDNS中引入了一种新的伪资源记录OPT（pseudo-RR），之所以叫做伪资源记录是因为它不包含任何DNS数据，**OPT RR不能被cache、不能被转发、不能被存储在zone文件中**。
OPT被放在DNS通信双方（`requestor`和`responsor`）DNS消息的`Additional data`区域中。
### OPT伪资源记录中的内容
OPT pseudo-RR中的内容包含固定部分和可变部分。它的结构如下：
![](attachments/Pasted%20image%2020240108160233.png)
图1中最下面的RDATA是可变部分，其余的部分都是固定部分。
> Name字段目前为空；
> TYPE字段是OPT RR的类型编号，IANA为其分配的是41（0x29）.
> RDLEN是可变部分RDATA的长度；
> RDATA是KV类型的可变部分。


#### TTL字段
TTL字段被用来存储扩展消息头部中的RCODE和flags，一共是4B。它的格式如下：
![](attachments/Pasted%20image%2020240108160405.png)

图2中高位8个bit是扩展RCODE（返回状态码），这8个bit加上DNS头部的4bit总共有12bit（8bit在高位），这样就可以表示更多的返回类型；
VERSOION字段表示EDNS的版本（EDNS根据支持不同的扩展内容会有很多版本）。
RFC2671中Z一般情况下被发送者设置为0，接收方可以忽略它。但是后续的扩展协议中会用到这16bit。

#### 可变部分RDATA
OPT RR中可变部分RDATA的结构如下图所示：
![](attachments/Pasted%20image%2020240108160647.png)

OPTION-CODE由IANA分配；OPTION-LENGTH是OPTION-DATA的长度；OPTION-DATA是具体长度。
![](attachments/Pasted%20image%2020240108160708.png)


#### 注意
每个DNS 消息中只能有一个OPT伪资源记录。
当有多种EDNS扩展协议时，各个Option 对一个紧接一个存储在RDATA中。类似于TCP中的Option。
如下图所示：
![](attachments/Pasted%20image%2020240108161114.png)
![](attachments/Pasted%20image%2020240108161118.png)
当有NSID和CSUBNET的时候，两个RDATA紧密排列在OPT的RDATA字段中，它们两的总长度由Data length指定。


### 范例
在自己的机器上用bind-9.8.1-p1中的dig请求Google首页，并把包大小参数设置为768:
```bash
./dig www.google.com.hk +bufsize=768
```

**请求包**：
![](attachments/Pasted%20image%2020240108162608.png)
上面抓包内容中：
1）TTL字段中的extended RCODE、VERSION和Z被ethereal拆分来显示了；
2）RDATA length为0说明没有可变消息RDATA，从下面的消息中可以看到确实没有RDATA。

**响应消息**：
![](attachments/Pasted%20image%2020240108163326.png)
# Client Subnet in DNS Queries
## 背景
DNS系统默认使用明文UDP协议通信，所以用户的查询内容很容易受到监控，而服务器返回的解析结果是可以被轻易篡改。为了解决这个问题，人们引入了 DNS over HTTPS/TLS/QUIC 之类的技术，希望通过加密的方式传输DNS查询。

在DNS解析中，帮我们出去向根递归的是LDNS，如果我们的LDNS和本地用户不在一个地理位置，那么用户则会得到一个LDNS所在位置最近的IP地址，例如我们很多人喜欢吧LDNS设置为8.8.8.8（google的公开DNS，稳定），这样我们智能DNS则会让用户得到一个美国服务器IP这显然是不合理的。

我们日常都使用运营商的递归服务器，它们跟用户的机器地理距离都很近，就不会产生大问题。但如果中国的用户使用了美国的 DNS over HTTPS 服务，那解析出来的可能是美国的IP，会严重影响用户访问。

## 实验
先将LDNS指定google的8.8.8.8。
```bash
$ dig -t A www.alibaba.com @8.8.8.8
 
; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t A www.alibaba.com @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3653
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 0, ADDITIONAL: 1
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;www.alibaba.com.  IN  A
 
;; ANSWER SECTION:
www.alibaba.com.  164  IN  CNAME  www.gds.alibaba.com.
www.gds.alibaba.com.  114  IN  A  198.11.132.23
 
;; Query time: 201 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
;; WHEN: Sun Jul 24 22:04:38 CST 2016
;; MSG SIZE rcvd: 82
```
耗时201ms，解析结果为198.11.132.23，所在地美国，GeoIP: San Mateo, California, United States。

再将LDNS指定为学校的DNSserver
```bash
$ dig -t A www.alibaba.com @202.119.160.11
; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t A www.alibaba.com @202.119.160.11
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 18762
;; flags: qr rd ra; QUERY: 1, ANSWER: 2, AUTHORITY: 2, ADDITIONAL: 1
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
;; QUESTION SECTION:
;www.alibaba.com. IN A
 
;; ANSWER SECTION:zhiwei
www.alibaba.com. 300 IN CNAME www-cn.gds.alibaba.com.
www-cn.gds.alibaba.com. 120 IN A 106.11.62.61
 
;; AUTHORITY SECTION:
gds.alibaba.com. 6460 IN NS gdsns1.alibaba.com.
gds.alibaba.com. 6460 IN NS gdsns2.alibaba.com.
 
;; Query time: 34 msec
;; SERVER: 202.119.160.11#53(202.119.160.11)
;; WHEN: Sun Jul 24 22:08:59 CST 2016
;; MSG SIZE rcvd: 127
```
耗时34ms，解析结果为106.11.62.61，所在地上海，GeoIP: Hangzhou, Zhejiang, China。
## 解决方案
### 方案一：域名白名单
之前并不知道 ECS 协议。那时候只能采用域名白名单来解决这个问题。

简单来说，我们在路由器上配置 dnsmasq 做递归解析。Dnsmasq 支持根据域名前缀设置不同的 DNS 解析服务器，比如下面是一条针对苹果的配置：
```bash
server=/apps.apple.com/114.114.114.114
```
dnsmasq 会从`114.114.114.114`查询域名`apps.apple.com`的DNS记录，所以就能拿到苹果在国内的CDN节点地址，从而实现「加速」效果。

为此，网友维护[dnsmasq-china-list](https://github.com/felixonmars/dnsmasq-china-list)项目，基本把国内域名以及谷歌和苹果的的国内域名都加上去了。
这种白名单只能说是笨办法。

### 方案二：EDNS Client Subnet (ECS)
ECS 简单说就是把用户的IP信息暴露给权威DNS服务器。但为了保护用户隐私，递归服务器并不直接把用户的IP发给权威服务器。相反，只把用户IP所在的网段发给权威DNS服务器。
如果客户使用 IPv4，那发送的网络前缀为 24，如果是 IP6，那网络前缀为56。一般来说，同网段的客户地址位置相近。这里的网段就叫 client subnet。

### 方案三：HTTPDNS
将DNS请求通过HttpDNS接口发起查询。
这样绕过了运营商的LocalDNS，用户解析域名的请求通过Http协议直接透传到了腾讯的HttpDNS服务器IP上，用户在客户端的域名解析请求将不会遭受到运营商解析转发，DNS污染，劫持，出口多NAT等等困扰

## 范例
用200.0.0.1/24作为用户所在的网段：
```bash
$ dig -t A www.zhxfei.com @172.16.130.129 +subnet=200.0.0.1/24
 
; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t A www.zhxfei.com @172.16.130.129 +subnet=200.0.0.1/24
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 11779
;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
;; WARNING: recursion requested but not available
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; CLIENT-SUBNET: 200.0.0.0/24/0
;; QUESTION SECTION:
;www.zhxfei.com. IN A
 
;; ANSWER SECTION:
www.zhxfei.com. 86400 IN A 1.1.1.1
 
;; AUTHORITY SECTION:
zhxfei.com. 86400 IN NS ns.zhxfei.com.
 
;; ADDITIONAL SECTION:
ns.zhxfei.com. 86400 IN A 172.16.130.129
 
;; Query time: 0 msec
;; SERVER: 172.16.130.129#53(172.16.130.129)
;; WHEN: Mon Jul 25 01:00:43 CST 2016
;; MSG SIZE rcvd: 103
```

查询如下所示：
![](attachments/Pasted%20image%2020240108162316.png)

应答如下所示：
![](attachments/Pasted%20image%2020240108162029.png)


用120.0.0.1/24作为用户所在的网段
```bash
$ dig -t A www.zhxfei.com @172.16.130.129 +subnet=120.0.0.1/24
 
; <<>> DiG 9.10.3-P4-Ubuntu <<>> -t A www.zhxfei.com @172.16.130.129 +subnet=120.0.0.1/24
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 21300
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
 
;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 4096
; CLIENT-SUBNET: 120.0.0.0/24/24
;; QUESTION SECTION:
;www.zhxfei.com. IN A
 
;; ANSWER SECTION:
www.zhxfei.com. 86400 IN A 10.0.0.1
 
;; AUTHORITY SECTION:
zhxfei.com. 86400 IN NS ns.zhxfei.com.
 
;; ADDITIONAL SECTION:
ns.zhxfei.com. 86400 IN A 172.16.130.129
 
;; Query time: 0 msec
;; SERVER: 172.16.130.129#53(172.16.130.129)
;; WHEN: Mon Jul 25 01:00:53 CST 2016
;; MSG SIZE rcvd: 103
```

查询如下所示：
![](attachments/Pasted%20image%2020240108162401.png)

应答如下所示：
![](attachments/Pasted%20image%2020240108162406.png)

# DNS支持健康检查
# DNS路由策略（负载均衡策略）
## round-robin 轮询


# 参考
```c

```