```table-of-contents
```
# 基本概念
## publicDns

![](attachments/Pasted%20image%2020240626171258.png)

公共解析 PublicDNS 向用户提供 DNS 服务器的地址。用户可以将设备的 DNS 服务器地址设置为该地址。公共解析 PublicDNS 可以提升用户的互联网访问速度，还可以帮助用户避免 DNS 欺骗、DNS 劫持等问题。


# DNS的安全缺陷
默认情况下，DNS 查询和响应以明文形式（通过 UDP）发送，这意味着它们可以被网络、ISP 或任何能够监视传输的人读取。即使网站使用 HTTPS，也会显示导航到该网站所需的 DNS 查询。

设计 DNS 的时候，互联网基本上还是个玩具。那年头的互联网协议，压根儿都没考虑安全性，DNS 当然也不例外。所以 DNS 的交互过程全都是【明文】滴，既无法做到“保密性”，也无法实现“完整性”。  



## 传统DNS(Udp 53)

传统的 UDP 53 协议虽然实现简单且具有广泛的适用性，但其存在诸多问题，如容易被劫持和篡改，以及易受分布式拒绝服务（DDoS）攻击等。因此，传统 DNS 已无法满足现代网络安全和隐私的需求。

![](attachments/Pasted%20image%2020240626172309.png)


**缺乏“保密性”**
意味着 — — 任何一个能【监视】你上网流量的人，都可以【看到】你查询了哪些域名。直接引发的问题就是隐私风险。  

**缺乏“完整性”**
意味着 — — 任何一个能【修改】你上网流量的人，都可以【篡改】你的查询结果。直接引发的问题就是“DNS 欺骗”（也叫“DNS 污染”或“DNS 缓存投毒”）

## 常见DNS攻击
常见的DNS攻击包括：DNS劫持、缓存投毒、DNS欺骗等等，目的就是通过各种攻击手段将正常访问合法网站的用户，引到黑客控制的假冒服务器上，进行钓鱼欺诈、窃取用户凭证或敏感数据等非法行为。

# DNSSEC
## 背景
普通的 DNS 请求基于面向无状态的 UDP 协议，并且不会对响应结果进行检查，这就给了攻击者们可乘之机：最常见的 DNS 污染就是对工作在 UDP 53 端口上的流量进行 IDS 入侵检测，并直接返回一个虚假的地址。

更严重的情况是，由于 DNS Cache 的存在，错误的记录可能在很长一段时间内使得用户无法访问到真正的服务器。

这种情况下，有必要以某种方式对返回结果进行 Validation，于是就有了 DNSSEC 技术。

## 介绍
DNSSEC（域名系统安全扩展：Domain Name System Security Extensions）诞生于1997年，已经列入互联网标准化文档（参考RFC 4033、RFC 4034、RFC 4035），是最早大规模部署的DNS安全协议。

DNSSEC 允许域名所有者对DNS记录进行数字签名，依靠**数字签名**来保证**DNS应答报文的真实性和完整性**。通过DNSSEC机制，DNS Resolver或者终端用户可以确保DNS查询到的结果是否可信，即是否为权威域名服务器返回。
>简单来说，权威服务器使用私钥对资源记录进行签名，递归服务器利用权威服务器的公钥对应答报文进行验证。如果验证失败，则说明这一报文可能是有问题的。

在2010年的时候，所有根域名服务器]都已经部署了 DNSSEC。到了2011年，若干顶级域名（.org 和 .com 和 .net 和 .edu）也部署了 DNSSEC。

## 概念
### RRSets
RRSet 是 Resource Record Set 的简称，在这个资源集里面包含了在该域下所有的相同类型记录资源；举个例子，init.blog 域下面有三条 A 类型的记录，分别为：
1. a.inig.blog 300 IN A 1.2.3.4
2. b.init.blog 600 IN A 3.4.5.6
3. c.init.blog 1200 IN A 1.0.0.1
这三条记录就组成了一个 RRSet。

### Zone-Singing Key
在 DNSSEC 中，每一个域都有一对公私钥对被称为 Zone-Singing Key (ZSK)。 其中的私钥用来签名上面我们提到的 RRSets。

在开启了DNSSEC后，权威服务器负责使用私钥对RRset（资源记录）进行签名，并将结果放入RRSIG记录中，同时将签名对应的公钥放在DNSKEY中，以便于DNS 递归服务器进行校验。

#### RRSIG资源记录
**RRSIG(RRset的数字签名)**
获得的数字签名将被存储到 RRSIG（resource record signature） 类型的 DNS 记录中。
RRSIG资源记录存储的是对资源记录集合（RRSets）的数字签名。如下：
![](../attachments/Pasted%20image%2020240108144615.png)

#### DNSKEY资源记录
**DNSKEY(RR签名的公钥)**
那么如何验证呢？
ZSK 的公钥将被存储在一个叫做 DNSKEY 的新类型 DNS 记录中。当 DNS Resolver 查询到包含 RRSIG 的记录时，就可以访问 Authorized NS（权威服务器） 的 DNSKEY 记录，从中获取 ZSK 的公钥，用以验证签名。

DNSKEY资源记录存储的是公开密钥，下面是一个DNSKEY的资源记录的例子：
![](../attachments/Pasted%20image%2020240108144747.png)

下面是当我查询 init.blog 域的 MX 记录时给出的响应，可以看到，相同类型的 MX 资源组成的 RRSet 被签名，并随响应一同返回了 RRSIG：
![](../attachments/Pasted%20image%2020240108120104.png)

### Key-Singing Key
在实践中，权威域的管理员通常用两个密钥配合完成对区数据的签名。一个是Zone-Signing Key(ZSK)，另一个是Key-Signing Key(KSK)。**ZSK用于签名区数据，而KSK用于对ZSK进行签名。**

**如何验证返回 DNSKEY 记录是否被篡改了呢？**
仅仅通过上述机制，只能部分证明RRset是否完整（功能等同于哈希），因为公钥和签名都来自不可信地方，攻击者可以通过篡改`DNSKEY`和`RRSIG`绕过校验。基于这种最基本的校验逻辑，DNSSEC进一步使用DS和`Key-Signing Key`机制，构建了DNSSEC的信任链。

为了防止ZSK被修改，DNSSEC还使用了一对叫做**Key-Signing Key**（KSK）的公私钥对。KSK 的私钥被用来对在 DNSKEY 记录中的 ZSK 公钥进行签名，并单独为 DNSKEY 记录创建一个 RRSIG。在DNSKEY记录中，会同时包含ZSK和KSK。所以，在一个zone中，除DNSKEY记录外，其他的记录均由ZSK签名。

下面是一个完整的 DNSKEY 查询响应，可以看到包含了 KSK 以及 ZSK：
![](../attachments/Pasted%20image%2020240108142612.png)

### DS 记录 和 DNSSEC信任链
如果攻击者完全伪造了一套 KSK 与 ZSK，那我们的验证手段就依然失效了吗？

这里 DNSSEC 引入了最后一个概念，**DS** 记录（Delegation Signer）：
DS（Delegation Signer）记录存储DNSKEY的hash值，用于验证DNSKEY的真实性，从而建立一个信任链。
DNSKEY存储在资源记录所有者所在的权威域的区文件中，但是DS记录存储在上级权威域名服务器中，比如`paypal.com`的`DS RR`存储在`.com`的区中。
![](../attachments/Pasted%20image%2020240108145942.png)
通过每一级的 DS 记录，就可以对下一级 DNSKEY 的 RRSIG 进行验证，这样就可以构成一条信任链。


### 为什么使用 ZSK 以及 KSK
有细心的朋友可能会发现，在这个过程中，其实完全没有必要使用一套单独的 KSK；在这种情况下，我们只需要对 DNSKEY 也使用 ZSK 签名生成 RRSIG，并将 ZSK 的 Hash 作为 DS 存储即可。

ZSK用于签名区数据，而KSK用于对ZSK进行签名。这样做的好处有二：
（1）用KSK密钥签名的数据量很少，被破解（即找出对应的私钥）概率很小，因此可以设置很长的生存期。这个密钥的散列值作为DS记录存储在上一级域名服务器中而且需要上级的数字签名，较长的生命周期可以减少密钥更新的工作量。

（2）ZSK签名的数据量比较大，因而破解的概率较大，生存期应该小一些。因为有了KSK的存在，ZSK可以不必放到上一级的域名服务中，更新ZSK不会带来太大的管理开销（不涉及和上级域名服务器打交道）。



## DNSSEC的原理
![](../attachments/Pasted%20image%2020240108150041.png)

### 范例
我们自上而下的重新梳理一遍 DNSSEC 的工作流程，假设我们需要获取 init.blog 的 A 记录地址，并且该域已经启用了 DNSSEC：
1. DNS Resolver 请求 init.blog 域的 A 记录。得到结果 IP 地址 76.223.126.88，并且同时得到了 init.blog 域中 A 记录 RRSet 的签名 RRSIG。该签名可以使用 ZSK 验证。
2. DNS Resolver 请求 init.blog 域的 DNSKEY 记录。得到了该域的 ZSK 和 KSK 公钥，并同时得到了 DNSKEY RRSet 的签名 RRSIG。该签名可以使用 KSK 验证。
3. DNS Resolver 使用得到的 KSK 验证了上一步得到的 DNSKEY 记录。没有发现问题。
4. DNS Resolver 使用得到的 ZSK 验证了第一步得到的 A 记录。没有发现问题。
5. 为了保证 DNSKEY RRSIG 中的 KSK 不被伪造，DNS Resolver 请求了 .blog 域与 init.blog 相关的 DS 记录，并且得到了 DS 记录的 RRSIG。通过计算 KSK 的 Hash 值，没有发现问题。
6. 对 .blog 域 DS 记录的 RRSIG 重复上面 2-5 步的过程，最终通过 . 根域的 DS 验证。

注：在实际的递归查询过程中，该过程是自顶向下的，这里为了方便理解，我将整个过程倒过来叙述。

当我们从根域名开始查询 init.blog 的 A 记录响应时，就可以发现除根域、本域之外的任意父域都包含了子域的 DS Record，这样就可以形成一个信任链：
![](../attachments/Pasted%20image%2020240108143934.png)

## DNSSEC的优缺点
### 优点
DNSSEC通过数字签名实现了一种认证机制，确保了DNS Resolver有方法校验查询到的DNS记录是否来自可信来源，对于中间人攻击有一定的效果。

### 缺点
#### 缺乏机密性
DNSSEC协议仅提供真实性和完整性的校验，无法确保DNS流量通信的机密性。
现有的 DNSSEC 也仅仅是对查询结果的 Validation —— 也就是验证，而对于整个路由路径上的任何一个节点，你的查询都是透明的，没有 Encryption。

#### DNSSEC与防域名劫持
dnssec并没有办法在域名劫持上起到很好的作用，如果发生域名劫持则无法得到真正的解析结果，因为数字签名校验是没有校验通过的。

#### DNSSEC可能导致解析失败
响应中也有RRSIG记录，会直接导致UDP包的大小超过512字节，那么可能造成部分localdns解析失败，因为根据之前对于线上的观察，部分localdns并不支持超过512字节大小的UDP包，从而可能直接导致响应失败。

#### 现状
**DNSSEC的普及度并不高**
国内大部分的 DNS Resolver 还不支持 DNSSEC，对于DNS Resolver而言必须做兼容处理，而攻击者完全可以对DNSSEC相关的数据继续剥离，欺骗DNS Resolver目标没有开启DNSSEC，在国内这种情况十分常见。

**DNSSEC 并不是一个保证 Client 到 Resolver 之间通讯安全性的协议**
在国内的网络用户如果使用国外的 DNS Resolver，其收到的解析结果也有很大几率被污染，因为DNSSEC 并不是一个保证 Client 到 Resolver 之间通讯安全性的协议；即使使用国内某些支持 DNSSEC 的解析器，其与国外 NS 的通讯过程也会被干扰，导致用户依然无法查询到解析结果。

# DoT
## 介绍
DNS over TLS（简称DoT）是一项域名解析安全扩展协议，它使用TLS协议加密传输用户和递归解析服务器之间的DNS消息，起到防止中间用户窃听和域名查询隐私泄漏的作用。


## DoT的流程

![](attachments/Pasted%20image%2020240626172440.png)


TLS 传输的过程，如下：
- TCP 三次握手
- SSL 的 ClientHello 和 ServerHello 和对应的秘钥交换 KeyExchange
- Client和 Server互相 ChangeCipherSpec 通知进入加密模式，此时可以进入数据传输状态
- 应用数据传输过程
- 应用数据传输完成，TCP 两次挥手

抛开 TCP 连接和数据包文传输的部分，TLS 握手部分将使用 2 个 RTT。
DNS-over-TLS 使用了 TCP 853 作为传输端口来完成 TLS 握手，再执行普通的 DNS 请求/应答。因此在 DNS-over-TLS 的整个过程中，将使用至少 4 次 RTT，这也将导致 DNS 的查询延时放大 4 倍。

## 优缺点

![](attachments/Pasted%20image%2020240626172533.png)

# DoH
## 介绍
“DNS over HTTPS”有时也被简称为【DoH】。即使用安全的 HTTPS 协议运行 DNS ，主要目的是增强用户的安全性和隐私性。

顾名思义，DNS over HTTPS 就是基于 HTTPS 隧道之上的域名协议。而 HTTPS 又是“HTTP over TLS”。所以 DoH 相当于是【双重隧道】的协议。  
与 DoT（DNS Over TLS） 类似，DoH 最终也是依靠 TLS 来实现了【保密性】与【完整性】。 TLS 本身已经实现了保密性与完整性，因此，DoH 和 DoT都具有 保密性与完整性。

协议栈如下所示：
```bash
--------  
DoH  
--------  
HTTP  
--------  
TLS  
--------  
TCP  
--------  
IP  
--------
```

## 流程

DoH（DNS-over-HTTPS）协议的标准文件为 RFC 8484；

![](attachments/Pasted%20image%2020240626172845.png)

这就是整个 DoH 请求的过程，客户端还需要自行解密 DNS 查询结果；然而 DoH 首次请求（在 keep-alive 保持连接后速度就很不错了），流程就是这么多，所以导致 DoH 的资源开销是目前加密技术最大的。

## 优缺点

![](attachments/Pasted%20image%2020240626173042.png)

**优点**
基本上，DoT 具备的优点，DoH 也具备。  
相比 DoT，DoH 还多了一个优点：  
由于 DoH 是基于 HTTP 之上。而主流的编程语言都有成熟的 HTTP 协议封装库；再加上 HTTP 协议的使用本身很简单。因此，要想用各种主流编程语言开发一个 DoH 的客户端，是非常容易滴。

**缺点**
DoH 目前还只有 RFC 的草案，尚未正式发布。
相比 DoT，DoH 还有一个小缺点 — — 由于 DoH 比 DoT 多了一层（请对比两者的协议栈），所以在性能方面，DoH 会比 DoT 略差。为啥说这是个【小】缺点捏？因为域名的查询并【不】频繁，而且客户端软件可以很容易地对域名的查询结果进行【缓存】（以降低查询次数）。所以 DoH 比 DoT 性能略差，无伤大雅。




## DoT 和 DoH 对比

近几年对DNS解析防劫持的要求越来越高， 关于dns加密查询，主要分为DOT， DOH 两种方式，含义如下：
```bash
DOT: DNS over TLS
DOH: DNS over HTTPS
```

**两者的目的一致，都是为了加密DNS的请求内容，防止伪造、劫持攻击**。
区别在于 HTTPS是TLS上的HTTP协议， 更通用些。


DoH默认端口是443，即HTTPS的默认端口（DNS over TLS有自己的端口853）

DoT 因为协议栈少了一层，性能会比 DoH 更好。但是前面也说了，域名查询的频度是比较低的，而且还可以利用客户端软件的 DNS 缓存，进一步减少域名查询的频度。所以 DoT 虽然性能更好，但优势不明显。

DoH 的强项体现在如下几方面：  
**1. 编程接口更简单**  
这是个很重要的优势 — — 有助于让更多软件切换到 DoH 之上。  

**2. 可以利用 HTTP 协议已有的特性**  
由于 DoH 是基于 HTTPS 之上，可以无缝地支持 Proxy；  
DoH 可以充分利用 HTTP 2.0 的特性（HTTP/2 在 HTTP/1.1 基础上加了很多功能）。
正是因为 DoH 的这些优势，浏览器厂商对 DoH 的支持更积极。对比一下就可以看出来 — — DoT 在两年前正式发布 RFC，主流的浏览器没一个支持；而 DoH 目前才仅仅是 RFC 草案，Firefox 与 Chrome/Chromium 都开始支持了。

# H3
## 介绍
H3（HTTP/3）协议的标准文件是 RFC 9114， H3 基于 QUIC 协议实现，使用 UDP 传输层。

![](attachments/Pasted%20image%2020240626173433.png)

## 优缺点

![](attachments/Pasted%20image%2020240626173450.png)


# DoQ
## 介绍
DNS over  QUIC （DoQ） 专用QUIC连接上的DNS；它的标准文件是 RFC 9250，基于 QUIC 协议实现，使用 UDP 传输层。DoQ 结合了 QUIC 协议的高效性能和 DNS 查询的加密保护。

QUIC 提供的加密与 TLS 提供的加密具有相似的属性，而 QUIC 传输消除了 TCP 固有的线头阻塞问题，并提供比 UDP 更有效的丢包恢复。DoQ 实现了隐私属性，也解决了 延迟大的问题。

DoQ 连接不得使用 UDP 端口 53。反对将端口 53 用于 DoQ 的建议是为了避免混淆 DoQ 和使用 DNS over UDP （传统DNS，RFC1035）。 在递归场景中，使用端口 443 作为双方同意的替代端口在操作上可能是有益的，因为与其他端口相比，端口 443 不太可能被阻塞。

## 流程

![](attachments/Pasted%20image%2020240626173935.png)

## 优缺点

![](attachments/Pasted%20image%2020240626174027.png)


## 对比

![](attachments/Pasted%20image%2020240626174223.png)



# 参考
```c

# 保护隐私与优化网络：深入比较 DoT、DoH、H3 和 DoQ 协议的功能与优势
https://www.timochan.cn/posts/study/protecting_privacy_and_optimizing_networks#DoH

# 阿里DNS：浅析DNS-over-TLS
https://zhuanlan.zhihu.com/p/47170371


# DNSSEC 技术详解
https://init.blog/dnssec/

## DNSSEC学习笔记
https://abcdxyzk.github.io/blog/2023/01/25/dns-sec-2/

# PublicDNS服务提供商增加字节，将支持 DoH/DoT/DoQ 等协议
https://blog.csdn.net/weixin_37813152/article/details/132322595

```