```table-of-contents
```
# 基础知识
互联网基于 TCP/IP 协议。为了方便管理网络内的主机，整个互联网分为若干个[域](https://en.wikipedia.org/wiki/Domain_name) （domain），每 个域又可以再分为若干个子域，例如，`.com`，`.org`，`.edu` 都是顶级域，而 `google.com` 是`.com` 下面的子域。

网络中的任意一台主机（host）都会属于某个域，并且有自己的名字，称为主机名（ hostname）。例如 `example.com` 就是`.com` 域中一台主机名为 `example.com`的主机。
域名/主机名是为了方便人记忆，而机器之间通信最终用的还是 IP 地址，因此需要一个将主 机名（域名）转换成 IP 地址的服务。域名服务系统（DNS, domain name system）做的就是 这个事情，对应的服务器称为域名服务器（Domain Name Server）。

例如，当通过浏览器访问[example.com](https://arthurchiao.art/blog/dns-practice-zh/example.com)，浏览器会首先访问 DNS 服务器，查找 `example.com` 对应的 IP 地址，然后和这个 IP 建立 TCP 连接，接下来才发起 HTTP 请求。

一个域名可以对应一个 IP 地址，也可以对应多个。对于后者，DNS 服务算法会从中选择一个 地址返回。大部分网络服务为了实现高可用，都是对应多个地址，我们后面会看到， baidu.com 就对应多个 IP。

# 问题
有一些场景会导致访问 DNS 服务不稳定，例如 DNS 服务器的设置有问题、网络有丢包、主机 DNS 配置错误等等。我们接下来查看几种 case。

# DNS配置
Linux 上的 DNS 配置在`/etc/resolv.conf` 里面。我们先来查看机器的配置：
```c
# cat /etc/resolv.conf
search xxx.com
nameserver 10.6.6.6
nameserver 10.10.10.10
options timeout:1
options attempts:1
options rotate
```
# DNS 问题排查
## 机器未配置 DNS 服务器导致域名查找失败
- **现象**：网络是通的（例如 ping IP 通），但是 DNS 查询总是失败
- **可能的原因**：机器没有配置 DNS 服务器
- **解决办法**：修改`/etc/resolv.conf`，给机器配置合适的 DNS 服务器

在正常的机器里用 `nslookup` 工具查看域名对应的 IP 地址：

```
/ # nslookup example.com

Name:      example.com
Address 1: 93.184.216.34
Address 2: 2606:2800:220:1:248:1893:25c8:1946
```
将`/etc/resolv.conf` 里的 DNS 服务器列表用`#`注释掉，模拟没有配置 DNS 服务器的场景。

再次测试：

```
/ # nslookup example.com

nslookup: can't resolve 'example.com': Try again
```

所以遇到这种问题，可以先去排查`/etc/resolv.conf` 里面是否配置了 DNS 服务器。

## DNS 服务太慢
- **现象**：DNS 查询太慢
- **可能的原因**：配置的 DNS 服务器不合理
- **解决办法**：修改`/etc/resolv.conf`，配置合适的 DNS 服务器

每个公司一般都有自维护的 DNS 服务器，不仅用来解析内网 DNS，而且可以加速解析公网域名 。
首先查看使用内网 DNS，查询域名的延迟：
```
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 0 msec
;; SERVER: 192.168.1.11#53(192.168.1.11)
```

可以看到非常快，在 `1ms` 以内。
然后我们测试如果使用 Google 的公网 DNS 服务器 `8.8.8.8` [1]，延迟会是多少。

修改`/etc/resolv.conf`，将其他 `nameserver` 注释掉，添加一行 `nameserver 8.8.8.8`。

再次测试：

```
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 150 msec
;; SERVER: 8.8.8.8#53(8.8.8.8)
```

延迟变成了 `150ms`，比原来大了 150 多倍。

因此，对于 DNS 查询特别慢的场景，首先要查看配置的 DNS 服务器是否合理。

## hardcode `/etc/hosts` 导致跳过 DNS 查询
- **现象**：某域名访问太慢、某域名总是指向相同 IP（多 IP 情况下）、特定机器不可访问 某域名等等
- **可能的原因**：`/etc/hosts` 有 hardcode 域名及 IP
- **解决办法**：修改`/etc/hosts`

大部分公网域名都对应多个 IP 地址，因此每次 DNS 查询拿到的 IP 地址都可能不一 样，我们用 ping 来测试一下：
```
/ # ping baidu.com
PING baidu.com (220.181.57.216): 56 data bytes
64 bytes from 220.181.57.216: seq=0 ttl=45 time=26.895 ms
64 bytes from 220.181.57.216: seq=1 ttl=45 time=26.701 ms
^C

/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.587 ms
64 bytes from 123.125.115.110: seq=1 ttl=43 time=27.757 ms
^C
```

可以看到，两次 ping 测试（内部首先查询 baidu.com 对应的 IP 地址）拿到的 IP 地址是不一样 的。用 `nslookup` 可以看到它们都是 baidu.com 对应的 IP 地址：

```
/ # nslookup baidu.com
Name:   baidu.com
Address: 220.181.57.216
Name:   baidu.com
Address: 123.125.115.110
```

`/etc/hosts` 里面可以直接 harcode 一个域名对应的 IP 地址，这会导致机器跳过 DNS 查询，直接拿这个 IP 作 为该域名的 IP。我们来验证一下。

修改`/etc/hosts`，添加一行 `123.125.115.110 baidu.com`，再次 ping 测试
```
/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.861 ms
^C
--- baidu.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 27.861/27.861/27.861 ms
/ # ping baidu.com
PING baidu.com (123.125.115.110): 56 data bytes
64 bytes from 123.125.115.110: seq=0 ttl=43 time=27.614 ms
^C
```

这是不管执行多少次，baidu.com 对应的 IP 地址都不会变了。而实际上，这个 IP 地址并不一定是最优的 IP 地址，甚至有可能这 个 IP 不可用，导致访问 baidu.com 失败。因此，实际中要极力避免在`/etc/hosts` 中 hardcode。

## DNS 查询不稳定
- **现象**：DNS 查询不稳定，时快时慢
- **可能的原因**：机器上有 `tc` 或 `iptables` 规则，导致到 DNS 服务器的 packet 变慢或丢 失
- **解决办法**：修改或删除 `tc`/`iptables` 规则

我们用 `tc` 来模拟网络延迟：
首先查看有没有 tc 规则：

```
/ # tc -p qdisc ls dev eth0
```

默认没有任何规则。

然后我们加一条：每个 packet 延迟 600ms：

```
/ # tc qdisc add dev eth0 root netem delay 600ms

/ # tc -p qdisc ls dev eth0
/ # qdisc netem 8001: root refcnt 2 limit 1000 delay 600.0ms
```
测试：

```
/ # dig example.com
...
example.com.            15814   IN      A       93.184.216.34

;; Query time: 600 msec
;; SERVER: 192.168.1.11#53(192.168.1.11)
```

可以看到，DNS 查询变成了 600ms。

这里我们测试的是**固定**延迟，这种问题很容易发现。我们还可以测试随机延迟，或者按 比例延迟等 [2]：

```
/ # tc qdisc change dev eth0 root netem delay 600ms 10ms 25%
/ # tc qdisc change dev eth0 root netem delay 600ms 20ms distribution normal
```

此类规则会导致 DNS 查询速度更有随机性。

最后删除 tc 规则：

```
/ # tc qdisc del dev eth0 root
```

`iptables` 规则也会导致类似的问题。

很多软件在运行之后，会在宿主机上添加 `tc` 或 `iptables` 规则，例如 OpenStack，K8S 等等 。因此遇到这种随机延迟问题，首先可以查看机器上是否有 `tc` 或 `iptables` 规则。

## DNS 反向查询不稳定
线上遇到过这样一个问题：从一台机器 ping 一个内网域名，每个 ping 包看起来都会卡 5～30s 不等，但是 CTL-C 关闭 ping 之后，打印出来的统计信息里，既没有丢包，ping 的延迟也很低 （毫秒级），这就很奇怪。

接下来：

1. `dig <URL>`，很快，毫秒级，说明 DNS 查询没有问题
2. `dig` 能看到域名对应的 IP，直接 ping 这个 IP，发现是没有卡顿的
3. 仍然 ping 域名，用 tcpdump 抓包，`tcpdump -i eth0 host <URL> and icmp`，发现 ping 包都是立即响应的，印证了统计信息里，ping 延迟很低的事实。

根据以上信息，说明 ping 卡顿的问题出在这台机器，而且应该就是 ping 程序本身在做什么耗 时的操作。继续：

1. 仍然 ping 域名，同时，用 `ltrace -p <PID>`跟踪 ping 进程，发现卡在一个叫 `gethostbyaddr()`的函数
2. 查阅文档，发现这个函数是根据 IP 反向查询 hostname，需要和 DNS 交互。

到这里，基本确定了是 DNS 服务器反向查询的问题，我们用另外几个命令行工具验证一下， 以下三个命令都是根据 IP 反查 hostname：

1. `nslookup <IP>`
2. `host <IP>`
3. `dig -x <IP>`

果然，以上三个命令都会卡住。修改`/etc/resolv.conf`，换一个 DNS 服务器之后，问题 消失了。接下来，就去查 DNS 服务器的问题吧。

## DNS配置更改不生效
- **现象**：DNS 配置更改之后，进行DNS查询依然是旧的配置
- **可能的原因**：机器上存在DNS cache
- **解决办法**：清除本机上的DNS cache

参考文章：如何刷新DNS缓存


# 参考
```c
https://arthurchiao.art/blog/dns-practice-zh/
```