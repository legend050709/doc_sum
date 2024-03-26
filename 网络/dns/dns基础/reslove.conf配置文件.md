```table-of-contents
```
# 概述
通常情况下，在 Linux 中可以用来配置 DNS 地址的文件有两个：

- 解析配置文件 `/etc/resolv.conf`，文件中的注释行可以采用 `#` 或者 `;` 开头
- 网卡 `ifcfg` 配置文件 `/etc/sysconfig/network-scripts/ifcfg-eth*`

# resolv.conf 的配置参数
通过查看 [man手册页](http://man7.org/linux/man-pages/man5/resolv.conf.5.html)，会看到有很多相关的参数，这里只说比较常用的几个
```c
man 5 resolv.conf
```
![](attachments/Pasted%20image%2020231107170900.png)
## nameserver
这个参数会指定系统使用的 DNS 的 IP 地址，在生产环境中，经常会看到 `/etc/resolv.conf` 中配置了一个或多个 `nameserver` 。

### 特性
配置了多个 `nameserver` 后（未指定rotate时），系统总是使用第一个 `nameserver` 对应的地址来进行 DNS 的查询，并未进行轮询、随机、或者均衡调度。

如果第一个 `nameserver` 对应的地址是不可用的 IP ，域名的解析均是从第一个 `nameserver` 对用的地址进行查询的，系统会按照 `nameserver` 设定的顺序从上往下进行顺序尝试，当无法获取结果后将尝试使用下一个，并且系统不会记录任何一个 `nameserver` 的工作状态，下次的域名查询还是使用第一个server进行查询，即使第一个server被证明无效。

如果有多个 `nameserver` 不可用时，对于 `dig` 和 `ping` 命令而言生效的只有前三个，当前三个 `nameserver` 都不可用时，不会再向其余的 `nameserver`请求解析。
对于 `curl` 命令来说生效的是全部，依次从上向下轮询，直到找到能有所响应的为止。

### 范例
- **选择第一个server**：

```c
[root@bogon ~]# cat /etc/resolv.conf 
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
nameserver 1.2.4.8
nameserver 180.76.76.76
nameserver 223.5.5.5
[root@bogon ~]# 
[root@bogon ~]# dig +short +tries=1 www.redhat.com www.centos.org www.kernel.org
```

解析的同时，使用 `tcpdump` 命令分析通信数据，从结果中可以看到，使用 dig 对三个域名的解析只用到了第一个 `nameserver` 的地址，即 8.8.8.8.
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:51:38.162534 IP 192.168.127.154.58459 > 8.8.8.8.53: 27136+ [1au] A? www.redhat.com. (43)
20:51:38.165500 IP 8.8.8.8.53 > 192.168.127.154.58459: 27136 4/0/0 CNAME ds-www.redhat.com.edgekey.net., CNAME ds-www.redhat.com.edgekey.net.globalredir.akadns.net., CNAME e3396.ca2.s.tl88.net., A 210.192.117.211 (185)
20:51:38.166018 IP 192.168.127.154.58937 > 8.8.8.8.53: 37326+ [1au] A? www.centos.org. (43)
20:51:38.449824 IP 8.8.8.8.53 > 192.168.127.154.58937: 37326 1/3/4 A 85.12.30.226 (161)
20:51:38.450397 IP 192.168.127.154.47932 > 8.8.8.8.53: 5121+ [1au] A? www.kernel.org. (43)
20:51:38.546600 IP 8.8.8.8.53 > 192.168.127.154.47932: 5121 3/6/1 CNAME git.kernel.org., CNAME hkg.git.kernel.org., A 147.75.42.139 (237)
```

换成一次解析一个域名
```c
[root@bogon ~]# dig +short +tries=1 www.baidu.com
[root@bogon ~]# dig +short +tries=1 www.qq.com
[root@bogon ~]# dig +short +tries=1 www.taobao.com
[root@bogon ~]# dig +short +tries=1 www.amazon.com
```
结果同上，也是只用到了第一个 `nameserver` 的地址，即 8.8.8.8.


- **第一个server查询失败时**：
```c
[root@bogon ~]# cat /etc/resolv.conf 
nameserver 123.45.67.8
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
nameserver 1.2.4.8
nameserver 180.76.76.76
nameserver 223.5.5.5
```
```c
[root@bogon ~]# dig +short +tries=1 www.redhat.com www.centos.org www.kernel.org
```

抓包结果:
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
21:09:32.043033 IP 124.160.121.68.48453 > 123.45.67.8.53: 3179+ [1au] A? www.redhat.com. (43)
21:09:33.042929 IP 124.160.121.68.37014 > 8.8.8.8.53: 3179+ [1au] A? www.redhat.com. (43)
21:09:33.327958 IP 8.8.8.8.53 > 124.160.121.68.37014: 3179 4/0/1 CNAME ds-www.redhat.com.edgekey.net., CNAME ds-www.redhat.com.edgekey.net.globalredir.akadns.net., CNAME e3396.ca2.s.tl88.net., A 122.224.45.211 (196)
21:09:33.328311 IP 124.160.121.68.54898 > 123.45.67.8.53: 13619+ [1au] A? www.centos.org. (43)
21:09:34.328397 IP 124.160.121.68.43004 > 8.8.8.8.53: 13619+ [1au] A? www.centos.org. (43)
21:09:34.563133 IP 8.8.8.8.53 > 124.160.121.68.43004: 13619 1/0/1 A 85.12.30.226 (59)
21:09:34.563347 IP 124.160.121.68.33925 > 123.45.67.8.53: 13240+ [1au] A? www.kernel.org. (43)
21:09:35.563401 IP 124.160.121.68.36621 > 8.8.8.8.53: 13240+ [1au] A? www.kernel.org. (43)
21:09:35.624542 IP 8.8.8.8.53 > 124.160.121.68.36621: 13240 3/0/1 CNAME git.kernel.org., CNAME hkg.git.kernel.org., A 147.75.42.139 (95)
```


- **多个server不可用时**：
```c
[root@bogon ~]# cat /etc/resolv.conf 
nameserver 123.45.67.8
nameserver 123.45.67.9
nameserver 123.45.67.10
nameserver 123.45.67.11
nameserver 123.45.67.12
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
nameserver 1.2.4.8
nameserver 180.76.76.76
nameserver 223.5.5.5
[root@bogon ~]# 
[root@bogon ~]# dig +short +tries=1  www.redhat.com www.centos.org www.kernel.org 
;; connection timed out; no servers could be reached
;; connection timed out; no servers could be reached
;; connection timed out; no servers could be reached
```

抓包结果：
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
21:25:18.162523 IP 124.160.121.68.35636 > 123.45.67.8.53: 34369+ [1au] A? www.redhat.com. (43)
21:25:19.162528 IP 124.160.121.68.35529 > 123.45.67.9.53: 34369+ [1au] A? www.redhat.com. (43)
21:25:20.162618 IP 124.160.121.68.44291 > 123.45.67.10.53: 34369+ [1au] A? www.redhat.com. (43)
21:25:25.162806 IP 124.160.121.68.57363 > 123.45.67.8.53: 36309+ [1au] A? www.centos.org. (43)
21:25:26.162892 IP 124.160.121.68.37002 > 123.45.67.9.53: 36309+ [1au] A? www.centos.org. (43)
21:25:27.162974 IP 124.160.121.68.56470 > 123.45.67.10.53: 36309+ [1au] A? www.centos.org. (43)
21:25:32.163160 IP 124.160.121.68.52253 > 123.45.67.8.53: 59858+ [1au] A? www.kernel.org. (43)
21:25:33.163201 IP 124.160.121.68.36526 > 123.45.67.9.53: 59858+ [1au] A? www.kernel.org. (43)
21:25:34.163285 IP 124.160.121.68.42950 > 123.45.67.10.53: 59858+ [1au] A? www.kernel.org. (43)
```
从上面的结果可以看到，`dig` 在请求解析每个域名时，都会向前三个 `nameserver` 请求。

```c
[root@bogon ~]# cat /etc/resolv.conf
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 123.45.67.3
nameserver 123.45.67.4
nameserver 123.45.67.5
nameserver 123.45.67.6
nameserver 123.45.67.7
nameserver 123.45.67.8
nameserver 8.8.8.8
[root@bogon ~]# curl -I www.baidu.com
HTTP/1.1 200 OK
Accept-Ranges: bytes
Cache-Control: private, no-cache, no-store, proxy-revalidate, no-transform
Connection: Keep-Alive
Content-Length: 277
Content-Type: text/html
Date: Sat, 12 May 2018 11:44:10 GMT
Etag: "575e1f5c-115"
Last-Modified: Mon, 13 Jun 2016 02:50:04 GMT
Pragma: no-cache
Server: bfe/1.0.8.18
```
抓包结果:
```c
[root@bogon ~]# tcpdump -nn -i eth1   udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
19:43:37.708473 IP 124.160.121.68.52347 > 123.45.67.1.53: 57511+ A? www.baidu.com. (31)
19:43:42.709785 IP 124.160.121.68.44730 > 123.45.67.2.53: 57511+ A? www.baidu.com. (31)
19:43:47.397695 IP 124.160.121.68.52848 > 123.45.67.3.53: 57511+ A? www.baidu.com. (31)
19:43:50.531180 IP 124.160.121.68.32775 > 123.45.67.4.53: 57511+ A? www.baidu.com. (31)
19:43:54.282260 IP 124.160.121.68.34249 > 123.45.67.5.53: 57511+ A? www.baidu.com. (31)
19:43:57.415747 IP 124.160.121.68.45492 > 123.45.67.6.53: 57511+ A? www.baidu.com. (31)
19:44:02.417119 IP 124.160.121.68.53927 > 123.45.67.7.53: 57511+ A? www.baidu.com. (31)
19:44:05.862921 IP 124.160.121.68.48605 > 123.45.67.8.53: 57511+ A? www.baidu.com. (31)
19:44:09.933530 IP 124.160.121.68.55600 > 8.8.8.8.53: 57511+ A? www.baidu.com. (31)
19:44:10.038720 IP 8.8.8.8.53 > 124.160.121.68.55600: 57511 3/0/0 CNAME www.a.shifen.com., A 61.135.169.121, A 61.135.169.125 (90)
```
### 注意
当第一个 `namesever` 解析超过指定的超时后会向第二个 `nameserver` 请求解析，此时第一个 `nameserver` 的 `socket` 已经关闭。
于是**不存在**这种情况：虽然第一个 nameserver 已经超时了，系统在向第二个 `nameserver` 请求解析时，就会有可能这时第一个 `nameserver` 将解析结果返回给系统了。
## domain
### 特性
![](attachments/Pasted%20image%2020231107173004.png)
当请求解析的内容不包含有域时，不管请求的内容是不是合法的，系统都直接填充 `domain` 参数，并且最后还会添加一个根域（即.）。
当请求解析的内容包含有域时，不管请求的内容是不是合法的，系统都会先直接解析而不是直接填充 `domain` 参数。

> 注：是否含有域，可以理解为是否含有. ???
### 范例
- **请求解析的内容不包含有域时**
```c
[root@bogon ~]# cat /etc/resolv.conf 
domain baidu.com
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
[root@bogon ~]# 
[root@bogon ~]# ping -c 1 www
PING www.a.shifen.com (61.135.169.121) 56(84) bytes of data.
64 bytes from 61.135.169.121 (61.135.169.121): icmp_seq=1 ttl=54 time=28.3 ms

--- www.a.shifen.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 28.390/28.390/28.390/0.000 ms
```
抓包结果：
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
15:34:53.581383 IP 124.160.121.68.42899 > 8.8.8.8.53: 62551+ A? www.baidu.com. (31)
15:34:53.711318 IP 8.8.8.8.53 > 124.160.121.68.42899: 62551 3/0/0 CNAME www.a.shifen.com., A 61.135.169.121, A 61.135.169.125 (90)
15:34:53.740080 IP 124.160.121.68.45679 > 8.8.8.8.53: 35362+ PTR? 121.169.135.61.in-addr.arpa. (45)
15:34:53.790788 IP 8.8.8.8.53 > 124.160.121.68.45679: 35362 NXDomain 0/1/0 (97)
```

```c
[root@bogon ~]# ping -c 1 baidu
ping: baidu: Name or service not known
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
15:40:46.521311 IP 124.160.121.68.54008 > 8.8.8.8.53: 51825+ A? baidu.baidu.com. (33)
15:40:46.619891 IP 8.8.8.8.53 > 124.160.121.68.54008: 51825 NXDomain 0/1/0 (76)
15:40:46.619961 IP 124.160.121.68.37288 > 8.8.8.8.53: 59104+ A? baidu. (23)
15:40:46.668562 IP 8.8.8.8.53 > 124.160.121.68.37288: 59104 0/1/0 (93)
```

- **请求解析的内容包含有域，且该域有对应的 A 记录**
```c
[root@bogon ~]# cat /etc/resolv.conf 
domain .org
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
```
```c
[root@bogon ~]# ping -c1 baidu.com
PING baidu.com (123.125.115.110) 56(84) bytes of data.
64 bytes from 123.125.115.110 (123.125.115.110): icmp_seq=1 ttl=54 time=33.6 ms

--- baidu.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 33.654/33.654/33.654/0.000 ms
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
15:52:51.006971 IP 124.160.121.68.35960 > 8.8.8.8.53: 29630+ A? baidu.com. (27)
15:52:51.054703 IP 8.8.8.8.53 > 124.160.121.68.35960: 29630 2/0/0 A 123.125.115.110, A 220.181.57.216 (59)
15:52:51.088759 IP 124.160.121.68.52785 > 8.8.8.8.53: 12744+ PTR? 110.115.125.123.in-addr.arpa. (46)
15:52:51.134573 IP 8.8.8.8.53 > 124.160.121.68.52785: 12744 NXDomain 0/1/0 (100)
```

- **请求解析的内容包含有域，但该域不合法或者没有对应的A记录** 
```c
[root@bogon ~]# ping -c1 baidu123.com
ping: baidu123.com: Name or service not known
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
15:56:01.274770 IP 124.160.121.68.60947 > 8.8.8.8.53: 32132+ A? baidu123.com. (30)
15:56:01.666132 IP 8.8.8.8.53 > 124.160.121.68.60947: 32132 0/1/0 (96)
```
## search
search的说明如下，此中的search应该是包含了 domain的特性。
![](attachments/Pasted%20image%2020231107174422.png)

### 特性
指定一组域名（用空格分割），当请求解析的内容仅写出主机名（没有点），就依次添加search里的每一项依次组成FQDN（完全合格域名）来查询，直到匹配完所有列出的域名，给出最终的结果。

```c
[root@bogon ~]# cat /etc/resolv.conf
search redhat.com centos.org baidu.com kernel.org
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
[root@bogon ~]# 
[root@bogon ~]# ping -c 1 -W 1 -q pan
PING yiyun.n.shifen.com (111.206.37.70) 56(84) bytes of data.

--- yiyun.n.shifen.com ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
rtt min/avg/max/mdev = 26.970/26.970/26.970/0.000 ms
```
```c
[root@bogon ~]# ping -c 1 -W 1 -q x1y2z3
ping: x1y2z3: Name or service not known
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
21:57:47.079874 IP 124.160.121.68.59551 > 8.8.8.8.53: 14646+ A? x1y2z3.redhat.com. (35)
21:57:47.388342 IP 8.8.8.8.53 > 124.160.121.68.59551: 14646 0/1/0 (79)
21:57:47.388416 IP 124.160.121.68.40022 > 8.8.8.8.53: 10171+ A? x1y2z3.centos.org. (35)
21:57:47.702240 IP 8.8.8.8.53 > 124.160.121.68.40022: 10171 NXDomain 0/1/0 (86)
21:57:47.702297 IP 124.160.121.68.56305 > 8.8.8.8.53: 15347+ A? x1y2z3.baidu.com. (34)
21:57:47.804776 IP 8.8.8.8.53 > 124.160.121.68.56305: 15347 NXDomain 0/1/0 (77)
21:57:47.821951 IP 124.160.121.68.40022 > 8.8.8.8.53: 10171+ A? x1y2z3.kernel.org. (35)
21:57:47.832714 IP 8.8.8.8.53 > 124.160.121.68.40022: 10171 NXDomain 0/1/0 (94)
21:57:47.804841 IP 124.160.121.68.52432 > 8.8.8.8.53: 1360+ A? x1y2z3. (24)
21:57:47.857440 IP 8.8.8.8.53 > 124.160.121.68.52432: 1360 NXDomain 0/1/0 (99)
```
### 范例
DNS配置文件如下：
```c
search openstack.local dev.com example.local
nameserver 192.168.122.21
```
例1：请求解析的内容没有点(主机名后面没有点，没有点的查询就认为是主机名)，所以先添加search里的每一项依次组成FQDN（完全合格域名）来查询，完全合格域名查询未找到，就再认为主机名是完全合格域名来查询。
```c
# host -a centos7-bind-1
Trying "centos7-bind-1.openstack.local"
Trying "centos7-bind-1.dev.com"
Trying "centos7-bind-1.example.local"
Trying "centos7-bind-1"
;; connection timed out; no servers could be reached
```

例2：请求解析的内容有点(不是末尾有点)，就认为是完全合格域名，先用它来查询，查询失败就把它当成是主机名来进行，添加search里的每一项组成FQDN（完全合格域名）来查询。
```c
# host -a centos7-bind-1.com
Trying "centos7-bind-1.com"
Received 109 bytes from 192.168.122.21#53 in 177 ms
Trying "centos7-bind-1.com.openstack.local"
Trying "centos7-bind-1.com.dev.com"
Trying "centos7-bind-1.com.example.local"
Host centos7-bind-1.com not found: 3(NXDOMAIN)
Received 125 bytes from 192.168.122.21#53 in 55 ms
```

例3：请求解析的内容有点(末尾有点)，则认为是完全合格域名，只用它来查询（不会再添加search里的每一项）。查询次数会与search里项域名个数有关。
```c
# host -a centos7-bind-1.
Trying "centos7-bind-1"
;; connection timed out; trying next origin
Trying "centos7-bind-1"
;; connection timed out; trying next origin
Trying "centos7-bind-1"
;; connection timed out; trying next origin
Trying "centos7-bind-1"
;; connection timed out; no servers could be reached
```
## options
### timeout
可选选项，系统一次 DNS 解析 timeout 的时间值，单位为秒。系统默认值为 5，最大可以设定的值是 30
#### 范例
下面对于 timeout 的实验我们都在 `/etc/resolv.conf` 中配置无效的 DNS。

在查看了 `dig` 命令的帮助手册之后发现它也有个 timeout 参数 `+time=T`，在不指定该参数的情况下默认为 5 秒，那么我们 **不在** `resolv.conf` 中配置 `timeout` 来进行测试看一下：
```c
[root@bogon ~]# cat /etc/resolv.conf
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 123.45.67.3
[root@bogon ~]# 
[root@bogon ~]# dig +tries=5 +time=2 +short www.baidu.com 
;; connection timed out; no servers could be reached
```

抓包结果
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes

17:00:22.378172 IP 124.160.121.68.39149 > 123.45.67.1.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:23.378136 IP 124.160.121.68.37930 > 123.45.67.2.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:24.378219 IP 124.160.121.68.58526 > 123.45.67.3.53: 18650+ [1au] A? www.baidu.com. (42)

17:00:26.378269 IP 124.160.121.68.39149 > 123.45.67.1.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:27.378355 IP 124.160.121.68.37930 > 123.45.67.2.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:28.378442 IP 124.160.121.68.58526 > 123.45.67.3.53: 18650+ [1au] A? www.baidu.com. (42)

17:00:30.378527 IP 124.160.121.68.39149 > 123.45.67.1.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:31.378614 IP 124.160.121.68.37930 > 123.45.67.2.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:32.378703 IP 124.160.121.68.58526 > 123.45.67.3.53: 18650+ [1au] A? www.baidu.com. (42)

17:00:34.378786 IP 124.160.121.68.39149 > 123.45.67.1.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:35.378870 IP 124.160.121.68.37930 > 123.45.67.2.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:36.378958 IP 124.160.121.68.58526 > 123.45.67.3.53: 18650+ [1au] A? www.baidu.com. (42)

17:00:38.379040 IP 124.160.121.68.39149 > 123.45.67.1.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:39.379122 IP 124.160.121.68.37930 > 123.45.67.2.53: 18650+ [1au] A? www.baidu.com. (42)
17:00:40.379210 IP 124.160.121.68.58526 > 123.45.67.3.53: 18650+ [1au] A? www.baidu.com. (42)
```

为了便于区分，上述内容人工填充了空行。从上面可以看到

- `dig` 向三个 `nameserver` 分别请求解析，向每个 `nameserver` 请求解析的时间间隔为 1 秒而不是 2 秒
- `dig` 在五次对文件中的 `nameserver` 轮询，每次轮询之间的时间间隔为 2 秒

如果我们在 `/etc/resolv.conf` 中配置了 `timeout`，到底是以 `dig` 的为准还是以 `/etc/resolv.conf` 中的为准？
- 当 `dig` 的 timeout 大于 `resolv.conf` 中的 timeout 时
```c
[root@bogon ~]# cat /etc/resolv.conf
options timeout:3
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 123.45.67.3
[root@bogon ~]# 
[root@bogon ~]# dig +tries=5 +timeout=4 +short www.baidu.com  
;; connection timed out; no servers could be reached
```

抓包结果
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
16:06:44.330997 IP 124.160.121.68.60827 > 123.45.67.1.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:45.331000 IP 124.160.121.68.45589 > 123.45.67.2.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:46.331082 IP 124.160.121.68.43181 > 123.45.67.3.53: 12194+ [1au] A? www.baidu.com. (42)

16:06:49.331133 IP 124.160.121.68.60827 > 123.45.67.1.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:50.331220 IP 124.160.121.68.45589 > 123.45.67.2.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:51.331312 IP 124.160.121.68.43181 > 123.45.67.3.53: 12194+ [1au] A? www.baidu.com. (42)

16:06:54.331403 IP 124.160.121.68.60827 > 123.45.67.1.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:55.331488 IP 124.160.121.68.45589 > 123.45.67.2.53: 12194+ [1au] A? www.baidu.com. (42)
16:06:56.331571 IP 124.160.121.68.43181 > 123.45.67.3.53: 12194+ [1au] A? www.baidu.com. (42)

16:06:59.331658 IP 124.160.121.68.60827 > 123.45.67.1.53: 12194+ [1au] A? www.baidu.com. (42)
16:07:00.331747 IP 124.160.121.68.45589 > 123.45.67.2.53: 12194+ [1au] A? www.baidu.com. (42)
16:07:01.331837 IP 124.160.121.68.43181 > 123.45.67.3.53: 12194+ [1au] A? www.baidu.com. (42)

16:07:04.331913 IP 124.160.121.68.60827 > 123.45.67.1.53: 12194+ [1au] A? www.baidu.com. (42)
16:07:05.332004 IP 124.160.121.68.45589 > 123.45.67.2.53: 12194+ [1au] A? www.baidu.com. (42)
16:07:06.332093 IP 124.160.121.68.43181 > 123.45.67.3.53: 12194+ [1au] A? www.baidu.com. (42)
```

dig 没有指定 timeout 参数则使用默认值 5 秒。
```c
[root@bogon ~]# dig +tries=5 +short www.baidu.com 
;; connection timed out; no servers could be reached
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes

16:25:06.558406 IP 124.160.121.68.54373 > 123.45.67.1.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:07.558450 IP 124.160.121.68.46082 > 123.45.67.2.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:08.558486 IP 124.160.121.68.47769 > 123.45.67.3.53: 51295+ [1au] A? www.baidu.com. (42)

16:25:11.558564 IP 124.160.121.68.54373 > 123.45.67.1.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:12.558629 IP 124.160.121.68.46082 > 123.45.67.2.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:13.558724 IP 124.160.121.68.47769 > 123.45.67.3.53: 51295+ [1au] A? www.baidu.com. (42)

16:25:16.558808 IP 124.160.121.68.54373 > 123.45.67.1.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:17.558894 IP 124.160.121.68.46082 > 123.45.67.2.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:18.558976 IP 124.160.121.68.47769 > 123.45.67.3.53: 51295+ [1au] A? www.baidu.com. (42)

16:25:21.559058 IP 124.160.121.68.54373 > 123.45.67.1.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:22.559147 IP 124.160.121.68.46082 > 123.45.67.2.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:23.559230 IP 124.160.121.68.47769 > 123.45.67.3.53: 51295+ [1au] A? www.baidu.com. (42)

16:25:26.559320 IP 124.160.121.68.54373 > 123.45.67.1.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:27.559404 IP 124.160.121.68.46082 > 123.45.67.2.53: 51295+ [1au] A? www.baidu.com. (42)
16:25:28.559488 IP 124.160.121.68.47769 > 123.45.67.3.53: 51295+ [1au] A? www.baidu.com. (42)
```

- 当 `dig` 的 timeout 小于 `resolv.conf` 中的 timeout 时
```c
[root@bogon ~]# cat /etc/resolv.conf
options timeout:10
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 123.45.67.3
[root@bogon ~]#
[root@bogon ~]# dig +tries=3 +time=2 +short www.baidu.com 
;; connection timed out; no servers could be reached
```
```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53 
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes

16:31:54.273506 IP 124.160.121.68.49564 > 123.45.67.1.53: 31368+ [1au] A? www.baidu.com. (42)
16:31:55.273448 IP 124.160.121.68.58295 > 123.45.67.2.53: 31368+ [1au] A? www.baidu.com. (42)
16:31:56.273534 IP 124.160.121.68.43357 > 123.45.67.3.53: 31368+ [1au] A? www.baidu.com. (42)

16:32:06.273581 IP 124.160.121.68.49564 > 123.45.67.1.53: 31368+ [1au] A? www.baidu.com. (42)
16:32:07.273668 IP 124.160.121.68.58295 > 123.45.67.2.53: 31368+ [1au] A? www.baidu.com. (42)
16:32:08.273755 IP 124.160.121.68.43357 > 123.45.67.3.53: 31368+ [1au] A? www.baidu.com. (42)

16:32:18.273872 IP 124.160.121.68.49564 > 123.45.67.1.53: 31368+ [1au] A? www.baidu.com. (42)
16:32:19.273953 IP 124.160.121.68.58295 > 123.45.67.2.53: 31368+ [1au] A? www.baidu.com. (42)
16:32:20.274030 IP 124.160.121.68.43357 > 123.45.67.3.53: 31368+ [1au] A? www.baidu.com. (42)
```

#### 小结
多次测试后发现：

- 对于 `dig` 来说，自身的 timeout 和 `/etc/resolv.conf` 二者的 timeout 都指的是每次轮询完文件 `nameserver` 的超时时间，而不是每个 `nameserver` 的超时时间
- 而对于 `ping` 来说，自身的 timeout 指的是每个包的超时时间，`/etc/resolv.conf` 的 timeout 指的是每个 `nameserver` 的超时时间，不是每次轮询的时间
- 只要 `/etc/resolv.conf` 配置了 timeout 就直接忽略 `dig` 的 timeout ，以 `/etc/resolv.conf` 中配置的为准；否则就以 `dig` 设定的（未设定则使用默认值）为准

### attempts
#### 定义
可选选项，系统尝试解析的次数，如果未设定则默认值是 2。系统允许最大值是 5，即当系统请求解析失败多少次之后，才会给系统或者程序返回解析失败的结果。

对于 `dig` 命令来说，文件中配置的 `attempts` 如果是一个小数，则可能会提示语法错误，如果是 0 则不会进行解析。
因为 `dig` 命令也有个尝试次数的参数 `+tries=T`，默认是请求三次。不管 `/etc/resolv.conf` 有没有配置 `attempts`，在使用 `dig` 的时候都会以 `dig` 的为准。
#### 范例
这里我们要对 `/etc/resolv.conf` 文件中的配置做实验，就使用 `ping` 来测试。
```c
[root@bogon ~]# cat /etc/resolv.conf
options timeout:3
options attempts:2
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 123.45.67.3
[root@bogon ~]#
[root@bogon ~]# time ping -c 3  www.baidu.com 
ping: www.baidu.com: Name or service not known

real    0m36.040s
user    0m0.000s
sys     0m0.004s
```

```c
[root@bogon ~]# tcpdump -nn -i eth0 udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes

17:57:48.821495 IP 124.160.121.68.37237 > 123.45.67.1.53: 62997+ A? www.baidu.com. (31)
17:57:51.824546 IP 124.160.121.68.42482 > 123.45.67.2.53: 62997+ A? www.baidu.com. (31)
17:57:53.826591 IP 124.160.121.68.48320 > 123.45.67.3.53: 62997+ A? www.baidu.com. (31)
17:57:57.830621 IP 124.160.121.68.37237 > 123.45.67.1.53: 62997+ A? www.baidu.com. (31)
17:58:00.833656 IP 124.160.121.68.42482 > 123.45.67.2.53: 62997+ A? www.baidu.com. (31)
17:58:02.835688 IP 124.160.121.68.48320 > 123.45.67.3.53: 62997+ A? www.baidu.com. (31)

17:58:06.839767 IP 124.160.121.68.54390 > 123.45.67.1.53: 47323+ A? www.baidu.com. (31)
17:58:09.842821 IP 124.160.121.68.60543 > 123.45.67.2.53: 47323+ A? www.baidu.com. (31)
17:58:11.844871 IP 124.160.121.68.51676 > 123.45.67.3.53: 47323+ A? www.baidu.com. (31)
17:58:15.848917 IP 124.160.121.68.54390 > 123.45.67.1.53: 47323+ A? www.baidu.com. (31)
17:58:18.851953 IP 124.160.121.68.60543 > 123.45.67.2.53: 47323+ A? www.baidu.com. (31)
17:58:20.853984 IP 124.160.121.68.51676 > 123.45.67.3.53: 47323+ A? www.baidu.com. (31)
```
上面的抓包结果看上去有些费解。我们先算一下时间，已知 `/etc/resolv.conf` 中的 `timeout` 对于 `ping` 来说指的是每个 `nameserver` 的超时时间，我们设置了 3 个无效的 DNS ，超时时间为 3 ，因此尝试（轮询）一次的时间为 `3 x 3 =9` , 两次尝试应该是 18 s 才对，为什么看到的却是 36 呢？这是因为 `ping` 在进行一次解析的时候会做两个不同的记录的询问请求，一个是 A 记录，另一个是 PTR 记录。当然 PTR 记录是得到 A 记录之后再做的，如果没有得到 A 记录就会再请求一次 A 记录。


我们不妨设置两个 DNS，第一个无效，第二个设为有效的来看看：
```c
[root@bogon ~]# cat /etc/resolv.conf
options timeout:3
options attempts:2
nameserver 123.45.67.1
nameserver 8.8.8.8
[root@bogon ~]# 
[root@bogon ~]# time ping -c 3  www.baidu.com
PING www.a.shifen.com (61.135.169.125) 56(84) bytes of data.
64 bytes from 61.135.169.125 (61.135.169.125): icmp_seq=1 ttl=54 time=28.4 ms
64 bytes from 61.135.169.125 (61.135.169.125): icmp_seq=2 ttl=54 time=28.3 ms
64 bytes from 61.135.169.125 (61.135.169.125): icmp_seq=3 ttl=54 time=28.3 ms

--- www.a.shifen.com ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 4084ms
rtt min/avg/max/mdev = 28.356/28.378/28.405/0.195 ms

real    0m7.215s
user    0m0.002s
sys     0m0.004s
```
```c
[root@bogon ~]# tcpdump -nn -i eth0   icmp or udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
18:11:43.480899 IP 124.160.121.68.60040 > 123.45.67.1.53: 29894+ A? www.baidu.com. (31)
18:11:46.483955 IP 124.160.121.68.57979 > 8.8.8.8.53: 29894+ A? www.baidu.com. (31)
18:11:46.579031 IP 8.8.8.8.53 > 124.160.121.68.57979: 29894 3/0/0 CNAME www.a.shifen.com., A 61.135.169.125, A 61.135.169.121 (90)
18:11:46.579360 IP 124.160.121.68 > 61.135.169.125: ICMP echo request, id 14864, seq 1, length 64
18:11:46.607733 IP 61.135.169.125 > 124.160.121.68: ICMP echo reply, id 14864, seq 1, length 64
18:11:46.607865 IP 124.160.121.68.36510 > 123.45.67.1.53: 16127+ PTR? 125.169.135.61.in-addr.arpa. (45)
18:11:49.610912 IP 124.160.121.68.53473 > 8.8.8.8.53: 16127+ PTR? 125.169.135.61.in-addr.arpa. (45)
18:11:49.660012 IP 8.8.8.8.53 > 124.160.121.68.53473: 16127 NXDomain 0/1/0 (97)
18:11:49.662472 IP 124.160.121.68 > 61.135.169.125: ICMP echo request, id 14864, seq 2, length 64
18:11:49.690812 IP 61.135.169.125 > 124.160.121.68: ICMP echo reply, id 14864, seq 2, length 64
18:11:50.663851 IP 124.160.121.68 > 61.135.169.125: ICMP echo request, id 14864, seq 3, length 64
18:11:50.692213 IP 61.135.169.125 > 124.160.121.68: ICMP echo reply, id 14864, seq 3, length 64
```

分析：
`124.160.121.68.60040 > 123.45.67.1.5`
向第一个 `DNS` 开始第一次尝试，做第一件事：询问 A 记录。第一个 `DNS` 无效，在等待超时时间 `options timeout:3` 也就是 3 秒之后，向第二个 DNS 第一次尝试，做第一件事：询问 A 记录：`124.160.121.68.57979 > 8.8.8.8.53`。第二个 `DNS` 有效，并返回了 A 记录 ，立刻发送了一个 ICMP 包并且得到了响应：
```c
18:11:46.579031 IP 8.8.8.8.53 > 124.160.121.68.57979: 29894 3/0/0 CNAME www.a.shifen.com., A 61.135.169.125, A 61.135.169.121 (90)
18:11:46.579360 IP 124.160.121.68 > 61.135.169.125: ICMP echo request, id 14864, seq 1, length 64
18:11:46.607733 IP 61.135.169.125 > 124.160.121.68: ICMP echo reply, id 14864, seq 1, length 64
```
得到百度的响应之后并没有继续发送剩余的 ICMP 报文，而是又向第一个 DNS 请求了 PTR 记录的解析，注意这里并不是进行了第二次尝试，而是在做第一次尝试中的第二件事情：询问 PTR 记录。
```c
18:11:46.607865 IP 124.160.121.68.36510 > 123.45.67.1.53: 16127+ PTR? 125.169.135.61.in-addr.arpa. (45)
```
第一个 `DNS` 无效，在等待超时时间 `options timeout:3` 也就是 3 秒之后，向第二个DNS 请求做第一次尝试中的第二件事情：询问 PTR 记录
```c
18:11:49.610912 IP 124.160.121.68.53473 > 8.8.8.8.53: 16127+ PTR? 125.169.135.61.in-addr.arpa. (45)
18:11:49.660012 IP 8.8.8.8.53 > 124.160.121.68.53473: 16127 NXDomain 0/1/0 (97)
```
最后，发送剩余的两个 ICMP 报文。
我们再算一下时间：
两个 DNS ，每个 DNS 都进行了一次尝试，做了两个不同的事情（询问A记录，再询问 PTR 记录），第一个 DNS 尝试一次的过程中超时两次时间是 6 秒，第二个 DNS 未超时，总共加起来是 7 秒左右。
因此对于 `ping` 来说一次尝试的超时时间应该是 `（PTR的超时时间 + A 记录的超时时间） * 无法解析的 nameserver 的个数`。


### rotate
为了避免 DNS 查询每次都从第一个 `nameserver` 开始，来均衡各个 `nameserver` 的压力。如果第一个 `nameserver` 失效时，使用这个选项就可以提高解析的效率。
![](attachments/Pasted%20image%2020231107192403.png)

>注：要注意很多云主机上都有自己分配好的 DNS ，如果此时用了 `rotate` 将会在一定程度上影响效率，因为对于云主机来说，自家的 DNS 肯定比公共的响应要快。

#### 范例
```c
[root@bogon ~]# cat /etc/resolv.conf
options rotate
nameserver 123.45.67.1
nameserver 123.45.67.2
nameserver 8.8.8.8
[root@bogon ~]# 
[root@bogon ~]# time ping www.baidu.com -c 5 -q
PING www.a.shifen.com (61.135.169.121) 56(84) bytes of data.

--- www.a.shifen.com ping statistics ---
5 packets transmitted, 5 received, 0% packet loss, time 4000ms
rtt min/avg/max/mdev = 27.947/28.093/28.183/0.169 ms

real    0m9.137s
user    0m0.001s
sys     0m0.004s
```
```c
[root@bogon ~]# tcpdump -nn -i eth0   udp port 53
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth1, link-type EN10MB (Ethernet), capture size 65535 bytes
20:23:05.883477 IP 124.160.121.68.51976 > 123.45.67.2.53: 29551+ A? www.baidu.com. (31)
20:23:10.888528 IP 124.160.121.68.39697 > 8.8.8.8.53: 29551+ A? www.baidu.com. (31)
20:23:10.988607 IP 8.8.8.8.53 > 124.160.121.68.39697: 29551 3/0/0 CNAME www.a.shifen.com., A 61.135.169.121, A 61.135.169.125 (90)
20:23:11.017108 IP 124.160.121.68.55712 > 8.8.8.8.53: 48423+ PTR? 121.169.135.61.in-addr.arpa. (45)
20:23:11.067780 IP 8.8.8.8.53 > 124.160.121.68.55712: 48423 NXDomain 0/1/0 (97)
```

### edns0
![](attachments/Pasted%20image%2020231107193204.png)
修改原来的 DNS 协议，让其可以传输超过 512 字节的报文限制，但是必须客户端和服务端同时支持 edns0 才能使用该协议。glibc 2.6 及以上可以支持。

### single-request-reopen
![](attachments/Pasted%20image%2020231107192906.png)
自从 CentOS6 之后，准确的说是 glibc >= 2.9 之后，Linux 系统 DNS 解析器（resolver）会使用相同的 socket 去请求 A(for IPv4) & AAAA(for IPv6) 记录解析。比如在存在防火墙等机制的网络环境中，同样源目的 ip，同样源目的 port，同样的第4层协议的连接，会被防火墙看成是同一个会话，因此会存在返回包被丢弃现象，过程如下：
![](attachments/Pasted%20image%2020231107192938.png)

默认的 DNS 解析过程是这样的：

- 主机从一个随机的源端口（图中为 12345 ），请求 DNS 的 A 记录
- 主机从同一个源端口，请求 DNS 的 AAAA 记录
- 主机先收到 DNS 返回的 AAAA 记录（也可能先收到 A 记录）
- 防火墙（可能是其他硬件设备，官方给出的描述是： Some hardware ）认为本次交互通信已经完成，关闭连接
- 剩下的 DNS 服务器返回的 A 记录响应包被防火墙丢弃

如果我们启用参数 `single-request-reopen` （默认未启用），一旦出现同一个 socket 发送的两次请求处理，解析端发送第一次请求后会关闭 socket，并在发送第二次请求前打开新的 socket 对 DNS server 端进行发送解析请求
![](attachments/Pasted%20image%2020231107193101.png)

生产环境中发现在阿里云的主机上可能会出现上述问题，因此如果主机使用了 IPv6 建议启用该参数。

# 总结和建议
文件 `/etc/resolv.conf` 中参数的有效性，准确的说应该视系统命令或应用程序而定的。例如：

- 对于 `ping` 和 `dig` 来说他们会最多轮询三个 `nameserver` ，但是 `curl` 则是轮询所有的 `nameserver`
- 当 `attempts` 为小数时， `nslookup` 和 `dig` 会提示该文件语法错误
- 当 `attmepts` 为 0 时，对于 `ping` 来说则不会进行 DNS 解析请求，但对于 `dig` 和 `nslookup` 来说依然发送 DNS 解析请求
- 一旦设定 `timeout` 参数值，系统会直接忽略掉 `dig` 自身的 `timeout`

为了避免单个 DNS 不可用时导致解析瘫痪，最好配置两个或多个 `nameserver`，一般最好配置三个，适当调整 `timeout` 和 `attempts` 提高解析效率。

配置示例：
```c
options timeout:1
options attempts:2
options rotate
options single-request-reopen
nameserver 8.8.8.8
nameserver 9.9.9.9
nameserver 114.114.114.114
```
# 什么会修改 resolv.conf
在维护的主机数量较多的时候，系统中 DNS 的配置我们应该做到统一化，随意修改或更新会造成管理混乱。比如统一只靠人为干预的方式来修改 `/etc/resolv.conf` ，并将其他可能会更新该文件的服务或配置禁用掉。因此我们需要找到可能会更新 `/etc/resolv.conf` 的来源：

- 用户：这个不必多说了，有权限的用户都可以配置该文件
- 服务或程序(比如 ：`NetworkManager` or 网卡配置文件中的 DNS配置, 如 `dhclient-script` 脚本)

## 范例
```c
[root@bogon ~]# cat /etc/resolv.conf 
nameserver 8.8.8.8

[root@bogon ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
BOOTPROTO="dhcp"
ONBOOT="yes"

[root@bogon ~]# ifdown eth0;ifup eth0
Device 'eth0' successfully disconnected.
Connection successfully activated (D-Bus active path: /org/freedesktop/NetworkManager/ActiveConnection/4)

[root@bogon ~]# cat /etc/resolv.conf 
# Generated by NetworkManager
search localdomain
nameserver 192.168.127.2
nameserver 114.114.114.114
```

实验后文件中都会有个注释：`# Generated by NetworkManager`，查看进程后发现系统运行了一个服务 `NetworkManager` 为了排除其因素我们将此服务停止再次测试
```c
systemctl stop NetworkManager
```
先不配置 `PEERDNS` 选项，即默认`PEERDNS=yes`
```c
[root@bogon ~]# systemctl stop NetworkManager

[root@bogon ~]# cat /etc/resolv.conf
nameserver 8.8.8.8

[root@bogon ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
BOOTPROTO="dhcp"
ONBOOT="yes"

[root@bogon ~]# ifdown eth0;ifup eth0
Determining IP information for eth0... done.

[root@bogon ~]# cat /etc/resolv.conf 
; generated by /usr/sbin/dhclient-script
search localdomain
nameserver 192.168.127.2
```

配置 `PEERDNS` 选项，即`PEERDNS=no`
```c
[root@bogon ~]# cat /etc/resolv.conf
nameserver 8.8.8.8

[root@bogon ~]# cat /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
BOOTPROTO="dhcp"
PEERDNS="no"
ONBOOT="yes"

[root@bogon ~]# ifdown eth0;ifup eth0
Determining IP information for eth0... done.

[root@bogon ~]# cat /etc/resolv.conf 
nameserver 8.8.8.8
```
## 其他
NetworkManager 是用于便携式计算机和其他可移动计算机的理想解决方案。提供了完善而直观的用户界面，可使用户轻松地切换其网络环境。也就是说 NetworkManager 更适用于桌面环境的 PC 电脑，服务器上一般都是命令行界面，我们完全可以不需要使用 NetworkManager ，而且 NetworkManager 服务和 network 服务有可能会起冲突，因此我们可以将此服务禁掉。

查看了 `NetworkManager` 的服务配置文件 `/etc/NetworkManager/NetworkManager.conf` 之后，给出了 `See "man 5 NetworkManager.conf" for details.` 的提示，通过查看 man 手册发现了与 DNS 相关的配置：
![](attachments/Pasted%20image%2020231107195204.png)

## 小结

为了避免其他来源覆盖本机的 `/etc/resoolv.conf`，可以从以下几处来进行优化：

- 网卡配置文件中删除 `DNS{1,2}=address` 相关的配置，并添加 `PEERDNS=no`
- 网卡配置文件中添加 `NM_CONTROLLED=no` 来避免 `network.service` 和 `NetworkManager.service` 冲突
- 修改 `NetworkManager` 的处理模式，在 `/etc/NetworkManager/NetworkManager.conf` 中添加 `dns=none`
- 如果没有必要的话，则停止 `NetworkManager.service` ，并禁止开机自启动
# 参考
```c
# CentOS中resolv.conf的配置实验
https://linuxgeeks.github.io/2016/04/11/110119-CentOS%E4%B8%ADresolv.conf%E7%9A%84%E9%85%8D%E7%BD%AE%E5%AE%9E%E9%AA%8C/


```