```table-of-contents
```
# 前言
dnsmasq支持dns、dns缓存、dhcp、tftp等服务，本文将使用dnsmasq配合国内白名单，实现国内外分流解析，拿到最优的解析节点，提升访问效率。
大致架构如下：
![](attachments/Pasted%20image%2020240123143555.png)

# 解析流程
**查询流程及优先级**
先查找`hosts`文件，再查找`/etc/dnsmasq.d/*.conf`，之后查找`/etc/dnsmasq.conf`。

是否查找`hosts`，还能通过`no-hosts`来定义，`no-hosts`表示不查找`hosts`文件。

因此，如果你想让`dnsmasq`本身提供解析服务，且无需去上游DNS查询，或者说你要做任意域名的DNS解析，就可以将记录写到上面任意一个文件上。
conf的语法形如：`address=/test.com/192.168.1.1`；
hosts则遵循hosts文件的语法：`192.168.1.1 test.com`。

# 参数
## 参数说明
![](attachments/Pasted%20image%2020240123143809.png)

## 参数配置
### 基础配置
可以在`dnsmasq.conf`配置如下参数：
```bash
log-queries
log-facility=/var/log/dnsmasq.log
no-hosts
bogus-nxdomain=119.29.29.29
cache-size=1000
port=53
```

在`/etc/resolv.conf`定义上游DNS解析：
```bash
nameserver 8.8.8.8
nameserver 8.8.4.4
```
需要注意的是，resolv.conf文件最多可以定义3个DNS服务器；
![](attachments/Pasted%20image%2020240123145121.png)
如果想让dnsmasq配置三个以上的上游DNS服务器，则可以在`dnsmasq.conf`文件中通过参数`resolv-file=xxx.conf`自定义读取文件即可。
### 拓展配置
如果设置`resolv.conf`，也会影响运行dnsmasq的本机器的请求。
对于`dns query`要去`resolv.conf`定义的上游DNS查找，那么如果想让服务器本身也走`dnsmasq`，则可以在`resolv.conf`文件中将DNS设置为本地地址：
```bash
nameserver ::1 
nameserver 127.0.0.1
```

之后在`dnsmasq.conf`定义上游DNS即可，如：
```bash
all-servers
server=8.8.8.8
server=8.8.4.4
server=1.1.1.1
```
`all-servers`表示从以下dns列表中查找，选择回应最快的一条作为查询结果返回，如果非53端口，则可以通过增加`#port`来自定义端口，如`server=8.8.8.8#25533`。

前面说过，如果不想影响本机器的配置，则可以通过`resolv-file`参数来自定义指定文件。


### 检查语法
配置后可使用`dnsmasq --test`检查语法。
![](attachments/Pasted%20image%2020240123144346.png)

# 国内外分流
## 思想
使用[dnsmasq-china-list](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fgithub.com%2Ffelixonmars%2Fdnsmasq-china-list&source=article&objectId=2161170)作为大陆域名白名单，定义国内域名使用的上游DNS，不匹配的则走dnsmasq定义的上游DNS，完美利用解析优先级机制。

## 配置
### `dnsmasq-china-list` 介绍
![](attachments/Pasted%20image%2020240123144741.png)
![](attachments/Pasted%20image%2020240123144850.png)

### 克隆项目
将项目克隆到本地：
```bash
cd /opt 
git clone https://github.com/felixonmars/dnsmasq-china-list
```

### 超链到`dnsmasq.d`目录
你可以选择直接将上面几个文件从`dnsmasq-china-list`目录中拷贝到`dnsmasq.d`目录下，但考虑到这个项目的文件是定时更新维护的，因此超链接的方式更方便，后续只需定时执行`git pull`更新项目文件即可，无需重新拷贝。
```bash
ln -sf /opt/dnsmasq-china-list/accelerated-domains.china.conf  /etc/dnsmasq.d/accelerated-domains.china.conf 
ln -sf /opt/dnsmasq-china-list/google.china.conf /etc/dnsmasq.d/google.china.conf
ln -sf /opt/dnsmasq-china-list/apple.china.conf /etc/dnsmasq.d/apple.china.conf
ln -sf /opt/dnsmasq-china-list/bogus-nxdomain.china.conf /etc/dnsmasq.d/bogus-nxdomain.china.conf 
```
![](attachments/Pasted%20image%2020240123145049.png)
如上所示，显示了指定域名对应的DNS服务器。

### 替换LDNS
可选项。
上一步可见国内域名默认都是指定114的DNS作为上游，你可以选择替换为运营商分配给你的LDNS，即本地出口DNS，LDNS可以通过[腾讯的华佗诊断分析系统查询](https://cloud.tencent.com/developer/tools/blog-entry?target=https%3A%2F%2Fping.huatuo.qq.com%2F&source=article&objectId=2161170)。
![](attachments/Pasted%20image%2020240123145551.png)

假定LDNS为113.87.49.47，那么替换命令可以这么写：
```bash
sed -i 's|114.114.114.114|113.87.49.47|g' accelerated-domains.china.conf
```
![](attachments/Pasted%20image%2020240123145327.png)

### 定时更新dnsmasq-china-list
可选项。
定时更新只为保障列表更全面更稳定，可配合`crond`定时任务实现。
脚本逻辑很简单：
```bash
#!/bin/bash
cd /opt/dnsmasq-china-list
git pull
systemctl restart dnsmasq.service
```
之后通过crond配置定时任务，每6小时更新一次：
```bash
0 */6 * * * /bin/bash /server/scripts/update-china-list.sh
```
![](attachments/Pasted%20image%2020240123145716.png)

## 验证
### 日志验证
通过以上配置后，最终来验证一下：
![](attachments/Pasted%20image%2020240123145838.png)
根据查询日志可见，按照预定的轨道进行解析，国内外使用不同的上游DNS查询，并且缓存一份到本地。

### 抓包验证
![](attachments/Pasted%20image%2020240123145936.png)
通过报文情况可以看到，国外域名第一次查询往往会比较久，因为物理链路距离较长，涉及跨境传输，后面的查询将结果缓存到本地后，则无需再去请求上游DNS，直接命中缓存返回A记录，通过dig命令前后两次对比也可以直观看出。

# 参考
```bash
# dnsmasq高阶配置详解 - 国内外域名分流解析
https://cloud.tencent.com/developer/article/2161170
```