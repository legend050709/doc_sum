```table-of-contents
```
# 背景
## 缓存的优点
本地缓存DNS解析信息，提供解析速度。这消除了对远程 DNS 服务器重复查询的需要，并允许你的 OS 或浏览器快速解析网站的 URL。
DNS服务挂了也没有问题，在缓存服务时间范围内，解析依旧正常。

注：此中的本地其实是 DNS服务器，有些Server将自身作为DNS服务器。

## 缓存的缺点
DNS解析信息会滞后，如域名解析更改需要手动刷新缓存。对于依赖DNS切换的服务，建议不要开启DNS缓存。

刷新dns缓存非常简单，任何时候都以进行。但是不同的系统，Windows、Mac OS和Linux上的方法是不一样的。
![](attachments/Pasted%20image%2020231017173054.png)
# 系统级别刷新
## windows系统
要在 Windows 10 和 Windows 8 中清除 DNS 缓存，请执行以下步骤：

1. 在 Windows 搜索栏中键入 cmd 。
    
2. 右键单击 “命令提示符”，然后右击 “以管理员身份运行”。这将打开 “命令提示符” 窗口。
    
3. 在命令行上，键入以下行，然后按回车：
```c
ipconfig /flushdns
```

## linux系统
在 Linux 上，除非已安装并运行诸如 `Systemd-Resolved`，`DNSMasq` 或 `Nscd` 之类的缓存服务，否则没有操作系统级 `DNS` 缓存。

### nscd
NSCD(name service cache daemon)是一个缓存守护程序，是大多数基于RedHat的Linux操作系统的首选DNS缓存系统，比如OpenSUSE Linux、CentOS就用此方式来刷新DNS。

如果是清除 nscd 上的 Cache，可重新启动 nscd 服务来达成清除 DNS Cache 的效果：
```

1》 查看是否启动了nscd
systemctl status nscd.service

2》运行systemctl restart命令，它将重新加载服务并自动清除DNS缓存
systemctl restart nscd
```
如果 nscd 服务不存在，先安装 nscd，命令如下：
```c
sudo yum install nscd
```
### systemd-resolve
大多数Linux用户正在运行一个内置Systemd init系统的操作系统，每个人都知道有一件事：Systemd使得操作系统级别的深度调整和维护比以往更加容易，清除DNS缓存时尤其如此。
Systemd以称为“systemd resolved”的方式处理DNS缓存，它是一个标准实用程序.


使用 resolvectl 命令刷新 DNS 缓存：
```c
# 查看是否启动 Systemd Resolved
systemctl status systemd-resolved.service

# Step 1. 查看 DNS 缓存状况  
sudo resolvectl --statistics  
  
# Step 2. 清除 DNS 缓存  
systemd-resolved --flush-caches

如果上诉方法不行，则 systemctl restart systemd-resolved.service。
  
# Step 3. 正在查看验证结果 (sysin)  
sudo resolvectl --statistics
```
如果 resolvectl 无法执行，先安装 systemd-resolved，命令如下：
```c
yum install systemd-resolved
```
### dnsmasq
Dnsmasq 是轻量级的 DHCP 和 DNS 缓存名称服务器。它可以应用在内部网和 Internet 连接的时候的 IP 地址 NAT 转换，也可以用做小型网络的 DNS 服务。

如果你的 DNS 服务器是用 dnsmasq 实现的，用下面这个命令:
```
1》 查看是否运行
systemctl status dnsmasq.service

2》可以使用systemctl restart命令立即清除DNSMasq的DNS缓存
systemctl restart dnsmasq
```
如果 dnsmasq 服务不存在，先安装 dnsmasq，命令如下：
```c
yum install dnsmasq
```

### bind之named
如果是清除 BIND（Berkeley Internet Name Domain） 服务器上的 CACHE，用这个命令:
```c
1> 查看DNS Cache
rndc -dumpdb

2> 清空DNS Cache:
rndc flush
or 
重启 named进程：systemctl restart named
or
重启 rndc: rndc restart

3> 清楚特定的域名缓存：
rndc flushname 2daygeek.com

```
如果 rndc 无法执行，先安装 bind，命令如下：
```c
yum install bind
```
### unboud
**unbound**：一个高性能的递归 DNS 解析器，可以替代系统默认的解析器。安装 unbound 并进行相应配置后，它会负责处理 DNS 查询并缓存结果。
unbound 使用 unbound-control 命令来管理 DNS 缓存：
```c
# 刷新所有缓存  
unbound-control flush all  
# 更多命令查看帮助  
unbound-control -h
```

如果 unbound-control 无法执行，先安装 unbound，命令如下：
```c
yum install unbound
```

# 浏览器级别刷新
大多数现代的 Web 浏览器都有一个内置的 DNS 客户端，以防止每次访问该网站时重复查询。

## 谷歌浏览器 Chrome
要清除 Google Chrome 的 DNS 缓存，请执行以下步骤：

1. 打开一个新标签，然后在地址栏输入 `chrome://net-internals/#dnsChrome`。
    
2. 点击 “清除主机缓存” 按钮。

此方法适用于所有基于 Chrome 的浏览器，包括 Chromium，Vivaldi 和 Opera。

# 参考
```c
https://leokongwq.github.io/2017/08/30/linux-clean-dns-cache.html

https://www.hi-linux.com/posts/56208.html
```
