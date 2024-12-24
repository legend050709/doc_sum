```table-of-contents
```

# 介绍
# 安装
```bash
yum install bind-utils
```
# 使用方法
dig 命令的使用方法如下：
```bash
dig [@server] [name] [type]

- [@server]查询指向的主机名或 IP 地址
- [name]要查询服务器的 DNS（域名服务器）
- [type]要检索的 DNS 记录类型。默认情况下（或如果留空），dig 查询 A 记录。
```

```c
# dig -h
Usage:  dig [@global-server] [domain] [q-type] [q-class] {q-opt}
            {global-d-opt} host [@local-server] {local-d-opt}
            [ host [@local-server] {local-d-opt} [...]]
Where:  domain	  is in the Domain Name System
        q-class  is one of (in,hs,ch,...) [default: in]
        q-type   is one of (a,any,mx,ns,soa,hinfo,axfr,txt,...) [default:a]
                 (Use ixfr=version for type ixfr)
        q-opt    is one of:
                 -4                  (use IPv4 query transport only)
                 -6                  (use IPv6 query transport only)
                 -b address[#port]   (bind to source address/port)
                 -c class            (specify query class)
                 -f filename         (batch mode)
                 -i                  (use IP6.INT for IPv6 reverse lookups)
                 -k keyfile          (specify tsig key file)
                 -m                  (enable memory usage debugging)
                 -p port             (specify port number)
                 -q name             (specify query name)
                 -t type             (specify query type)
                 -u                  (display times in usec instead of msec)
                 -x dot-notation     (shortcut for reverse lookups)
                 -y [hmac:]name:key  (specify named base64 tsig key)
        d-opt    is of the form +keyword[=value], where keyword is:
                 +[no]aaflag         (Set AA flag in query (+[no]aaflag))
                 +[no]aaonly         (Set AA flag in query (+[no]aaflag))
                 +[no]additional     (Control display of additional section)
                 +[no]adflag         (Set AD flag in query (default on))
                 +[no]all            (Set or clear all display flags)
                 +[no]answer         (Control display of answer section)
                 +[no]authority      (Control display of authority section)
                 +[no]badcookie      (Retry BADCOOKIE responses)
                 +[no]besteffort     (Try to parse even illegal messages)
                 +bufsize=###        (Set EDNS0 Max UDP packet size)
                 +[no]cdflag         (Set checking disabled flag in query)
                 +[no]class          (Control display of class in records)
                 +[no]cmd            (Control display of command line)
                 +[no]comments       (Control display of comment lines)
                 +[no]cookie         (Add a COOKIE option to the request)
                 +[no]crypto         (Control display of cryptographic fields in records)
                 +[no]defname        (Use search list (+[no]search))
                 +[no]dnssec         (Request DNSSEC records)
                 +domain=###         (Set default domainname)
                 +[no]dscp[=###]     (Set the DSCP value to ### [0..63])
                 +[no]edns[=###]     (Set EDNS version) [0]
                 +ednsflags=###      (Set EDNS flag bits)
                 +[no]ednsnegotiation (Set EDNS version negotiation)
                 +ednsopt=###[:value] (Send specified EDNS option)
                 +noednsopt          (Clear list of +ednsopt options)
                 +[no]expire         (Request time to expire)
                 +[no]fail           (Don't try next server on SERVFAIL)
                 +[no]header-only    (Send query without a question section)
                 +[no]identify       (ID responders in short answers)
                 +[no]idnin          (Parse IDN names)
                 +[no]idnout         (Convert IDN response)
                 +[no]ignore         (Don't revert to TCP for TC responses.)
                 +[no]keepopen       (Keep the TCP socket open between queries)
                 +[no]mapped         (Allow mapped IPv4 over IPv6)
                 +[no]multiline      (Print records in an expanded format)
                 +ndots=###          (Set search NDOTS value)
                 +[no]nsid           (Request Name Server ID)
                 +[no]nssearch       (Search all authoritative nameservers)
                 +[no]onesoa         (AXFR prints only one soa record)
                 +[no]opcode=###     (Set the opcode of the request)
                 +[no]qr             (Print question before sending)
                 +[no]question       (Control display of question section)
                 +[no]rdflag         (Recursive mode (+[no]recurse))
                 +[no]recurse        (Recursive mode (+[no]rdflag))
                 +retry=###          (Set number of UDP retries) [2]
                 +[no]rrcomments     (Control display of per-record comments)
                 +[no]search         (Set whether to use searchlist)
                 +[no]short          (Display nothing except short
                                      form of answer)
                 +[no]showsearch     (Search with intermediate results)
                 +[no]sigchase       (Chase DNSSEC signatures)
                 +[no]split=##       (Split hex/base64 fields into chunks)
                 +[no]stats          (Control display of statistics)
                 +subnet=addr        (Set edns-client-subnet option)
                 +[no]tcp            (TCP mode (+[no]vc))
                 +timeout=###        (Set query timeout) [5]
                 +[no]topdown        (Do +sigchase in top-down mode)
                 +[no]trace          (Trace delegation down from root [+dnssec])
                 +trusted-key=####   (Trusted Key to use with +sigchase)
                 +tries=###          (Set number of UDP attempts) [3]
                 +[no]ttlid          (Control display of ttls in records)
                 +[no]ttlunits       (Display TTLs in human-readable units)
                 +[no]unknownformat  (Print RDATA in RFC 3597 "unknown" format)
                 +[no]vc             (TCP mode (+[no]tcp))
                 +[no]zflag          (Set Z flag in query)
        global d-opts and servers (before host name) affect all queries.
        local d-opts and servers (after host name) affect only that lookup.
        -h                           (print help and exit)
        -v                           (print version and exit)
```
# dig理解
## `dig +trace`
`dig +trace [hostname]`命令返回的内容是DNS解析过程中所有经过的DNS服务器和域名路径，每个DNS服务器的IP地址、DNS记录类型、TTL（生存时间）、DNS记录值等信息都会被列出来。即 从根服务器开始追踪一个域名的解析过程。
### `dig +trace www.baidu.com`流程解析

### 其他
![](../attachments/Pasted%20image%2020240116203202.png)
# 常见方法
## dig命名查询的内容解析
![](../attachments/Pasted%20image%2020231108103907.png)

例如，解析longshuai.com域中的主机www的A记录。
```bash
[root@xuexi ~]# dig www.longshuai.com  A
; <<>> DiG 9.9.4-RedHat-9.9.4-50.el7_3.1 <<>> -t a www.longshuai.com  
;; global options: +cmd  
;; Got answer:  
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 8670  
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2  
  
;; OPT PSEUDOSECTION:  
; EDNS: version: 0, flags:; udp: 4096  
;; QUESTION SECTION:  
;www.longshuai.com.             IN      A  
  
;; ANSWER SECTION:  
www.longshuai.com.      21600   IN      A       172.16.10.16  
  
;; AUTHORITY SECTION:  
longshuai.com.          21600   IN      NS      dnsserver.longshuai.com.  
  
;; ADDITIONAL SECTION:  
dnsserver.longshuai.com. 21600  IN      A       172.16.10.15  
  
;; Query time: 0 msec  
;; SERVER: 172.16.10.15#53(172.16.10.15)  
;; WHEN: Sat Aug 12 23:38:17 CST 2017  
;; MSG SIZE  rcvd: 102
```

在结果中：
**第一行**
![](../attachments/Pasted%20image%2020240116204408.png)
dig的版本以及查询的目标。
如果不希望在输出中包括这些行，请使用+nocmd参数。 （此参数必须是 dig 命令后的第一个参数。）比如：
```bash
dig +nocmd www.baidu.com 
而不是 
dig www.baidu.com +nocmd
```

**HEADER部分**
`HEADER`部分显示从被请求机构（DNS 服务器）收到响应的详细技术信息。
标题显示由 dig 执行操作的「操作码」和「操作状态」的「标头」，上述示例中的「操作状态」是NOERROR，这意味着被请求的 DNS 服务器可以没有任何阻碍地提供查询。
![](../attachments/Pasted%20image%2020240116204417.png)
> 可以用+comments参数隐藏本部分输出，使用此参数时还会禁用一些其它部分输出的标题。

**OPT PSEUDOSECTION部分**
`OPT PSEUDOSECTION`显示高级数据，仅在较新版本的 dig 工具中显示；您可以阅读更多[关于 DNS（EDNS）的扩展机制](https://en.wikipedia.org/wiki/Extension_Mechanisms_for_DNS)。

- EDNSDNS 的扩展机制
- Flags未指定标志时为空
- UDPUDP 数据包大小

![](../attachments/Pasted%20image%2020240116204733.png)
> 要隐藏此部分输出可以使用+noedns参数。

**QUESTION SECTION**
显示发送的查询数据：
- 第一列是查询的域名
- 第二列是查询的类型（IN 表示互联网）
- 第三列指定了记录类型（如果未指定则默查询 A 记录）

![](../attachments/Pasted%20image%2020240116204827.png)
> 可以使用+noquestion参数禁用此部分输出。

**ANSWER SECTION**
表示对查询的回复。回复的结果是”www.longshuai.com."的A记录值为"172.16.10.16"，这正是dig所期望的结果。

**AUTHORITY SECTION**
表示该查询是权威服务器给的答案，并给出了权威服务器的ns记录。在此例中，”www.longshuai.com"主机所在的域"longshuai.com"的权威服务器为"dnsserver.longshuai.com."。**如果没有该段，表示非权威应答**。

**”ADDITIONAL SECTION**
是额外的回复，回复的内容是权威服务器的A记录。

**STATISTICS统计信息部分**
`STATISTICS`统计信息部分显示关于查询的元数据:
- `Query time`响应花费的时间
- `SERVER`响应 DNS 服务器的 IP 地址和端口
- `WHEN`命令运行的时间戳
- `MSG SIZE rcvd`从 DNS 服务器收到的回复大小

![](../attachments/Pasted%20image%2020240116205004.png)
> 可以使用+nostats参数禁用此部分输出。

**其他**
还可以测试ns记录、soa记录、cname记录等。
不过需要注意的是soa记录和ns记录查询的对象是域名而不是主机名，而CNAME记录的对象则必须是主机名。
```bash
[root@xuexi ~]# dig -t ns longshuai.com  
[root@xuexi ~]# dig -t soa longshuai.com  
[root@xuexi ~]# dig -t cname www1.longshuai.com
```
### 查询状态
查询状态分为：
**NOERROR：** 代表没有错误；
**NXDOMAIN：** 否定回答，不存在此记录；
**REFUSED：** DNS服务器拒绝访问；
**SERVFAIL：** dns查询记录失败

### 查询标记
qr： `query` 查询标志，代表查询操作
rd： `recursion desired`，代表希望进行递归查询操作
ra： `recursion available` 在返回中设置，代表查询的服务器支持递归查询操作
aa： `authoritative answer` 权威回复，如果查询结果是由管理域名的域名服务器而不是缓存服务器提供的，则称为权威回复

### 查询类型
查询类型分为：
**A记录：**IPV4的地址解析；
**AAA记录：**IPV6的地址解析；
**NS记录：**域名服务器的记录；
**MX记录：** 邮件交换记录；
**PTR记录（指针记录）：**A记录的逆向记录，作用是把IP地址解析为域名；
**CNAME记录：** 别名记录；


# 常用范例
## 指定DNS服务器查询
默认情况下，如果未指定名称服务器，dig 命令会使用`/etc/resolv.conf`文件中列出的服务器进行查询。
要指定 DNS 服务器进行查询，可以使用@后跟 DNS 服务器 IP 地址的方式来手动指定 DNS 服务器。
```bash
dig @Server_ip NS
```

##  `+nsid`返回响应此次查询的server的信息
如果多个`DNS server`是通过 `anycast IP` + `BGP` 方式部署为集群，那么`client`发起DNS请求，发现请求出现问题，也不知道该DNS请求被哪个服务器所处理以及相应。
通过添加 `+nsid` 在dns响应中，可以获取到 `dns server`的信息。
```
dig 域名 +[no]nsid

是否请求名称服务器 (NS) ID。
```

范例如下所示：
![](../attachments/Pasted%20image%2020240115193615.png)

报文解析如下所示：
请求报文：
![](../attachments/Pasted%20image%2020240115193703.png)
响应报文：
![](../attachments/Pasted%20image%2020240115193743.png)


### 应用场景
集群部署的DNS，使用相同的anycast IP. Client请求存在超时或者失败，不确定是哪个DNS服务器的问题。通过添加 `+nsid` 可以看到那个server的响应存在问题。

```bash
for i in `seq 1 100`; do dig xxxxx +nsid; done

比如：
for a in `seq 1 3`; do echo ------${a}----; dig www.baidu.com +nsid | grep -e NSID -e "ANSWER SECTION" -A 3; done
```

## 设置查询的超时时间和重试次数
```bash
dig +time=5 +tries=1 @10.0.0.10 cdnxxx.com

time dig +time=5 +tries=1 @10.0.0.10 cdnxxx.com
```


## `dig -x`反解查询
```bash
dig -x IP

注：不要写为： dig -t PTR IP 这种形式。
```

## dig -x IP 和  dig -t PTR IP的区别

dig -x IP 实际发出的请求 为 逆序的IP + in-addr.arpa。
比如：`-x 172.16.0.159` 实际发出的请求name为：`159.0.16.172.in-addr.arpa.`。
但是 -t  PTR IP 发出的请求就是 IP。
比如：`-t PTR 172.16.0.159` 发出的 请求的 name 就是 `172.16.0.159`。
 
![](attachments/Pasted%20image%2020240511163202.png)


实际查询的日志如下所示：

![](attachments/Pasted%20image%2020240511163734.png)

查看具体的 zone 文件，如下所示：

![](attachments/Pasted%20image%2020240511163906.png)


## `+tcp`使用TCP协议进行查询
默认情况下dig将采用udp协议进行查询，除非是 AXFR 或 IXFR 请求，才使用 TCP 连接。如果要采用tcp方式进行查询，可以加上 `+tcp`参数。

```bash
dig www.baidu.com +tcp
```
**查询报文如下**：
![](../attachments/Pasted%20image%2020240112145330.png)

**响应报文如下**：
![](../attachments/Pasted%20image%2020240112145420.png)
## `+norecursive`非递归查询
`+[no]recursive`：查询中的 RD（要求递归）位设置。在缺省情况下设置该位，也就是说 dig 正常情形下发送递归查询。当使用查询选项 `+nssearch` 或 `+trace` 时，递归自动禁用。
## `+trace`从根域开始逐级追踪解析过程
```bash
# dig www.baidu.com +trace
```

范例：
```bash
# dig www.sysgeek.cn +trace

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.8 <<>> www.sysgeek.cn +trace
;; global options: +cmd
.			78	IN	NS	i.root-servers.net.
.			78	IN	NS	b.root-servers.net.
.			78	IN	NS	k.root-servers.net.
.			78	IN	NS	c.root-servers.net.
.			78	IN	NS	f.root-servers.net.
.			78	IN	NS	h.root-servers.net.
.			78	IN	NS	g.root-servers.net.
.			78	IN	NS	e.root-servers.net.
.			78	IN	NS	a.root-servers.net.
.			78	IN	NS	j.root-servers.net.
.			78	IN	NS	m.root-servers.net.
.			78	IN	NS	l.root-servers.net.
.			78	IN	NS	d.root-servers.net.
;; Received 239 bytes from 10.6.6.6#53(10.6.6.6) in 0 ms

cn.			172800	IN	NS	a.dns.cn.
cn.			172800	IN	NS	b.dns.cn.
cn.			172800	IN	NS	c.dns.cn.
cn.			172800	IN	NS	d.dns.cn.
cn.			172800	IN	NS	e.dns.cn.
cn.			172800	IN	NS	ns.cernet.net.
cn.			86400	IN	DS	57724 8 2 5D0423633EB24A499BE78AA22D1C0C9BA36218FF49FD95A4CDF1A4AD 97C67044
cn.			86400	IN	RRSIG	DS 8 1 86400 20240129050000 20240116040000 30903 . HoiYK2CDR0/zr7pA6IGebfu61sgADWRPYDhrtG+ZsBr9FimFhYq3k6Z4 I3kWK2ucKaQoOHjjBbOiIZu8Ulw5YDPywI13k6Wx1xJ4PQCSid/dnTku zl5/vvrpUFFECvnmHcZgfkkIuFoMgb9nv23rj/ChhawnsXk+mkUHw1NG ENZN5bwHuq2VyNTNbIScctNzmeNxlQ+MUJiPX4qD4raY8IW4nNArS5IM QvHTVDWb1Pp6HizasstwBwA+G8z0dJUxiO1bCSnLpCG7kSs3IW9viY3F 0LZlZKYoHgv3aybrcUI4ez3PYRILzlEkz2CAdlcF3Ff/tNZrQPRx0nij 4kvkIg==
;; Received 725 bytes from 193.0.14.129#53(k.root-servers.net) in 2 ms

sysgeek.cn.		86400	IN	NS	ns1.huaweicloud-dns.cn.
sysgeek.cn.		86400	IN	NS	ns1.huaweicloud-dns.com.
sysgeek.cn.		86400	IN	NS	ns1.huaweicloud-dns.net.
sysgeek.cn.		86400	IN	NS	ns1.huaweicloud-dns.org.
3qdaqa092ee5belp64a74ebnb8j53d7e.cn. 21600 IN NSEC3 1 1 10 AEF123AB 3QHKTF6LTFG8AAFUUAJSR8RVAJP99SFU NS SOA RRSIG DNSKEY NSEC3PARAM
3qdaqa092ee5belp64a74ebnb8j53d7e.cn. 21600 IN RRSIG NSEC3 8 2 21600 20240126133436 20231227124228 38388 cn. aihgA66SxvRaE6RRd9tJ1SEKzOFUhf8c+h1WzC1hIKX17ve08TxVnUfb +dVQR9k/ZTFklmC/9SArfF8R5pwA7aur6gDzQx9Y9Ugl73Qw0VhvcdPH vx1rSHIoW4i/clcxu5L5HbwFCq85Acx3V6LQqPk7VqV3luMMqrQyZszY 4EA=
2q9h6g1dqdlnftsbordj8gdm0grfl0gn.cn. 21600 IN NSEC3 1 1 10 AEF123AB 2QHGG80ERRIBTRQCU6GOQRJHPNE4HBR7 NS DS RRSIG
2q9h6g1dqdlnftsbordj8gdm0grfl0gn.cn. 21600 IN RRSIG NSEC3 8 2 21600 20240126125238 20231227124407 38388 cn. Lw1LJerEQegBdOa+5kkSlx6kvndJ3d1uslPlM56Z954fFXhIzLgsT4nz YYGqLNKe+wr74bssyU4p4hoPX3QSGUtERIPtw2daOqDT0k5bIkQN5lbQ rdRZhfqt3qcPgQSTzfBk5ftNRSe87Bq1ebxyaSL2gqk5Owt9mtSeLvD+ Hxc=
;; Received 747 bytes from 203.119.25.1#53(a.dns.cn) in 2 ms

www.sysgeek.cn.		3600	IN	CNAME	www.sysgeek.cn.eo.dnse0.com.
;; Received 84 bytes from 139.159.208.46#53(ns1.huaweicloud-dns.com) in 43 ms
```

抓包分析如下所示：
```bash
# tcpdump -r sysgeek_trace.pcap -nn
reading from file sysgeek_trace.pcap, link-type EN10MB (Ethernet)
19:52:32.664436 IP 10.106.4.203.47974 > 10.6.6.6.53: 36137 [1au] NS? . (28)
19:52:32.664635 IP 10.6.6.6.53 > 10.106.4.203.47974: 36137 13/0/1 NS b.root-servers.net., NS k.root-servers.net., NS c.root-servers.net., NS d.root-servers.net., NS l.root-servers.net., NS h.root-servers.net., NS a.root-servers.net., NS j.root-servers.net., NS i.root-servers.net., NS f.root-servers.net., NS e.root-servers.net., NS g.root-servers.net., NS m.root-servers.net. (239)
19:52:32.666006 IP 10.106.4.203.52847 > 10.6.6.6.53: 41082+ A? b.root-servers.net. (36)
19:52:32.666017 IP 10.106.4.203.52847 > 10.6.6.6.53: 29875+ AAAA? b.root-servers.net. (36)
19:52:32.666362 IP 10.6.6.6.53 > 10.106.4.203.52847: 41082 1/0/0 A 170.247.170.2 (52)
19:52:32.669534 IP 10.6.6.6.53 > 10.106.4.203.52847: 29875 1/0/0 AAAA 2801:1b8:10::b (64)
19:52:32.669985 IP 10.106.4.203.39545 > 10.10.10.10.53: 4250+ A? k.root-servers.net. (36)
19:52:32.669993 IP 10.106.4.203.39545 > 10.10.10.10.53: 38051+ AAAA? k.root-servers.net. (36)
19:52:32.672972 IP 10.10.10.10.53 > 10.106.4.203.39545: 4250 1/0/0 A 193.0.14.129 (52)
19:52:32.673537 IP 10.10.10.10.53 > 10.106.4.203.39545: 38051 1/0/0 AAAA 2001:7fd::1 (64)
19:52:32.673740 IP 10.106.4.203.43905 > 10.6.6.6.53: 12831+ A? c.root-servers.net. (36)
19:52:32.673746 IP 10.106.4.203.43905 > 10.6.6.6.53: 63014+ AAAA? c.root-servers.net. (36)
19:52:32.677680 IP 10.6.6.6.53 > 10.106.4.203.43905: 12831 1/0/0 A 192.33.4.12 (52)
19:52:32.680238 IP 10.6.6.6.53 > 10.106.4.203.43905: 63014 1/0/0 AAAA 2001:500:2::c (64)
19:52:32.680419 IP 10.106.4.203.48120 > 10.10.10.10.53: 48680+ A? d.root-servers.net. (36)
19:52:32.680425 IP 10.106.4.203.48120 > 10.10.10.10.53: 59951+ AAAA? d.root-servers.net. (36)
19:52:32.684047 IP 10.10.10.10.53 > 10.106.4.203.48120: 48680 1/0/0 A 199.7.91.13 (52)
19:52:32.684484 IP 10.10.10.10.53 > 10.106.4.203.48120: 59951 1/0/0 AAAA 2001:500:2d::d (64)
19:52:32.684664 IP 10.106.4.203.63866 > 10.6.6.6.53: 29749+ A? l.root-servers.net. (36)
19:52:32.684670 IP 10.106.4.203.63866 > 10.6.6.6.53: 10811+ AAAA? l.root-servers.net. (36)
19:52:32.688402 IP 10.6.6.6.53 > 10.106.4.203.63866: 29749 1/0/0 A 199.7.83.42 (52)
19:52:32.689021 IP 10.6.6.6.53 > 10.106.4.203.63866: 10811 1/0/0 AAAA 2001:500:9f::42 (64)
19:52:32.689198 IP 10.106.4.203.56863 > 10.10.10.10.53: 27263+ A? h.root-servers.net. (36)
19:52:32.689205 IP 10.106.4.203.56863 > 10.10.10.10.53: 50821+ AAAA? h.root-servers.net. (36)
19:52:32.692482 IP 10.10.10.10.53 > 10.106.4.203.56863: 27263 1/0/0 A 198.97.190.53 (52)
19:52:32.692919 IP 10.10.10.10.53 > 10.106.4.203.56863: 50821 1/0/0 AAAA 2001:500:1::53 (64)
19:52:32.693091 IP 10.106.4.203.37892 > 10.6.6.6.53: 53823+ A? a.root-servers.net. (36)
19:52:32.693098 IP 10.106.4.203.37892 > 10.6.6.6.53: 14406+ AAAA? a.root-servers.net. (36)
19:52:32.696254 IP 10.6.6.6.53 > 10.106.4.203.37892: 14406 1/0/0 AAAA 2001:503:ba3e::2:30 (64)
19:52:32.696675 IP 10.6.6.6.53 > 10.106.4.203.37892: 53823 1/0/0 A 198.41.0.4 (52)
19:52:32.696848 IP 10.106.4.203.41555 > 10.10.10.10.53: 158+ A? j.root-servers.net. (36)
19:52:32.696855 IP 10.106.4.203.41555 > 10.10.10.10.53: 26276+ AAAA? j.root-servers.net. (36)
19:52:32.700245 IP 10.10.10.10.53 > 10.106.4.203.41555: 26276 1/0/0 AAAA 2001:503:c27::2:30 (64)
19:52:32.700499 IP 10.10.10.10.53 > 10.106.4.203.41555: 158 1/0/0 A 192.58.128.30 (52)
19:52:32.700679 IP 10.106.4.203.50224 > 10.6.6.6.53: 9830+ A? i.root-servers.net. (36)
19:52:32.700685 IP 10.106.4.203.50224 > 10.6.6.6.53: 24173+ AAAA? i.root-servers.net. (36)
19:52:32.703987 IP 10.6.6.6.53 > 10.106.4.203.50224: 9830 1/0/0 A 192.36.148.17 (52)
19:52:32.704737 IP 10.6.6.6.53 > 10.106.4.203.50224: 24173 1/0/0 AAAA 2001:7fe::53 (64)
19:52:32.704911 IP 10.106.4.203.55377 > 10.10.10.10.53: 34820+ A? f.root-servers.net. (36)
19:52:32.704918 IP 10.106.4.203.55377 > 10.10.10.10.53: 58378+ AAAA? f.root-servers.net. (36)
19:52:32.705174 IP 10.10.10.10.53 > 10.106.4.203.55377: 34820 1/0/0 A 192.5.5.241 (52)
19:52:32.705183 IP 10.10.10.10.53 > 10.106.4.203.55377: 58378 1/0/0 AAAA 2001:500:2f::f (64)
19:52:32.705349 IP 10.106.4.203.39820 > 10.6.6.6.53: 48651+ A? e.root-servers.net. (36)
19:52:32.705354 IP 10.106.4.203.39820 > 10.6.6.6.53: 21010+ AAAA? e.root-servers.net. (36)
19:52:32.708978 IP 10.6.6.6.53 > 10.106.4.203.39820: 48651 1/0/0 A 192.203.230.10 (52)
19:52:32.709535 IP 10.6.6.6.53 > 10.106.4.203.39820: 21010 1/0/0 AAAA 2001:500:a8::e (64)
19:52:32.709713 IP 10.106.4.203.46330 > 10.10.10.10.53: 22233+ A? g.root-servers.net. (36)
19:52:32.709719 IP 10.106.4.203.46330 > 10.10.10.10.53: 24800+ AAAA? g.root-servers.net. (36)
19:52:32.712553 IP 10.10.10.10.53 > 10.106.4.203.46330: 24800 1/0/0 AAAA 2001:500:12::d0d (64)
19:52:32.712919 IP 10.10.10.10.53 > 10.106.4.203.46330: 22233 1/0/0 A 192.112.36.4 (52)
19:52:32.713294 IP 10.106.4.203.38868 > 10.6.6.6.53: 20000+ A? m.root-servers.net. (36)
19:52:32.713304 IP 10.106.4.203.38868 > 10.6.6.6.53: 35886+ AAAA? m.root-servers.net. (36)
19:52:32.716209 IP 10.6.6.6.53 > 10.106.4.203.38868: 20000 1/0/0 A 202.12.27.33 (52)
19:52:32.717382 IP 10.6.6.6.53 > 10.106.4.203.38868: 35886 1/0/0 AAAA 2001:dc3::35 (64)
19:52:32.719622 IP 10.106.4.203.47483 > 202.12.27.33.53: 14948 [1au] A? www.sysgeek.cn. (43)
19:52:32.819272 IP 202.12.27.33.53 > 10.106.4.203.47483: 14948- 0/8/12 (729)
19:52:32.820113 IP 10.106.4.203.37479 > 10.10.10.10.53: 52476+ A? d.dns.cn. (26)
19:52:32.820123 IP 10.106.4.203.37479 > 10.10.10.10.53: 59402+ AAAA? d.dns.cn. (26)
19:52:32.823483 IP 10.10.10.10.53 > 10.106.4.203.37479: 59402 1/0/0 AAAA 2001:dc7:1000::1 (54)
19:52:32.823562 IP 10.10.10.10.53 > 10.106.4.203.37479: 52476 1/0/0 A 203.119.28.1 (42)
19:52:32.823761 IP 10.106.4.203.46481 > 10.6.6.6.53: 34378+ A? b.dns.cn. (26)
19:52:32.823767 IP 10.106.4.203.46481 > 10.6.6.6.53: 31312+ AAAA? b.dns.cn. (26)
19:52:32.826656 IP 10.6.6.6.53 > 10.106.4.203.46481: 34378 1/0/0 A 203.119.26.1 (42)
19:52:32.827859 IP 10.6.6.6.53 > 10.106.4.203.46481: 31312 1/0/0 AAAA 2001:dc7:1::1 (54)
19:52:32.828033 IP 10.106.4.203.57816 > 10.10.10.10.53: 27700+ A? a.dns.cn. (26)
19:52:32.828040 IP 10.106.4.203.57816 > 10.10.10.10.53: 47674+ AAAA? a.dns.cn. (26)
19:52:32.836082 IP 10.10.10.10.53 > 10.106.4.203.57816: 47674 1/0/0 AAAA 2001:dc7::1 (54)
19:52:32.838645 IP 10.10.10.10.53 > 10.106.4.203.57816: 27700 1/0/0 A 203.119.25.1 (42)
19:52:32.838812 IP 10.106.4.203.55227 > 10.6.6.6.53: 45215+ A? c.dns.cn. (26)
19:52:32.838819 IP 10.106.4.203.55227 > 10.6.6.6.53: 45221+ AAAA? c.dns.cn. (26)
19:52:32.845093 IP 10.6.6.6.53 > 10.106.4.203.55227: 45215 1/0/0 A 203.119.27.1 (42)
19:52:32.847378 IP 10.6.6.6.53 > 10.106.4.203.55227: 45221 1/0/0 AAAA 2001:dc7:2::1 (54)
19:52:32.847555 IP 10.106.4.203.64411 > 10.10.10.10.53: 48266+ A? ns.cernet.net. (31)
19:52:32.847569 IP 10.106.4.203.64411 > 10.10.10.10.53: 53396+ AAAA? ns.cernet.net. (31)
19:52:32.851570 IP 10.10.10.10.53 > 10.106.4.203.64411: 53396 2/0/0 AAAA 2001:250:c006::44, AAAA 2001:dd9::44 (87)
19:52:32.851888 IP 10.10.10.10.53 > 10.106.4.203.64411: 48266 2/0/0 A 103.137.60.44, A 202.112.0.44 (63)
19:52:32.852081 IP 10.106.4.203.64295 > 10.6.6.6.53: 13526+ A? e.dns.cn. (26)
19:52:32.852088 IP 10.106.4.203.64295 > 10.6.6.6.53: 41691+ AAAA? e.dns.cn. (26)
19:52:32.855505 IP 10.6.6.6.53 > 10.106.4.203.64295: 41691 1/0/0 AAAA 2001:dc7:3::1 (54)
19:52:32.856006 IP 10.6.6.6.53 > 10.106.4.203.64295: 13526 1/0/0 A 203.119.29.1 (42)
19:52:32.856415 IP 10.106.4.203.46575 > 203.119.27.1.53: 63115 [1au] A? www.sysgeek.cn. (43)
19:52:32.861115 IP 203.119.27.1.53 > 10.106.4.203.46575: 63115- 0/8/1 (747)
19:52:32.861712 IP 10.106.4.203.54026 > 10.10.10.10.53: 59423+ A? ns1.huaweicloud-dns.cn. (40)
19:52:32.861721 IP 10.106.4.203.54026 > 10.10.10.10.53: 46122+ AAAA? ns1.huaweicloud-dns.cn. (40)
19:52:32.868222 IP 10.10.10.10.53 > 10.106.4.203.54026: 59423 2/0/0 A 139.9.224.17, A 139.9.23.86 (72)
19:52:32.937553 IP 10.10.10.10.53 > 10.106.4.203.54026: 46122 1/0/0 AAAA 2407:c080:20:ffff:ffff:fffe:0:1 (68)
19:52:32.937759 IP 10.106.4.203.60256 > 10.6.6.6.53: 48688+ A? ns1.huaweicloud-dns.com. (41)
19:52:32.937766 IP 10.106.4.203.60256 > 10.6.6.6.53: 52281+ AAAA? ns1.huaweicloud-dns.com. (41)
19:52:32.941623 IP 10.6.6.6.53 > 10.106.4.203.60256: 52281 1/0/0 AAAA 2407:c080:20:ffff:ffff:fffe:0:1 (69)
19:52:32.941977 IP 10.6.6.6.53 > 10.106.4.203.60256: 48688 2/0/0 A 139.159.208.46, A 139.9.224.17 (73)
19:52:32.942162 IP 10.106.4.203.64860 > 10.10.10.10.53: 1607+ A? ns1.huaweicloud-dns.net. (41)
19:52:32.942169 IP 10.106.4.203.64860 > 10.10.10.10.53: 1614+ AAAA? ns1.huaweicloud-dns.net. (41)
19:52:32.946157 IP 10.10.10.10.53 > 10.106.4.203.64860: 1607 3/0/0 A 159.138.160.107, A 159.138.208.3, A 159.138.17.112 (89)
19:52:32.946432 IP 10.10.10.10.53 > 10.106.4.203.64860: 1614 1/0/0 AAAA 2407:c080:20:ffff:ffff:fffe:0:1 (69)
19:52:32.946636 IP 10.106.4.203.59395 > 10.6.6.6.53: 167+ A? ns1.huaweicloud-dns.org. (41)
19:52:32.946642 IP 10.106.4.203.59395 > 10.6.6.6.53: 58541+ AAAA? ns1.huaweicloud-dns.org. (41)
19:52:32.950006 IP 10.6.6.6.53 > 10.106.4.203.59395: 167 3/0/0 A 159.138.17.187, A 159.138.160.107, A 159.138.208.3 (89)
19:52:32.950491 IP 10.6.6.6.53 > 10.106.4.203.59395: 58541 1/0/0 AAAA 2407:c080:20:ffff:ffff:fffe:0:1 (69)
19:52:32.950889 IP 10.106.4.203.42062 > 139.9.224.17.53: 52516 [1au] A? www.sysgeek.cn. (43)
19:52:32.987438 IP 139.9.224.17.53 > 10.106.4.203.42062: 52516*- 1/0/1 CNAME www.sysgeek.cn.eo.dnse0.com. (84)
19:52:33.291795 IP 10.106.4.203.38194 > 10.6.6.6.53: 22286+ AAAA? pcs-api.internal. (34)
19:52:33.291855 IP 10.106.4.203.62248 > 10.10.10.10.53: 17049+ A? pcs-api.internal. (34)
19:52:33.291917 IP 10.10.10.10.53 > 10.106.4.203.62248: 17049* 2/0/0 A 10.20.254.105, A 10.20.253.166 (66)
19:52:33.292021 IP 10.6.6.6.53 > 10.106.4.203.38194: 22286* 0/1/0 (98)
```

**1>查询根的NS以及对应的A记录**
![](../attachments/Pasted%20image%2020240116200038.png)

**2>查询一级域的NS以及对应的A记录**
![](../attachments/Pasted%20image%2020240116200248.png)

**3>查询二级域的NS以及对应的A记录**
![](../attachments/Pasted%20image%2020240116200603.png)

**4>查询权威记录**
![](../attachments/Pasted%20image%2020240116200754.png)
>注：为什么此中查询到Cname之后就结束了，并没有后续的A记录的查询。如下所示。
> 访问别名只会返回规范名，不会返回规范名的A记录？？？

![](../attachments/Pasted%20image%2020240116200920.png)
## 查询某个域名的zone的SOA记录
```bash
dig -t SOA 域名
```
![](../attachments/Pasted%20image%2020240116203517.png)

## 输出简化
```bash
+nocomments     – 不显示注释
+noauthority    – 不显示AUTHORITY SECTION
+noadditional   – 不显示ADDITIONAL SECTION
+nostats        – 不显示Stats section
+noanswer       – 不显示ANSWER SECTION
+noall          - 不显示所有的信息，一般会这样用 dig zhouliang.pro +noall +answer
和上面参数对应还有 +comments ， +answer 等，后文有示例，此处不赘述。另外，还有如下两个参数需要了解：

+short    - 显示简短的信息
-t       指定查询的记录类型，可以是CNAME、A、MX、NS，分别表示CNAME、A记录、MX记录、DNS服务器，默认是A
-x       表示反向查找，也就是根据IP地址查找域名
-c       可以设置协议类型（class），包括IN(默认)、CH和HS
-f       dig支持从一个文件里读取内容进行批量查询
-4和-6   用于设置仅适用哪一种作为查询包传输协议，分别对应着IPv4和IPv6
-q       显式设置你要查询的域名
```
### 输出简短结果
如果只想获取 DNS 查询的简短响应，可以使用`+short`参数，例如：
```bash
# dig www.sysgeek.cn +short
www.sysgeek.cn.eo.dnse0.com.
153.3.224.53
153.3.223.140
```
### 输出详细响应
要获取 DNS 查询更详细的响应结果，可以先使用`+noall`参数关闭所有结果，再使用`+answer`参数打开结果部分。
```bash
dig www.sysgeek.cn +noall +answer

# 使用 dig 命令查询 A 记录
dig +nocmd www.sysgeek.cn a +noall +answer

# 使用 dig 命令查询 CNAME 记录
dig +nocmd www.sysgeek.cn cname +noall +answer
```
范例如下：
```bash
# dig www.sysgeek.cn +noall +answer

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.8 <<>> www.sysgeek.cn +noall +answer
;; global options: +cmd
www.sysgeek.cn.		1	IN	CNAME	www.sysgeek.cn.eo.dnse0.com.
www.sysgeek.cn.eo.dnse0.com. 1	IN	A	153.3.223.140
www.sysgeek.cn.eo.dnse0.com. 1	IN	A	153.3.224.53
```
# 注意
## `/etc/resolv.conf` 文件中的超时和重试不对`dig、host、nslookup`生效

`/etc/resolv.conf` 文件中也存在2个参数。一个是超时的 timeout，一个是重试的 attempts，默认情况下，前者是 5s 后者是 2 次。
` dig, host, nslook` 这类工具，因为他们并没有调用 resolver 的库，因此 `/etc/resolv.conf` 文件中的超时和重试不生效。

任何使用 `glibc resolver` 都需要经过 `resolv.conf` 文件。可以使用 `getent 或者 ping` 来测试`/etc/resolv.conf` 文件中的配置。

```bash
strace -t getent hosts baidu.com 
strace ping baidu.com
```

## `/etc/hosts`文件不对`dig、host、nslookup`生效

`dig, nslook` 只会解析 `resolv.conf` 的内容，而不会解析 `hosts` 里面内容，所以如果想让 dig 解析 `hosts` 里面的内容，可以通过 `dnsmasq` 实现。



# 参考
```c
http://linux.51yip.com/search/dig
```