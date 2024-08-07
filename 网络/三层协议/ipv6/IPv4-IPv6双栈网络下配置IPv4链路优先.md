```table-of-contents
```
# 背景

 在 IPv6+IPv4 双栈下，DNS查询会同时发送 AAAA 和 A 解析，无论访问域名有没有 AAAA 解析都会浪费一定时间去查询。**如果访问的域名同时拥有 A 和 AAAA 解析，那么 Linux 系统会优先使用 AAAA 解析，也就是 IPv6 地址**，同时网络出口的优先级都会比 IPv4 高。
 
![](attachments/Pasted%20image%2020240111141153.png)
 
 某些特定的应用和场景下，我们并不想要 IPv6 优先，这时候就需要修改一些配置文件让 IPv4 优先。另外，现在 IPv6 网络还不理想。IPv6网络不理解，是指的是跟IPv4对比，如下所示：

ipv6 连接如下所示：
```bash
# wget https://papermc.io/api/v1/paper/1.15.2/385/download
--2020-08-14 00:21:05--  https://papermc.io/api/v1/paper/1.15.2/385/download
正在解析主机 papermc.io (papermc.io)... 2606:4700:20::ac43:4580, 2606:4700:20::681a:202, 2606:4700:20::681a:302, ...
正在连接 papermc.io (papermc.io)|2606:4700:20::ac43:4580|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：44783881 (43M) [application/java-archive]
正在保存至: “download”

 2% [=>                                                                             ] 1,317,487   9.59KB/s 剩余 35m 23s
```

ipv4 连接如下所示：
```bash
# wget https://papermc.io/api/v1/paper/1.15.2/385/download
--2020-08-14 00:26:40--  https://papermc.io/api/v1/paper/1.15.2/385/download
正在解析主机 papermc.io (papermc.io)... 172.67.69.128, 104.26.3.2, 104.26.2.2, ...
正在连接 papermc.io (papermc.io)|172.67.69.128|:443... 已连接。
已发出 HTTP 请求，正在等待回应... 200 OK
长度：44783881 (43M) [application/java-archive]
正在保存至: “download”

 1% [                                                                               ] 466,050      105KB/s 剩余 6m 51s
```

**总结**：

在 IPv6+IPv4 双栈下，DNS查询会同时发送 AAAA 和 A 解析，无论访问域名有没有 AAAA 解析都会浪费一定时间去查询。如果访问的域名同时拥有 A 和 AAAA 解析，那么 Linux 系统会优先使用 AAAA 解析，也就是 IPv6 地址；只有 IPv6 无法访问的时候才会尝试访问 IPv4。

# happy eyeballs


# 双栈的影响
## dns查询可能变慢

由于**双栈**情况下域名解析会向 `localdns` 发起 `AAAA` 和`A`记录查询。
即域名解析的过程会等待AAAA记录的返回（无论是否有解析记录）；
Q: 如果无AAAA解析记录会影响什么？
A: 理论上有A记录 也就是IPv4兜底，肯定有解析。但是 dns的解析过程中，**如果 `localdns`无AAAA记录结果，会向上级域递归，直至根域查询无果。
这个过程会浪费掉一定的时间。这个时间也会算到dns解析过程中**。

故在双栈的情况下 一些域名的解析可能会缓慢，进而影响到服务。

##  网络出口IPv6优先的问题

通过IPv6访问，可能出现质量不稳定：大网环境不稳定 IPv6正常测试推进过程中，整体网络环境 不如现在的稳定。还有 被调用方支持较弱，也就是覆盖节点不全面 或者是灰度部分不重要的地域，服务质量没办法保证。

## 小结
综上，在当前情况下 服务器上及时开启双栈支持IPv6也是面临一定的问题。环境下又必须推进这件事情（国家网信办推行IPv6的大前提），那就使用折中方案。配置双栈 但是 IPv4优先。

# 方案
## 降低IPv6优先级 使得IPv4优先
### gai.conf 
库函数 `getaddrinfo` 进行域名解析时可能返回会多个A记录/AAAA记录。
根据 RFC 3484 ，系统必须对这些答案进行排序，来让成功率最高的地址优先级最高。RFC 提供了一种排序算法，但并不是全面的，RFC 还要求系统管理员可以动态修改排序，`gai.conf`(gai：get addr info的缩写 )就是修改优先级的文件。
即：`/etc/gai.conf` 文件，用于系统的 `getaddrinfo` 调用。默认情况下，它会使用 IPv6 优先。

#### 地址介绍

![](attachments/Pasted%20image%2020240111121936.png)

|Prefix|Precedence|Label|Usage|
|---|---|---|---|
|::1/128|50|0|Localhost|
|::/0|40|1|Default unicast|
|::ffff:0:0/96|35|4|IPv4-mapped IPv6 address|
|2002::/16|30|2|6to4|
|2001::/32|5|5|Teredo tunneling|
|fc00::/7|3|13|Unique local address|
|::/96|1|3|IPv4-compatible addresses (deprecated)|
|fec0::/10|1|11|Site-local address (deprecated)|
|3ffe::/16|1|12|6bone (returned)|

![](attachments/Pasted%20image%2020240111122314.png)

#### gai.conf 的格式
下面是`gai.conf`的配置文件中的每一行都包含一个关键字及其参数，并忽略任何空白的地方。以“#”开头的是注释，将会被忽略。
```text
getaddrinfo 和 getnameinfo 是彼此的反函数。它们与网络协议无关，同时支持IPv4和IPv6。它是在构建独立于协议的应用程序中进行名称解析以及将遗留 IPv4 代码转换为 IPv6 Internet 的推荐接口。

在内部，这些函数通过调用其他较低级别的函数（例如gethostbyname() ）使用域名系统(DNS) 执行解析。”

来自：getaddrinfo - Wikipedia

gai.conf
前面提到，getaddrinfo()将域名、主机名、IP地址转化为人类可读的文本形式。在使用getaddrinfo()获取到多个IP地址时，根据 RFC 3484，就需要对获取到的IP地址进行排序，即确定优先级。RFC 提供了一种排序算法，但并不是全面的，RFC 还要求系统管理员应该有可能动态更改排序。为此在Linux系统中，gai.conf就是用来干这个事情的，配置IPv4、IPv6的优先级，需要从这个文件入手。

下面是gai.conf的配置文件中的每一行都包含一个关键字及其参数，并忽略任何空白的地方。以“#”开头的是注释，将会被忽略。

# Configuration for getaddrinfo(3).
#
# So far only configuration for the destination address sorting is needed.
# RFC 3484 governs the sorting.  But the RFC also says that system
# administrators should be able to overwrite the defaults.  This can be
# achieved here.
#
# All lines have an initial identifier specifying the option followed by
# up to two values.  Information specified in this file replaces the
# default information.  Complete absence of data of one kind causes the
# appropriate default information to be used.  The supported commands include:
#
# reload  <yes|no>
#    If set to yes, each getaddrinfo(3) call will check whether this file
#    changed and if necessary reload.  This option should not really be
#    used.  There are possible runtime problems.  The default is no.
#
# label   <mask>   <value>
#    Add another rule to the RFC 3484 label table.  See section 2.1 in
#    RFC 3484.  The default is:
#
#label ::1/128       0
#label ::/0          1
#label 2002::/16     2
#label ::/96         3
#label ::ffff:0:0/96 4
#label fec0::/10     5
#label fc00::/7      6
#label 2001:0::/32   7
#
#    This default differs from the tables given in RFC 3484 by handling
#    (now obsolete) site-local IPv6 addresses and Unique Local Addresses.
#    The reason for this difference is that these addresses are never
#    NATed while IPv4 site-local addresses most probably are.  Given
#    the precedence of IPv6 over IPv4 (see below) on machines having only
#    site-local IPv4 and IPv6 addresses a lookup for a global address would
#    see the IPv6 be preferred.  The result is a long delay because the
#    site-local IPv6 addresses cannot be used while the IPv4 address is
#    (at least for the foreseeable future) NATed.  We also treat Teredo
#    tunnels special.
#
# precedence  <mask>   <value>
#    Add another rule to the RFC 3484 precedence table.  See section 2.1
#    and 10.3 in RFC 3484.  The default is:
#
#precedence  ::1/128       50
#precedence  ::/0          40
#precedence  2002::/16     30
#precedence ::/96          20
#precedence ::ffff:0:0/96  10
#
#    For sites which prefer IPv4 connections change the last line to
#
#precedence ::ffff:0:0/96  100
 
#
# scopev4  <mask>  <value>
#    Add another rule to the RFC 6724 scope table for IPv4 addresses.
#    By default the scope IDs described in section 3.2 in RFC 6724 are
#    used.  Changing these defaults should hardly ever be necessary.
#    The defaults are equivalent to:
#
#scopev4 ::ffff:169.254.0.0/112  2
#scopev4 ::ffff:127.0.0.0/104    2
#scopev4 ::ffff:0.0.0.0/96       14
配置IPv4协议优先
根据 RFC 3484协议的第10.3条款，在IPv4/IPv6双栈协议中设置IPv4优先，只需赋予 ::ffff:0.0.0.0/96 前缀更高的优先级即可。

10.3. Configuring Preference for IPv6 or IPv4
 
   The default policy table gives IPv6 addresses higher precedence than
   IPv4 addresses.  This means that applications will use IPv6 in
   preference to IPv4 when the two are equally suitable.  An
   administrator can change the policy table to prefer IPv4 addresses by
   giving the ::ffff:0.0.0.0/96 prefix a higher precedence:
 
      Prefix        Precedence Label
      ::1/128               50     0
      ::/0                  40     1
      2002::/16             30     2
      ::/96                 20     3
      ::ffff:0:0/96        100     4
 
   This change to the default policy table produces the following
   behavior:
 
   Candidate Source Addresses: 2001::2 or fe80::1 or 169.254.13.78
   Destination Address List: 2001::1 or 131.107.65.121
   Unchanged Result: 2001::1 (src 2001::2) then 131.107.65.121 (src
   169.254.13.78) (prefer matching scope)
 
   Candidate Source Addresses: fe80::1 or 131.107.65.117
   Destination Address List: 2001::1 or 131.107.65.121
   Unchanged Result: 131.107.65.121 (src 131.107.65.117) then 2001::1
   (src fe80::1) (prefer matching scope)
 
   Candidate Source Addresses: 2001::2 or fe80::1 or 10.1.2.4
   Destination Address List: 2001::1 or 10.1.2.3
   New Result: 10.1.2.3 (src 10.1.2.4) then 2001::1 (src 2001::2)
   (prefer higher precedence)
编辑/etc/gai.conf文件，将precedence ::ffff:0:0/96 100取消注释后保存即可。

参考资料

The client is not respecting the IPv6/IPv4 priority from gai.conf
gai.conf(5) — Linux manual page
RFC 3484
how to sort the ip adresses returning from getaddrinfo() like /etc/gai.conf in linux
Chapter 13. Address Resolver & Selection
この素晴らしい世界に祝福を！最后更新于 2022-12-18 啥也没有呀
getaddrinfo()
gai.conf
配置IPv4协议优先
上一篇文章
不是很专业的开箱——小米 Watch S1 Pro
下一篇文章
只是觉得女孩子们之间真是好啊
查看评论 - NOTHING
Copyright © 2021-2024 kanochan.net
CC BY-NC-SA 4.0   License
萌ICP备20222323号


Theme Sakurairo by Fuukei



# Configuration for getaddrinfo(3).
#
# So far only configuration for the destination address sorting is needed.
# RFC 3484 governs the sorting.  But the RFC also says that system
# administrators should be able to overwrite the defaults.  This can be
# achieved here.
#
# All lines have an initial identifier specifying the option followed by
# up to two values.  Information specified in this file replaces the
# default information.  Complete absence of data of one kind causes the
# appropriate default information to be used.  The supported commands include:
#
# reload  <yes|no>
#    If set to yes, each getaddrinfo(3) call will check whether this file
#    changed and if necessary reload.  This option should not really be
#    used.  There are possible runtime problems.  The default is no.
#
# label   <mask>   <value>
#    Add another rule to the RFC 3484 label table.  See section 2.1 in
#    RFC 3484.  The default is:
#
#label ::1/128       0
#label ::/0          1
#label 2002::/16     2
#label ::/96         3
#label ::ffff:0:0/96 4
#label fec0::/10     5
#label fc00::/7      6
#label 2001:0::/32   7
#
#    This default differs from the tables given in RFC 3484 by handling
#    (now obsolete) site-local IPv6 addresses and Unique Local Addresses.
#    The reason for this difference is that these addresses are never
#    NATed while IPv4 site-local addresses most probably are.  Given
#    the precedence of IPv6 over IPv4 (see below) on machines having only
#    site-local IPv4 and IPv6 addresses a lookup for a global address would
#    see the IPv6 be preferred.  The result is a long delay because the
#    site-local IPv6 addresses cannot be used while the IPv4 address is
#    (at least for the foreseeable future) NATed.  We also treat Teredo
#    tunnels special.
#
# precedence  <mask>   <value>
#    Add another rule to the RFC 3484 precedence table.  See section 2.1
#    and 10.3 in RFC 3484.  The default is:
#
#precedence  ::1/128       50
#precedence  ::/0          40
#precedence  2002::/16     30
#precedence ::/96          20
#precedence ::ffff:0:0/96  10
#
#    For sites which prefer IPv4 connections change the last line to
#
#precedence ::ffff:0:0/96  100
 
#
# scopev4  <mask>  <value>
#    Add another rule to the RFC 6724 scope table for IPv4 addresses.
#    By default the scope IDs described in section 3.2 in RFC 6724 are
#    used.  Changing these defaults should hardly ever be necessary.
#    The defaults are equivalent to:
#
#scopev4 ::ffff:169.254.0.0/112  2
#scopev4 ::ffff:127.0.0.0/104    2
#scopev4 ::ffff:0.0.0.0/96       14
```
#### 配置IPv4协议优先
根据 [RFC 3484](https://www.ietf.org/rfc/rfc3484.txt)协议的第10.3条款，在IPv4/IPv6双栈协议中设置IPv4优先，只需赋予 `::ffff:0.0.0.0/96` 前缀更高的优先级即可。
```text
10.3. Configuring Preference for IPv6 or IPv4
 
   The default policy table gives IPv6 addresses higher precedence than
   IPv4 addresses.  This means that applications will use IPv6 in
   preference to IPv4 when the two are equally suitable.  An
   administrator can change the policy table to prefer IPv4 addresses by
   giving the ::ffff:0.0.0.0/96 prefix a higher precedence:
 
      Prefix        Precedence Label
      ::1/128               50     0
      ::/0                  40     1
      2002::/16             30     2
      ::/96                 20     3
      ::ffff:0:0/96        100     4
 
   This change to the default policy table produces the following
   behavior:
 
   Candidate Source Addresses: 2001::2 or fe80::1 or 169.254.13.78
   Destination Address List: 2001::1 or 131.107.65.121
   Unchanged Result: 2001::1 (src 2001::2) then 131.107.65.121 (src
   169.254.13.78) (prefer matching scope)
 
   Candidate Source Addresses: fe80::1 or 131.107.65.117
   Destination Address List: 2001::1 or 131.107.65.121
   Unchanged Result: 131.107.65.121 (src 131.107.65.117) then 2001::1
   (src fe80::1) (prefer matching scope)
 
   Candidate Source Addresses: 2001::2 or fe80::1 or 10.1.2.4
   Destination Address List: 2001::1 or 10.1.2.3
   New Result: 10.1.2.3 (src 10.1.2.4) then 2001::1 (src 2001::2)
   (prefer higher precedence)
```
比如：编辑`/etc/gai.conf`文件，将`precedence ::ffff:0:0/96 100`取消注释后保存即可。
```bash
sed -i 's/#precedence ::ffff:0:0\/96  100/precedence ::ffff:0:0\/96  100/' /etc/gai.conf
```

#### 范例
安装了 `curl` 并且本地支持 IPv6，那么可以使用 `curl ip.sb` 测试
```bash
root@debian ~ # curl ip.sb
2001:db8::2
```
效果等同于 `curl ip.sb -6`
如果你不想使用 IPv6 优先，可以在这个文件中找到：
```bash
#precedence ::ffff:0:0/96  100
```
取消注释，修改为：
```bash
precedence ::ffff:0:0/96  100
```
此时再使用 `curl ip.sb` 测试
```bash
root@debian ~ # curl ip.sb
192.0.2.2
```
效果等同于 `curl ip.sb -4`

## 禁用IPv6
### sysctl.conf配置 禁用ipv6
编辑`/etc/sysctl.conf`配置，增加`net.ipv6.conf.all.disable_ipv6=1` 
执行`sysctl -p` 来生效。
### network配置，禁用ipv6
编辑`/etc/sysconfig/network`配置，增加 `NETWORKING_IPV6=no`，保存并退出。
编辑`/etc/sysconfig/network-scripts/ifcfg-eno16777736`，确保`IPV6INIT=no`，`ifcfg-eno16777736`是根据自己机器的，实际网卡信息来看，不是固定的。
重启`networkd` 服务生效。

### 禁用ipv6相关进程/模块
关闭防火墙的开机自启动  
`systemctl disable ip6tables.service`
# 设置与查看 
## 查看是否开启了ipv6
使用ifconfig命令查看网卡信息，如果出现inet6 `fe80::20c:29ff:fed0:3514`，说明机器开启了ipv6。
![](attachments/Pasted%20image%2020240111113249.png)
## 配置IPv6协议优先
打开 `/etc/gai.conf` 文件，将 `::ffff:0:0/96`（这是 IPv4 映射到 IPv6 的前缀）的优先级设为低于 IPv6 地址：
```bash
precedence ::ffff:0:0/96 100 
precedence ::/0 200
```

**检查**：
确认当前网络接口的顺序；
```bash
ip addr show
```
该命令将显示当前所有的网络接口及其 IPv4 和 IPv6 地址。请确保 IPv6 地址在前面。
注：如果没有 IPv6 地址，你需要检查系统和路由器的配置，确保 IPv6 能够正常工作。
# 参考
```c
# IPv4/IPv6双栈网络下配置IPv4链路优先
https://kanochan.net/archives/3249.html


# Debian 双栈网络时开启 IPv4 优先
https://u.sb/debian-prefer-ipv4/
```