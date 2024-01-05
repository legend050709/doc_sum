```table-of-contents
```
# 简介
Bind是`Berkeley Internet Name Domain Service`的简写，它是一款实现DNS 服务器的开放源码软件。已经成为世界上使用最为广泛的DNS服务器软件，目前Internet上半数以上的DNS服务器有都是用Bind来架设的，已经成为DNS中事实上的标准。
# BIND概述
## bind相关包介绍
- 包名：bind
- 进程：named
- 协议：dns
- 使用端口：53（tcp，udp）

**常用命令**
```bash
#检查配置文件
named-checkconf

#启动bind9
systemctl start named

#检查bind9服务状态
systemctl status named
```

查看文件所属的包
```bash
# rpm -qf /sbin/named
bind-9.11.4-16.P2.el7_8.3.x86_64
```

**相关包**：
- bind : 提供的DNS Server主软件程序包
- bind-chroot：选装，将named进程的活动范围限定在chroot目录，保证安全性。让named运行于一个jail（沙箱）模式下。
- bind-devel：与开发相关的头文件和库文件（编译安装bind时所需）
- bind-libs：被bind和bind-utils包中的程序共同用到的文件。
- bind-utils ：bind客户端程序集，包含了如dig,host,nslookup等工具。
- bind-export：
- bind-license：


查看`bind`包中的相关文件：
```bash
# rpm -ql bind-9.11.4
/etc/logrotate.d/named
/etc/named
/etc/named.conf
/etc/named.iscdlv.key
/etc/named.rfc1912.zones
/etc/named.root.key
/etc/rndc.conf
/etc/rndc.key
/etc/rwtab.d/named
/etc/sysconfig/named
/run/named
/usr/bin/arpaname
/usr/bin/named-rrchecker
/usr/lib/python2.7/site-packages/isc
/usr/lib/python2.7/site-packages/isc-2.0-py2.7.egg-info
/usr/lib/python2.7/site-packages/isc/__init__.py
/usr/lib/python2.7/site-packages/isc/__init__.pyc
/usr/lib/python2.7/site-packages/isc/__init__.pyo
/usr/lib/python2.7/site-packages/isc/checkds.py
/usr/lib/python2.7/site-packages/isc/checkds.pyc
/usr/lib/python2.7/site-packages/isc/checkds.pyo
/usr/lib/python2.7/site-packages/isc/coverage.py
/usr/lib/python2.7/site-packages/isc/coverage.pyc
/usr/lib/python2.7/site-packages/isc/coverage.pyo
/usr/lib/python2.7/site-packages/isc/dnskey.py
/usr/lib/python2.7/site-packages/isc/dnskey.pyc
/usr/lib/python2.7/site-packages/isc/dnskey.pyo
/usr/lib/python2.7/site-packages/isc/eventlist.py
/usr/lib/python2.7/site-packages/isc/eventlist.pyc
/usr/lib/python2.7/site-packages/isc/eventlist.pyo
/usr/lib/python2.7/site-packages/isc/keydict.py
/usr/lib/python2.7/site-packages/isc/keydict.pyc
/usr/lib/python2.7/site-packages/isc/keydict.pyo
/usr/lib/python2.7/site-packages/isc/keyevent.py
/usr/lib/python2.7/site-packages/isc/keyevent.pyc
/usr/lib/python2.7/site-packages/isc/keyevent.pyo
/usr/lib/python2.7/site-packages/isc/keymgr.py
/usr/lib/python2.7/site-packages/isc/keymgr.pyc
/usr/lib/python2.7/site-packages/isc/keymgr.pyo
/usr/lib/python2.7/site-packages/isc/keyseries.py
/usr/lib/python2.7/site-packages/isc/keyseries.pyc
/usr/lib/python2.7/site-packages/isc/keyseries.pyo
/usr/lib/python2.7/site-packages/isc/keyzone.py
/usr/lib/python2.7/site-packages/isc/keyzone.pyc
/usr/lib/python2.7/site-packages/isc/keyzone.pyo
/usr/lib/python2.7/site-packages/isc/parsetab.py
/usr/lib/python2.7/site-packages/isc/parsetab.pyc
/usr/lib/python2.7/site-packages/isc/parsetab.pyo
/usr/lib/python2.7/site-packages/isc/policy.py
/usr/lib/python2.7/site-packages/isc/policy.pyc
/usr/lib/python2.7/site-packages/isc/policy.pyo
/usr/lib/python2.7/site-packages/isc/rndc.py
/usr/lib/python2.7/site-packages/isc/rndc.pyc
/usr/lib/python2.7/site-packages/isc/rndc.pyo
/usr/lib/python2.7/site-packages/isc/utils.py
/usr/lib/python2.7/site-packages/isc/utils.pyc
/usr/lib/python2.7/site-packages/isc/utils.pyo
/usr/lib/systemd/system/named-setup-rndc.service
/usr/lib/systemd/system/named.service
/usr/lib/tmpfiles.d/named.conf
/usr/lib64/bind
/usr/libexec/generate-rndc-key.sh
/usr/sbin/ddns-confgen
/usr/sbin/dnssec-checkds
/usr/sbin/dnssec-coverage
/usr/sbin/dnssec-dsfromkey
/usr/sbin/dnssec-importkey
/usr/sbin/dnssec-keyfromlabel
/usr/sbin/dnssec-keygen
/usr/sbin/dnssec-keymgr
/usr/sbin/dnssec-revoke
/usr/sbin/dnssec-settime
/usr/sbin/dnssec-signzone
/usr/sbin/dnssec-verify
/usr/sbin/genrandom
/usr/sbin/isc-hmac-fixup
/usr/sbin/lwresd
/usr/sbin/named
/usr/sbin/named-checkconf
/usr/sbin/named-checkzone
/usr/sbin/named-compilezone
/usr/sbin/named-journalprint
/usr/sbin/nsec3hash
/usr/sbin/rndc
/usr/sbin/rndc-confgen
/usr/sbin/tsig-keygen
/usr/share/doc/bind-9.11.4
/usr/share/doc/bind-9.11.4/Bv9ARM.ch01.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch02.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch03.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch04.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch05.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch06.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch07.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch08.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch09.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch10.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch11.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch12.html
/usr/share/doc/bind-9.11.4/Bv9ARM.ch13.html
/usr/share/doc/bind-9.11.4/Bv9ARM.html
/usr/share/doc/bind-9.11.4/Bv9ARM.pdf
/usr/share/doc/bind-9.11.4/CHANGES
/usr/share/doc/bind-9.11.4/README
/usr/share/doc/bind-9.11.4/isc-logo.pdf
/usr/share/doc/bind-9.11.4/man.arpaname.html
/usr/share/doc/bind-9.11.4/man.ddns-confgen.html
/usr/share/doc/bind-9.11.4/man.delv.html
/usr/share/doc/bind-9.11.4/man.dig.html
/usr/share/doc/bind-9.11.4/man.dnssec-checkds.html
/usr/share/doc/bind-9.11.4/man.dnssec-coverage.html
/usr/share/doc/bind-9.11.4/man.dnssec-dsfromkey.html
/usr/share/doc/bind-9.11.4/man.dnssec-importkey.html
/usr/share/doc/bind-9.11.4/man.dnssec-keyfromlabel.html
/usr/share/doc/bind-9.11.4/man.dnssec-keygen.html
/usr/share/doc/bind-9.11.4/man.dnssec-keymgr.html
/usr/share/doc/bind-9.11.4/man.dnssec-revoke.html
/usr/share/doc/bind-9.11.4/man.dnssec-settime.html
/usr/share/doc/bind-9.11.4/man.dnssec-signzone.html
/usr/share/doc/bind-9.11.4/man.dnssec-verify.html
/usr/share/doc/bind-9.11.4/man.dnstap-read.html
/usr/share/doc/bind-9.11.4/man.genrandom.html
/usr/share/doc/bind-9.11.4/man.host.html
/usr/share/doc/bind-9.11.4/man.isc-hmac-fixup.html
/usr/share/doc/bind-9.11.4/man.lwresd.html
/usr/share/doc/bind-9.11.4/man.mdig.html
/usr/share/doc/bind-9.11.4/man.named-checkconf.html
/usr/share/doc/bind-9.11.4/man.named-checkzone.html
/usr/share/doc/bind-9.11.4/man.named-journalprint.html
/usr/share/doc/bind-9.11.4/man.named-nzd2nzf.html
/usr/share/doc/bind-9.11.4/man.named-rrchecker.html
/usr/share/doc/bind-9.11.4/man.named.conf.html
/usr/share/doc/bind-9.11.4/man.named.html
/usr/share/doc/bind-9.11.4/man.nsec3hash.html
/usr/share/doc/bind-9.11.4/man.nslookup.html
/usr/share/doc/bind-9.11.4/man.nsupdate.html
/usr/share/doc/bind-9.11.4/man.pkcs11-destroy.html
/usr/share/doc/bind-9.11.4/man.pkcs11-keygen.html
/usr/share/doc/bind-9.11.4/man.pkcs11-list.html
/usr/share/doc/bind-9.11.4/man.pkcs11-tokens.html
/usr/share/doc/bind-9.11.4/man.rndc-confgen.html
/usr/share/doc/bind-9.11.4/man.rndc.conf.html
/usr/share/doc/bind-9.11.4/man.rndc.html
/usr/share/doc/bind-9.11.4/named.conf.default
/usr/share/doc/bind-9.11.4/notes.html
/usr/share/doc/bind-9.11.4/notes.pdf
/usr/share/doc/bind-9.11.4/sample
/usr/share/doc/bind-9.11.4/sample/etc
/usr/share/doc/bind-9.11.4/sample/etc/named.conf
/usr/share/doc/bind-9.11.4/sample/etc/named.rfc1912.zones
/usr/share/doc/bind-9.11.4/sample/var
/usr/share/doc/bind-9.11.4/sample/var/named
/usr/share/doc/bind-9.11.4/sample/var/named/data
/usr/share/doc/bind-9.11.4/sample/var/named/my.external.zone.db
/usr/share/doc/bind-9.11.4/sample/var/named/my.internal.zone.db
/usr/share/doc/bind-9.11.4/sample/var/named/named.ca
/usr/share/doc/bind-9.11.4/sample/var/named/named.empty
/usr/share/doc/bind-9.11.4/sample/var/named/named.localhost
/usr/share/doc/bind-9.11.4/sample/var/named/named.loopback
/usr/share/doc/bind-9.11.4/sample/var/named/slaves
/usr/share/doc/bind-9.11.4/sample/var/named/slaves/my.ddns.internal.zone.db
/usr/share/doc/bind-9.11.4/sample/var/named/slaves/my.slave.internal.zone.db
/usr/share/man/man1/arpaname.1.gz
/usr/share/man/man1/named-rrchecker.1.gz
/usr/share/man/man5/named.conf.5.gz
/usr/share/man/man5/rndc.conf.5.gz
/usr/share/man/man8/ddns-confgen.8.gz
/usr/share/man/man8/dnssec-checkds.8.gz
/usr/share/man/man8/dnssec-coverage.8.gz
/usr/share/man/man8/dnssec-dsfromkey.8.gz
/usr/share/man/man8/dnssec-importkey.8.gz
/usr/share/man/man8/dnssec-keyfromlabel.8.gz
/usr/share/man/man8/dnssec-keygen.8.gz
/usr/share/man/man8/dnssec-keymgr.8.gz
/usr/share/man/man8/dnssec-revoke.8.gz
/usr/share/man/man8/dnssec-settime.8.gz
/usr/share/man/man8/dnssec-signzone.8.gz
/usr/share/man/man8/dnssec-verify.8.gz
/usr/share/man/man8/genrandom.8.gz
/usr/share/man/man8/isc-hmac-fixup.8.gz
/usr/share/man/man8/lwresd.8.gz
/usr/share/man/man8/named-checkconf.8.gz
/usr/share/man/man8/named-checkzone.8.gz
/usr/share/man/man8/named-compilezone.8.gz
/usr/share/man/man8/named-journalprint.8.gz
/usr/share/man/man8/named.8.gz
/usr/share/man/man8/nsec3hash.8.gz
/usr/share/man/man8/rndc-confgen.8.gz
/usr/share/man/man8/rndc.8.gz
/usr/share/man/man8/tsig-keygen.8.gz
/var/log/named.log
/var/named
/var/named/data
/var/named/dynamic
/var/named/named.ca
/var/named/named.empty
/var/named/named.localhost
/var/named/named.loopback
/var/named/slaves
```
## bind的工作端口
- 53/UDP
- 53/TCP
（其主要工作在UDP端口，如果传送的数据太大导致失败，其会以TCP模式尝试重传）
- 953/TCP : `rndc`的连接端口

## 重要文件
**程序文件**：
```c
/usr/sbin/named
/usr/sbin/named-checkconf    # 检测/etc/named.conf文件语法
/usr/sbin/named-checkzone    # 检测zone和对应zone文件的语法
/usr/sbin/rndc               # 远程dns管理工具
/usr/sbin/rndc-confgen       # 生成rndc密钥
```


**主程序目录**：/var/named
```c
/var/named/named.ca          # 根解析库
/var/named/named.localhost   # 本地主机解析库
/var/named/slaves            # 从ns服务器文件夹
```

**配置文件**：
```bash
/etc/named.conf              # bind主配置文件
/etc/named.rfc1912.zones     # 定义zone的文件
```
## 权限相关
**bind权限相关**：安装完named会自动创建用户named系统用户
# bind服务器
## 分类
 bind服务器配置角色：
- **主服务器（Primary Authoritative Name Server）**：
负责至少解析一个域内（zone）的域名，维护所负责解析的域数据库（zone文件），**可对负责的域数据进行读写操作**
-  **辅服务器（Secondary Authoritative Name Server）**：
负责从主服务器或其他辅服务器中复制相关解析库，为主服务器缓解解析压力
- **缓存服务器（Caching Name Servers）**：
不负责域名解析，仅仅作为缓存，加速解析速度
- **转发服务器（Forwarding Name Servers）**：
发现非本机负责的请求后，不再向根发起请求，而是直接转发给指定的一台或多台服务器，自身并不保存查询

## 应用
不同角色配置起到不同的功能，在实际环境中，主服务器和辅助服务器 同时也是缓存服务器和转发服务器，可以把配置参数进行结合启用，那么就相当于一台服务器多种角色。

## DNS服务器返回结果
### 答案类别
- `有查询结果`（肯定答案）
- `不存在查询结果`（否定答案）
### 有查询结果分类
- `权威回答`
    - DNS服务器自己直接负责的域返回的答案

- `非权威回答`
    - DNS服务器未负责的域，由缓存或者查询到的记录返回的答案

## 主从DNS服务器的数据复制选项
- `serial` : 序列号，即主DNS数据库的版本号。
主服务器数据库内容发生变化时，其版本号需要递增，从服务器会对比与主服务器的数据库版本号，一样的版本号就不需要更新，否则需要更新。

- `refresh` : 从服务器每多久到主服务器检查序列号的变化

- `retry` : 从服务器到服务器请求同步解析库失败时，再次发起解析请求的时间间隔，这个时间需短时刷新时间。

- `expire` : 从服务器始终联系不到主服务器时，多久之后放弃从主服务器同步数据，超过此时间后，从服务器也将停止解析。

- `negative answer` : 否定答案的缓存时长。

注意：以上选项时间单位都支持`W`,`D`,`H`,`M`,这参数定义在资源记录的文件中，位置为SOA的后面，以（）包含，其括号前后都有空格。
## 主从服务器的zone更新同步
### 动态更新(Dynamic Update)
### 通知(NOTIFY)
![](attachments/Pasted%20image%2020231225113423.png)
### 主从DNS之间的区域传送(zone transfer)
![](attachments/Pasted%20image%2020231225111500.png)
#### A full zone transfer(AXFR)
`axfr` : 全量传送，一般为从服务器第一次读取，将会传送整个数据库。
#### Incremental Zone Transfers (IXFR)
`ixfr(incremental zone transfer)`  ：增量传送，仅传送变化的数据。
![](attachments/Pasted%20image%2020231225112241.png)

### zone文件更新(静态域和动态域维护)
#### 静态域维护
前提条件：在区域配置文件中 添加 allow-update { none; }; 表示不允许动态更新。如下所示：
```shell
[root@dns1 ~]# cat /etc/named.rfc1912.zones 
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { none; };
};
```

每次配置更改区域数据库文件(zone文件)，需要手动前滚`serial number`，通知辅DNS更新。
```shell
[root@dns1 ~]# cat /var/named/host.com.zone
$TTL 600        ;10 minutes
@                       IN SOA  dns.host.com. test.qq.com. (
                                2021070705 ; serial number  需要手动+1
                                10800      ; refresh (3 hours)
                                900        ; retry (15 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS   dns.host.com.
$ORIGIN host.com.
$TTL 60 ; 1 minute
dns1                    A    192.168.10.222
dns2                    A    192.168.10.223
test                    A    192.168.10.55
dns                     A    192.168.10.222
```

#### 动态域维护nsupdate
前提条件：在区域配置文件中 添加 allow-update { acl; }; 表示根据acl指定策略进行动态更新。可填写ip地址。
```shell
[root@dns1 ~]# cat /etc/named.rfc1912.zones 
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { 192.168.10.222; };
};
```
每次配置更改区域数据库文件，不需要手动前滚`serial number`，自动通知辅DNS更新。

##### nsupdate
`nsupdate`是一个动态`DNS`更新工具，可以向DNS服务器提交更新记录的请求，它可以从区文件中添加或删除资源记录，而不需要手动进行编辑区文件。
使用 `nsupdate` 等工具进行动态配置。​ 使用`nsupdate` 不会更改区域数据库文件，而是产生了一个`jnl`的数据文件，不能使用文本编辑器打开，只能使用完全区域数据传送查看。
注：`jnl`文件（`journal`文件）是`BIND9`动态更新的时候记录更新内容所生成的日志文件。

**优缺点**
- 优点
	- 命令简单，便于记忆
	- 不用手动变更SOA的serial序列号，自动滚动
	- 不需要重启/重载BIND9服务/配置，生效快
	- 可以通过配置acl实现远程管理


- 缺点
	- jnl文件无法使用文本文件的方式打开
	- 只能依赖完全区域传送查看所有区域的记录
	- 更新操作复杂，先删再增
	- 远程管理有安全隐患，需要加强审计
	- 动态域在rndc管理上多一步


**使用方法**
```shell
#发送请求到servername服务器的port端口.如果不指定servername,nsupdate将把请求发送给当前去的主DNS服务器.
server servername [ port ]

# 添加一条资源记录
update add domain-name ttl [ class ] type data…

# 删除domain-name的资源记录.如果指定了type和data,仅删除匹配的记录。
update delete domain-name [ ttl ] [ class ] [ type [ data... ] ]


# 将要求信息和更新请求发送到DNS服务器.等同于输入一个空行
send

# 退出nsupdate工具
quit
```



**范例**
```shell
# 前提条件：允许设备动态修改配置，将以下参数修改为any或者指定ip。
allow-update { any; };

添加记录

[root@dns1 ~]# nsupdate
> server 192.168.10.222
> update add aaa.host.com 600 IN A 192.168.10.224
> send
> quit 
```
查看测试结果：
```shell
[root@dns2 ~]# dig -t AXFR host.com @192.168.10.222

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.5 <<>> -t AXFR host.com @192.168.10.222
;; global options: +cmd
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070706 10800 900 604800 86400
host.com.               600     IN      NS      dns.host.com.
aaa.host.com.           600     IN      A       192.168.10.224
dns.host.com.           60      IN      A       192.168.10.222
dns1.host.com.          60      IN      A       192.168.10.222
dns2.host.com.          60      IN      A       192.168.10.223
test.host.com.          60      IN      A       192.168.10.55
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070706 10800 900 604800 86400
;; Query time: 0 msec
;; SERVER: 192.168.10.222#53(192.168.10.222)
;; WHEN: Fri Jul 16 10:09:54 CST 2021
;; XFR size: 8 records (messages 1, bytes 234)
```

### 主从同步范例
**环境准备：**
```c
192.168.10.222 dns1.host.com 主dns服务器
192.168.10.223 dns2.host.com 辅dns服务器
```

**配置要点：**
```c
- 辅助DNS的Bind版本必须小于主DNS的软件版本。
- 主DNS named.conf里配置allow-transfer和also-notify选项
- 辅助DNS主配置文件中option段，masterfile-format text；
- 辅助DNS的配置文件里 type:slave
- 启动辅助DNS时，检查完全区域传送：dig -t axfr @192.168.10.222
- 辅助DNS不可修改主DNS配置。
```
#### 配置主DNS
配置主配置文件，添加以下字段：
- `allow-transfer { 192.168.10.223; };` 
允许本区域传输至特定的从DNS服务器。
- `also-notify { 192.168.10.223; };`
主动通知从域名服务器（辅助DNS）进行更新，在主域名服务器进行更新后，而不需要在等规定的时间后才通知从域名服务器进行更新。

主配置文件中主要修改以下字段：
```json
[root@dns1 ~]# cat /etc/named.conf 
options {
        listen-on port 53 { 192.168.10.222; };
        allow-query     { any; };
        allow-transfer { 192.168.10.223; };
        also-notify { 192.168.10.223; };       
};
```

在zone引导配置中添加正解域和反解域：
```json
[root@dns1 ~]# vi /etc/named.rfc1912.zones 
vi /etc/named.rfc1912.zones
zone "host.com" IN {
        type  master;
        file  "host.com.zone";
        allow-update { none; };
};
zone "10.168.192.in-addr.arpa" IN {
        type master;
        file "10.168.192.in-addr.arpa.zone";
        allow-update { none; };
};
```

配置区域数据库文件：
```json
[root@dns1 ~]# cd /var/named/
[root@dns1 named]# cat host.com.zone 
$TTL 600        ;10 minutes
@                       IN SOA  dns.host.com. test.qq.com. (
                                2021070705 ; serial
                                10800      ; refresh (3 hours)
                                900        ; retry (15 minutes)
                                604800     ; expire (1 week)
                                86400      ; minimum (1 day)
                                )
                        NS   dns.host.com.
$ORIGIN host.com.
$TTL 60 ; 1 minute
dns1                    A    192.168.10.222
dns2                    A    192.168.10.223
test                    A    192.168.10.55
dns                     A    192.168.10.222

[root@dns1 named]# cat 10.168.192.in-addr.arpa.zone 
$TTL 600 ;10min
@       IN      SOA     dns.host.com    17614902580@163.com (
                        2021071101      ;serial number
                        10600           ;refresh 3 hours
                        900             ;retry 15 minites
                        604800          ;expire 1 week
                        86400           ;minimum 1 day
                        )
                ns      dns.host.com.
$ORIGIN 10.168.192.in-addr.arpa.
$TTL 60
222     PTR     dns1.host.com.
223     PTR     dns2.host.com.
224     PTR     dns3.host.com.
```

#### 配置辅助DNS
修改主配置文件 /etc/named.conf，修改以下三个位置：
```json
[root@dns2 ~]# cat /etc/named.conf
options {
        listen-on port 53 { 192.168.10.223; };
        allow-query     { any; };
        masterfile-format text;
};
```

辅助dns检查主dns完全区域数据传送，解析列表如下:
```json
[root@dns2 slaves]#  dig -t AXFR host.com @192.168.10.222

; <<>> DiG 9.11.4-P2-RedHat-9.11.4-26.P2.el7_9.5 <<>> -t AXFR host.com @192.168.10.222
;; global options: +cmd
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070705 10800 900 604800 86400
host.com.               600     IN      NS      dns.host.com.
dns.host.com.           60      IN      A       192.168.10.222
dns1.host.com.          60      IN      A       192.168.10.222
dns2.host.com.          60      IN      A       192.168.10.223
test.host.com.          60      IN      A       192.168.10.55
host.com.               600     IN      SOA     dns.host.com. test.qq.com. 2021070705 10800 900 604800 86400
;; Query time: 0 msec
;; SERVER: 192.168.10.222#53(192.168.10.222)
;; WHEN: Thu Jul 15 16:13:37 CST 2021
;; XFR size: 7 records (messages 1, bytes 214)
```

配置辅助dns的zone引导配置，添加正解域和反解域：
```json
[root@dns2 slaves]# cat /etc/named.rfc1912.zones

zone "host.com" IN {
        type slave ;
        masters {192.168.10.222 ;} ;
        file "slaves/host.com.zone" ;
};
zone "10.168.192.in-addr.arpa" IN {
        type slave;
        masters {192.168.10.222 ;} ;
        file "slaves/10.168.192.in-addr.arpa.zone";
};

```
**注意：我们只有在/etc/named.rfc1912.zone中添加了需要同步的域名，辅助dns才会进行同步。不添加的域名dns是不会进行同步**。


重启辅DNS服务器：
```shell
# systemctl restart named
```

查看是否有区域数据库文件传输到辅助DNS slaves文件夹下:
```shell
[root@dns2 ~]# ls /var/named/slaves/
10.168.192.in-addr.arpa.zone  host.com.zone
```

在主DNS测试辅DNS是否可以解析。如果可以解析则说明主辅配同步完成：
```shell
[root@dns1 ~]# dig dns.host.com @192.168.10.223 +short
192.168.10.222
```

### 区域传输tune调优
参考:  [zone transfer tune](https://kb.isc.org/docs/aa-00726)
**潜在问题**：
![](attachments/Pasted%20image%2020231225161101.png)
即：zone更新的延迟生效、zone更新的同时影响对于client的dns请求等。

**master 服务器调优*：
![](attachments/Pasted%20image%2020231225161905.png)
![](attachments/Pasted%20image%2020231225162526.png)

**slave服务器调优**：
![](attachments/Pasted%20image%2020231225163005.png)


# bind配置文件
## 配置总体概述
`/var/named/` : 资源记录文档的存放位置
`/var/named/named.ca` ：13个根服务器存放文件
`/var/named/named.empty`：
`/var/named/named.localhost`：
`/var/named/named.loopback`：

`/etc/logrotate.d/named` ：日志切割配置文件

`/etc/named.conf`： 主配置文件
`/etc/named.rfc1912.zones`： 区域配置文件（即zone文件，用include指令包含在主配置文件）
`/etc/named.root.key`： 根区域的key文件以实现事务签名；
`/etc/rndc.key`： `rndc`加密密钥

`/etc/rndc.conf`： `rndc`（远程名称服务器控制器）配置文件

## 主配置文件`named.conf`
BIND 9 配置由**块、语句和注释**组成。
- 注释
- 语句
named.conf，每个语句都要使用分号结尾
- 块
此中的块，就是模块。一个模块中包含多个语句。


|named.conf配置文件所有的配置语句|功能说明|
|---|---|
|acl|定义一个主机匹配列表，用户访问控制权限|
|controls|定义rndc命令使用的控制通道，若省略，则只允许经过rndc.key认证的127.0.0.1的rndc控制|
|include|include filename; 引入外部局部配置文件|
|dnssec-policy|dnssec 安全策略，一般默认不启用dnssec功能|
|key|使用 TSIG 指定用于身份验证和授权的密钥信息，一般rndc的授权密钥信息|
|logging|日志记录|
|options|定义全局选项|
|statistics-channels|声明通信的通道，用于访问bind的统计信息数据|
|tls|指定 TLS 连接的配置信息|
|http|指定 HTTP 连接的配置信息|
|trust-anchors|定义 DNSSEC 密钥信息|
|zone|定义一个区域声明|
|view|定义域名空间的一个视图|
|server|可以出现在配置文件的顶级，可以在一个view中，定义对特定的服务器设置参数|
|primaries|定义一个命名的主服务器列表，一般包含在存根区或者辅区的primaries或者also-notify列表中。|
|parental-agents|定义要由主要和次要区域使用的委派代理列表|

重点说明下：acl、view、logging、options、zone等模块
### 范例
```json
# named.conf主配置文件参数详解 

//全局配置段options {
        listen-on port 53 { 192.168.31.113; }; #设置通信的网段，这里建议使用本机IP，并非127.0.0.1
        listen-on-v6 port 53 { ::1; }; #监听bind端口
        directory       "/var/named";  #指定区域文件存放路径
        dump-file       "/var/named/data/cache_dump.db"; #设置当执行rndc dumpdb命令后导出文件存放路径
        statistics-file "/var/named/data/named_stats.txt";
        memstatistics-file "/var/named/data/named_mem_stats.txt"; #服务器输出的内存使用统计文件名
        recursing-file  "/var/named/data/named.recursing";
        secroots-file   "/var/named/data/named.secroots";
        allow-query     { any; };  #允许查询来源(这里建议修改为IP地址，localhost代表只允许本机查询) any代表所有网段
        allow-transfer { none; } #允许查询的网段
        recursion yes; #是否开启递归查询
        dnssec-enable yes; #是否支持DNSEEC开关
        dnssec-validation yes; #是否开启dnsec确认开关
        bindkeys-file "/etc/named.root.key";
        managed-keys-directory "/var/named/dynamic";
        pid-file "/run/named/named.pid";
        session-keyfile "/run/named/session.key";
};  

//日志配置段
logging {
        channel default_debug {         
                file "data/named.run";
                severity dynamic;
        };
  #本段参数解释，将日志写入工作目录下的named.run文件。注意：如果服务器用-f参数启动，则named.run会被stderr所代替，severity 按照服务器当前Debug级别记录日志
  #bind日志可以写到很多地方，具体写入方式可以参考https://blog.csdn.net/zhu_tianwei/article/details/45103455
}; 

//区域配置段
zone "." IN {             #.代表根域
        type hint;       #代表根服务器，除hint还有master 代表主域名服务器，slave代表辅助域名服务器，forward 代表转发服务器
        file "named.ca";  #域信息源数据库信息文件名
}; 

include "/etc/named.rfc1912.zones";   #区域管理文件(包含资源记录、宏定义和注释)
                                      #通常定义在/var/named目录下且以.zone结尾
include "/etc/named.root.key";
```
### 注释
C 语言的注释风格，分别是`/* xxx */` 以及 `//`。 如下所示：
```bash
// : 表示注释一行
/* */ : 表示注释多行的内容
```
![](attachments/Pasted%20image%2020231221152917.png)
![](attachments/Pasted%20image%2020231221152953.png)

>说明：zone区域配置文件可以使用分号 `;` 字符注释，但是在主配置文件中`;`不是注释，而是配置文件的行末尾结束符。

## 主配置文件中的options 块
`options` 的参数设置会影响整个 BIND9 DNS环境的配置，具体各部分常用到的配置参数如下：
- `listen-on`： 用于配置监听的端口以及IPv4地址，默认的监听端口为：53；
- `listen-on-v6`：用于监听 IPv6 地址以及端口；
- `tls-port`：接收和发送 DNS-over-TLS 协议流量的 TCP 端口号。默认值为 853；
- `http-port`：通过 HTTP 接收和发送未加密的 DNS 流量的 TCP 端口号；
- `https-port`：接收和发送 DNS-over-HTTPS 协议流量的 TCP 端口号。默认值为 443；
- `directory`: 用于指定读取DNS数据文件的文件夹，默认的文件夹的路径为：`/var/cache/bind`；
- `dump-file`：用来设置域名缓存数据库文件的位置，可以自己定义。默认的存储文件为：`named_dump.db`；
- `statistics-file`：用来设置状态统计文件的位置，可以自己定义。；
- `memstatistics-file` ：用来设置服务器输出的内存使用统计信息。默认保存在 `/var/named/data` 目录下，文件名为 `named.memstats`；
- `allow-query`：用来设置允许DNS查询的客户端地址，默认值为`localhost`, 可以设置为某个网段、任意地址、具体的某台主机三种情况。例如，要修改为任意地址，就在括号内的加入 `any`，也可以引用之前创建的 `acl` 内的所有地址；
- `recursion`：用于设置递归查询，一般客户机和服务器之间属于递归查询，即当客户机向DNS服务器发出查询请求后，若DNS服务器本身不能解析，则会向另外的DNS服务器发出查询请求，得到结果后转交给客户机。此选项有`yes`和`no`两个值。这个选项用于设置 Failover 非常有用；
- `dnssec-enable`： 选项用来设置是否启用`DNSSEC`支持，`DNSSEC`可以用来验证`DNS数据`的有效性，该选项有`yes`和`no`两个值，默认值为`yes`。
- `dnssec-validation`：选项用来设置是否启用DNSSEC确认，默认值为`yes`，可以选择 `auto`。
- `bindkeys-file` ： 用来设置内置信任的密钥文件，其默认值为 `/etc/named/iscdlv.key`；
- `managed-keys-directory`： 选项用于指定目录中的文件存储位置，跟踪管理 `DNSSEC` 密钥
- `forwarders`：DNS转发器。用于设定该DNS解析服务器无法进行当前域名解析的情况下，进行转发解析的DNS地址，其中 `8.8.8.8` 和 `8.8.4.4` 是谷歌的免费DNS服务器的网络地址；`233.5.5.5` 和 `233.6.6.6` 是阿里云的免费DNS地址。当设置了 `forwarder` 的转发器之后，所有的非本域的和在缓存中无法查找到的域名查询都转发都设置的DNS转发器，由DNS转发器 完成转发操作。因此这台转发器的缓存中就记录了丰富的域名信息。因此如果遇到非本域的查询，转发器的缓存就可以做到查询，从而减少了向外部的查询流量。
- `rrset-order`：  
    在 BIND 9 提供的负载均衡策略建立在一个名称（域名 - Name）使用多个资源记录 ( Records ) 的情况下，其实现的轮询机制并不是传统的负载均衡服务器实现的轮询机制 - 即追踪和记录每一次应答的资源顺序；  
    BIND 9 实现了一个类似 List 的数据结构，将所有的资源记录填入到 一个顺序表中，这个填入的次序随机，或者根据设定的参数随机；  
    格式：[class _class_name_] [type _type_name_] [name “_domain_name_”] order _ordering_  
    如果参数没有被赋值，那么默认的赋值为： class: ANY type: ANY Name: *  
    参数：
    - `fixed` ： 根据 zone 文件定义资源记录的顺序按照顺序逐个进行解析；
    - `random`： 根据 zone 文件资源记录随机返回解析记录；
    - `cyclic`： 创建一个循环，循环输出资源记录；
    - `none`： 完全随机的资源返回形式；

## 主配置文件中的区域zone配置
### Zone 的引导配置
正向解析域名的引导文件，通过以下的格式进行定义：
```json
zone "<YOUR DNS Domain Name >" {
    <Configurations>
}
```
反向解析域名的引导文件，通过以下的格式进行定义：
```json
zone "<YOUR IP ADDRESS>-addr.arpa" {
    <Configurations>
}
```
#### 字段说明
`named.rfc1912.zones`主配置文件的区域配置
```c
zone "ZONE_NAME" IN {
    type {master|slave|hint|forward};
    file "ZONE_NAME.zone";
};
```

```json
zone <string> [ <class> ] {
	in-view <string>;
};
zone <string> [ <class> ] {
	type delegation-only;
};
zone <string> [ <class> ] {
	type forward;
	delegation-only <boolean>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
};
zone <string> [ <class> ] {
	type hint;
	check-names ( fail | warn | ignore );
	delegation-only <boolean>;
	file <quoted_string>;
};
zone <string> [ <class> ] {
	type mirror;
	allow-notify { <address_match_element>; ... };
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update-forwarding { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	check-names ( fail | warn | ignore );
	database <string>;
	file <quoted_string>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-in <integer>;
	max-transfer-time-out <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	request-expire <boolean>;
	request-ixfr <boolean>;
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	try-tcp-refresh <boolean>;
	use-alt-transfer-source <boolean>;
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type primary;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	auto-dnssec ( allow | maintain | off );
	check-dup-records ( fail | warn | ignore );
	check-integrity <boolean>;
	check-mx ( fail | warn | ignore );
	check-mx-cname ( fail | warn | ignore );
	check-names ( fail | warn | ignore );
	check-sibling <boolean>;
	check-spf ( warn | ignore );
	check-srv-cname ( fail | warn | ignore );
	check-wildcard <boolean>;
	database <string>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	dlz <string>;
	dnskey-sig-validity <integer>;
	dnssec-dnskey-kskonly <boolean>;
	dnssec-loadkeys-interval <integer>;
	dnssec-policy <string>;
	dnssec-secure-to-insecure <boolean>;
	dnssec-update-mode ( maintain | no-resign );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	inline-signing <boolean>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	key-directory <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-out <integer>;
	max-zone-ttl ( unlimited | <duration> );
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-to-soa <boolean>;
	nsec3-test-zone <boolean>; // test only
	parental-agents [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	parental-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	parental-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	serial-update-method ( date | increment | unixtime );
	sig-signing-nodes <integer>;
	sig-signing-signatures <integer>;
	sig-signing-type <integer>;
	sig-validity-interval <integer> [ <integer> ];
	update-check-ksk <boolean>;
	update-policy ( local | { ( deny | grant ) <string> ( 6to4-self | external | krb5-self | krb5-selfsub | krb5-subdomain | krb5-subdomain-self-rhs | ms-self | ms-selfsub | ms-subdomain | ms-subdomain-self-rhs | name | self | selfsub | selfwild | subdomain | tcp-self | wildcard | zonesub ) [ <string> ] <rrtypelist>; ... } );
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type redirect;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	dlz <string>;
	file <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-records <integer>;
	max-zone-ttl ( unlimited | <duration> );
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type secondary;
	allow-notify { <address_match_element>; ... };
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	allow-transfer [ port <integer> ] [ transport <string> ] { <address_match_element>; ... };
	allow-update-forwarding { <address_match_element>; ... };
	also-notify [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	alt-transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	alt-transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	auto-dnssec ( allow | maintain | off );
	check-names ( fail | warn | ignore );
	database <string>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	dlz <string>;
	dnskey-sig-validity <integer>;
	dnssec-dnskey-kskonly <boolean>;
	dnssec-loadkeys-interval <integer>;
	dnssec-policy <string>;
	dnssec-update-mode ( maintain | no-resign );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	inline-signing <boolean>;
	ixfr-from-differences <boolean>;
	journal <quoted_string>;
	key-directory <quoted_string>;
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-ixfr-ratio ( unlimited | <percentage> );
	max-journal-size ( default | unlimited | <sizeval> );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-idle-out <integer>;
	max-transfer-time-in <integer>;
	max-transfer-time-out <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	notify ( explicit | master-only | primary-only | <boolean> );
	notify-delay <integer>;
	notify-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	notify-to-soa <boolean>;
	nsec3-test-zone <boolean>; // test only
	parental-agents [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	parental-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	parental-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	request-expire <boolean>;
	request-ixfr <boolean>;
	sig-signing-nodes <integer>;
	sig-signing-signatures <integer>;
	sig-signing-type <integer>;
	sig-validity-interval <integer> [ <integer> ];
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	try-tcp-refresh <boolean>;
	update-check-ksk <boolean>;
	use-alt-transfer-source <boolean>;
	zero-no-soa-ttl <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type static-stub;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	max-records <integer>;
	server-addresses { ( <ipv4_address> | <ipv6_address> ); ... };
	server-names { <string>; ... };
	zone-statistics ( full | terse | none | <boolean> );
};
zone <string> [ <class> ] {
	type stub;
	allow-query { <address_match_element>; ... };
	allow-query-on { <address_match_element>; ... };
	check-names ( fail | warn | ignore );
	database <string>;
	delegation-only <boolean>;
	dialup ( notify | notify-passive | passive | refresh | <boolean> );
	file <quoted_string>;
	forward ( first | only );
	forwarders [ port <integer> ] [ dscp <integer> ] { ( <ipv4_address> | <ipv6_address> ) [ port <integer> ] [ dscp <integer> ]; ... };
	masterfile-format ( raw | text );
	masterfile-style ( full | relative );
	max-records <integer>;
	max-refresh-time <integer>;
	max-retry-time <integer>;
	max-transfer-idle-in <integer>;
	max-transfer-time-in <integer>;
	min-refresh-time <integer>;
	min-retry-time <integer>;
	multi-master <boolean>;
	primaries [ port <integer> ] [ dscp <integer> ] { ( <remote-servers> | <ipv4_address> [ port <integer> ] | <ipv6_address> [ port <integer> ] ) [ key <string> ] [ tls <string> ]; ... };
	transfer-source ( <ipv4_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	transfer-source-v6 ( <ipv6_address> | * ) [ port ( <integer> | * ) ] [ dscp <integer> ];
	use-alt-transfer-source <boolean>;
	zone-statistics ( full | terse | none | <boolean> );
};

```

- class: 上面的 class 一般默认指**IN**类型。
-  `zone "ZONE_NAME“ `：定义解析库名字。通常和解析库文件前缀对应起来。
-  `type `：
	-  `master `：指的是主dns解析
	-  `slave `：指的是从dns解析
		> 注：slave服务器不需要配置具体的`zone`文件以及`zone`引导配置。因为slave启动成功后，会自动同步master里面的RR记录，并且表明同步来的zone记录的type是slave类型，以及 master的`ip`地址。
	-  `hint `：指的是根域名解析（根提示域）
	-  `forward `：指的是转发（取值为`first` or `only`），需要配置`forwarders`字段。
	> 注：`forward `字段只有当forwarders 列表中有内容的时候才有意义。当值是 first，默认情况下，使服务器先查询设置的forwarders，如果它没有得到回答，服务器就会查询全局options转发器寻找答案。如果设定的是 only，服务器就只会把请求转发到指定的服务器上去。

-  `file ` ：指定存放dns记录的数据文件名称（位置默认在/var/named下面）
file的前缀通常和zone的名字通常对应起来，然后加一个.zone的后缀。
-  `allow-update `：是否允许客户主机或服务器自行更新dns记录。
- `allow-transfer`: 用来给出 Failover 或者是 递归查询DNS服务器的IP地址，如果之前在 `options` 里配置的`allow-transfer` 如果设置成了参数 `yes`， 那么需要在这里指出递归查询服务器的IP地址；
- `in-view`: 
> BIND9.10+ only. "in-view", was added that lets multiple views refer to the same in-memory instance of a zone. Allows a zone clause within one view to be used by another view. This allows both views to use the same zone without the overhead of loading it more than once.
> The **view-name** must refer to a valid view which contains a zone of the same name and the view containing the zone must have been previously defined (only backward references to views are allowed, not forward references).

```json
格式：
in-view "view-name";

范例：
in-view "internal";
```

#### 范例

范例如下所示：
```json
 zone example.com {
  // shed load of statements
  // type may be master, slave etc.
  ...
 };
};

view "gondor" {
 ...
 zone example.com {
  in-view "mordor";
  // valid back-reference to previous defined view
  // a forward reference to "khand" would fail
  // forward and forwarders statements are allowed - nothing else
 };
};

view "khand" {
 ...
 zone example.com {
  in-view "mordor";
 };
};
```


>注：自定义的配置解析库文件（Zone files）,一般是在/var/named下写，文件名格式一般写为ZONE_NAME.zone。


### 区域文件(Zone文件)
zone文件：保存 RR (Record Resource) 信息的文件。
zone文件包括正向Zone文件和反向Zone文件。
DNS正解: 域名解析->返回IP；
DNS反解: IP解析->域名；
> 注：在 zone file 中，注释的符号是`;`。

#### 区域文件的写法
```json
zone "abc.com" IN {
    type primary;
    file "/usr/local/bind/etc/named.abc.com";  // RR记录文件名称可以自定义 最好取有意义名称
};
```
**FQDN的写法：**
```json
$TTL 600
@                    IN  SOA     primary.abc.com. admin.abc.com. ( 2022120802 3H 15M 1W 1D )
@                    IN  NS      primary.abc.com.
primary.abc.com.     IN  A       192.168.10.200
@                    IN  MX  10  www.abc.com.
www.abc.com.         IN  A       192.168.10.121
bbs.abc.com.         IN  CNAME   www.abc.com.
ftp.abc.com.         IN  CNAME   www.abc.com.
linux.abc.com.       IN  CNAME   www.abc.com.
secondary.abc.com.   IN  A       192.168.10.201
122.abc.com.         IN  A       192.168.10.122


# 其中2022120802 3H 15M 1W 1D,分别是serial,refresh,retry,expire,Minimum

```

**简写：**
```json
$TTL 600
@               IN  SOA     primary.abc.com. admin.abc.com.( 2022120802 3H 15M 1W 1D )
@               IN  NS      primary
primary         IN  A       192.168.10.200
@               IN  MX  10  www
www             IN  A       192.168.10.121
bbs             IN  CNAME   www
ftp             IN  CNAME   www
linux           IN  CNAME   www
secondary       IN  A       192.168.10.201
122             IN  A       192.168.10.122
```

简写不太容易看明白，而FQDN的写法，又太啰嗦，而且要注意.（点号），所以我个人偏好喜欢这样的写法。
```json
$TTL 600
@               IN  SOA     primary.abc.com. admin.abc.com.( 2022120802 3H 15M 1W 1D )
@               IN  NS      primary.abc.com.
master          IN  A       192.168.10.200
@               IN  MX  10  www.abc.com.
www             IN  A       192.168.10.121
bbs             IN  CNAME   www.abc.com.
ftp             IN  CNAME   www.abc.com.
linux           IN  CNAME   www.abc.com.
secondary       IN  A       192.168.10.201
122             IN  A       192.168.10.122
```

#### 正向区域文件
```json
[root@linuxmaster named]# vi /var/named/aliyun.com.zone
$TTL 300;
@   IN SOA  linuxmaster.aliyun.com. admin.aliyun.com. (
                    2017051720      ;serial
                    1H          ;refresh
                    5M          ;retry
                    7D          ;expiry
                    3D )        ;minimum
    IN  NS  linuxmaster
    IN  MX 20 MX
linuxmaster IN  A   172.24.8.10
www         IN  A   172.24.8.30
mirrors     IN  A   172.24.8.30
ftp         IN  CNAME   www
```

在 zone file 中：
注释的符号是：`;`
`$ORIGIN [REGION NAME]` 设定了当前的解析域名区域；
`@`：代表当前的区域；
> 一般在`$ORIGIN [REGION NAME]` 设定了当前的区域；否则，就默认为在主配置文件`named.conf`中当前文件对应的 `ZONE_NAME`。

比如：
```josn
;
; BIND data file for local loopback interface
;
; Import ZSK / KSK
;
;
$ORIGIN domain.com.
; 我们已经定义了一个区域，那么在定义 SOA 的时候可以进行两种定义方式
@ IN SOA ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
; 或者我们不需要 at-sign - @ 符号，直接引用ORIGIN的名字
；在这里这两条配置代表的含义是一样的
domain.com. ns.domain.com. admin.domain.com. (
                  3     ; Serial
             604800     ; Refresh
              86400     ; Retry
            2419200     ; Expire
             604800 )   ; Negative Cache TTL
```

##### TTL
- TTL：定义区域中数据文件里面的各项记录的默认TTL值，单位为秒；
RR都会被保存在DNS的解析服务器的cache上，有一个失效的时间，TTL就是控制这个失效时间的一个参数；
这个参数可以单独进行设定，也可以在 SOA 设定中进行配置：

- 单独设定： `$TTL [TIME]`
- 在 SOA 中进行设定： `SOA - Negative Cache TTL`

##### SOA记录
- SOA：SOA记录，@代表相应的域名，每个区域数据文件只能有一个SOA，其中参数如下：
	- serial：表示配置文件的修改版本，格式为年月日加上修改的次数；
	- refresh：设定辅助dns和主dns进行同步的间隔时间；
	- retry：如果辅助dns进行更新失败后，间隔多久进行重试；
	- expiry：设定辅助dns与主dns同步失败后，多长时间后清除对应的记录；
	- minimum：默认最小的TTL值，如果在前面没有设置TTL，则以此值为准。

- **注意**：
>在所有的配置中，`ns.domain.com != ns.domain.com.` ，必须注意在 zone file 中的配置文件的最后 `.` 必须不能省略；
>如果不写最后一个的 `.` 那么该域名就是一个 **相对名** ，结果就是在解析的过程中，这条资源就被当成 `ns.domain.com.domain.com`

##### NS记录
Name Server Records 定义了在当前 区域的 DNS服务器 ；
每个NS必须有对应的 IP地址，即在每一个 zone file 中必须指定 主/从 域名解析器的IP地址（`A` 记录/`AAAA`记录）。

```json
; 记录 NS 记录
@               IN          NS          ns.domain.com.
; 记录 NS 记录对应的 IP 地址信息
ns.domain.com.          IN          A           192.168.1.1
```
##### A记录
Address Records 记录了 域名 与 IP 地址的对应关系；
```json
ns.domain.com. IN A 192.168.1.1
```

##### Cname记录
CName 将 单个昵称或者别名映射到一个可能存在在区域外的真实的区域，在一个域名下存在多个子域名。
如果需要更改映射之前的子域名，那么只需要更改映射的域名地址就可以了；
```json
; 
$TTL 2d
$ORIGIN domain.com.
...
server1     IN      A       192.168.1.1
www             IN      CNAME       Server1
```


#### 反向区域文件
区域名称：是网络地址的反向.in-addr.arpa.
一个IP只能对应唯一的FQDN反解PTR记录，且应该与正解域对应。
> 注意：需要注意一点的是反解析对应的配置文件应该不带有点分十进制ip地址的最后一份。例如解析的IP为`192.168.31.113` 配置文件应该写为`31.168.192.in-addr.arpa` 其中`113`这里不写，是在后面添加解析时使用。

```c
比如：
192.168.111. –> 111.168.192.in-addr.arpa.

配置方法：
在/etc/named.rfc1512.zones文件下插入下面内容：
zone "Reverse_Net_Addr.in-addr.arpa" IN {
    type {master|slave|forward};
    file "Net_Addr.zone"
}
```
另外，在zone文件中，反解域也是要有SOA记录的，在反解域中ns记录就不用在写A记录了，因为反解区域文件只可以有PTR，不可以有A记录。

#### slave和master中的zone配置差异
**master(primary)中的zone配置**：
```json
# zone引导配置
zone "abc.com" IN {
    type primary;
    file "/usr/local/bind/etc/named.abc.com";  // RR记录文件名称可以自定义 最好取有意义名称
};


# zone配置文件
# cat /usr/local/bind/etc/named.abc.com
$TTL 600
@               IN  SOA     primary.abc.com. admin.abc.com.( 2022120802 3H 15M 1W 1D )
@               IN  NS      primary.abc.com.
master          IN  A       192.168.10.200
@               IN  MX  10  www.abc.com.
www             IN  A       192.168.10.121
bbs             IN  CNAME   www.abc.com.
ftp             IN  CNAME   www.abc.com.
linux           IN  CNAME   www.abc.com.
secondary       IN  A       192.168.10.201
122             IN  A       192.168.10.122
```

slave(secondary)不需要配置对应的zone配置，因为secondary启动成功后，会自动同步primary里面的RR记录。在named.conf中的配置也不一样，如下所示：
```json
zone "abc.com" IN {
    type secondary;
    file "/usr/local/bind/etc/named.abc.com";
    primarys {192.168.10.200;};
};
```

### 正向区域文件的配置范例
这里给出一个自定义的总的区域定义文件，新加一个区域文件的定义：
```c
zone "zhangqifei.top" IN {
    type master;
    file "zhangqifei.top.zone";
};
```

/var/named/ZONE_NAME.zone区域配置文件：这里给出我一个自定义的区域文件：zhangqifei.top.zone
```json
$TTL 1D
@    IN  SOA  ns1.zhangqifei.top. me.zhangqifei.top ( 0 1H 10M 1D 3H)
     IN  NS   ns1.zhangqifei.top
     IN  NS   ns2
         MX 10 mail1
         MX 20 mail2
ns1.zhangqifei.top  IN  A  192.168.111.254
ns2  IN  A  192.168.111.253  
db1   A    192.168.111.100
db2   A    192.168.111.111
web1  A    192.168.111.200
web2  A    192.168.111.222
mail1 A    192.168.111.10
mail2 A    192.168.111.20
www   CNAME   web1
```
说明：
```c
@指的就是本域 zhangqifei.top
IN ns2.zhangqifei.top可以省略写成IN ns2
IN都可以省略不写,比如直接写MX mail


设置.zone文件权限（参照/var/named/name.xxxx的权限来设置，这里xxx为任意字符）
chmod 640 zhangqifei.top.zone
chown :named zhangqifei.top.zone

配置完毕后，启动或重启(已启动的话）服务。
service named start|restart
```

修改文件属性：
```c
将新创建的zone文件修改为和/var/named下的其他的zone文件相同的属性：

chown named:named /var/named/zhangqifei.top.zone
chmod 640 /var/named/zhangqifei.top.zone
```
检查配置文件：
```c
named-checkzone  zhangqifei.top  /var/named/zhangqifei.top.zone
```

测试：
```c
dig ns2.zhangqifei.top
```
### 反向区域文件的配置范例

**范例一**：
编辑`/etc/named.rfc1912.zones`，添加一个域名
```c

zone "111.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.111.zone";
};
配置/var/named/ZONE_NAME.zone
不需要MX、A、AAAA，要有SOA、NS记录，PTR记录。
```
然后配置192.168.111.zone：
```c
$TTL 1D
@   IN  SOA ns1.magedu.com. me.zhangqifei.top (
                    20170001
                    1H
                    5M
                    7D
                    1D)
    IN NS ns1.zhangqifei.top.
    IN NS ns2.zhangqifei.top.
254 IN PTR ns1.zhangqifei.top.
253 IN PTR ns2.zhangqifei.top.
100 IN PTR db1.magedu.com.
111 IN PTR db2.magedu.com.
200 IN PTR web1.magedu.com.
222 IN PTR web2.magedu.com.
10  IN PTR mail1.magedu.com.
20  IN PTR mail2.magedu.com.
```

检查区域配置文件是否正常：
```c
named-checkzone 111.168.192.in-addr.arpa /var/named/192.168.111.zone
```


**范例二**：
编辑`/etc/named.rfc1912.zones`，添加一个域名：
```json
zone "31.168.192.in-addr.arpa" IN {
        type master;
        file "31.168.192.in-addr.arpa.zone";
        allow-update { 192.168.31.113; };};

#检查配置文件
[root@dns01-113 ~]# named-checkconf
```

在`/var/named/`下创建反解域区域数据库文件：
```json
[root@dns01-113 ~]# vim /var/named/31.168.192.in-addr.arpa.zone     #名称需要根据上面定义的来创建
$TTL  60
@         IN  SOA    dns.host.com.    604419314.qq.com. (
                     20200818
                     10800
                     900
                     64800
                     86400
                     )
              NS     dns.host.com.

$ORIGIN  31.168.192.in-addr.arpa.
$TTL 60 
113          PTR     dns01-113.host.com.
114          PTR     dns02-114.host.com. 
#####################
提示: 如果我们$ORIGIN 31.168.192.in-addr.arpa.不写全，也是可以的。例如 
$ORIGIN in-addr.arpa.
$TTL 60
113.31.168.192   PTR   dns01-113.host.com.
#上面的写法也是可以的
```

说明: 反解域也是要有SOA记录的，在反解域中ns记录就不用在写A记录了，因为反解区域文件只可以有PTR，不可以有A记录

检查区域配置文件是否正常：
```json
#31.168.192.in-addr.arpa为我们反向解析的域名 
[root@dns01-113 ~]# named-checkzone 31.168.192.in-addr.arpa. /var/named/31.168.192.in-addr.arpa.zone
zone 31.168.192.in-addr.arpa/IN: loaded serial 20200818
OK
```

重启`named`：
```json
systemctl restart named
```

测试：
```bash
#第一种方法
# dig -t PTR 113.31.168.192.in-addr.arpa. @192.168.31.113  +short
dns01-113.host.com. 

#第二种方法
# dig -x 192.168.31.114 @192.168.31.113 +short
dns02-114.host.com. 

#dig 后面ip为需要解析的ip，@后面ip为dns服务器ip
```

## 主配置文件中的acl 和 view块
一般来说，ACL模块用来控制哪些主机可以访问域名解析服务器，使用`ACL`访问控制列表可以使配置简单而清晰，一次定义之后可以在多处使用，不会使配置文件因为大量的 IP 地址而变得混乱。
采用这个配置可以有效防范DOS以及Spoofing攻击。

ACL匹配客户端是否能够接入到域名服务器基于三个基本的特征:
- 客户端的IPv4或者IPv6地址
- 用于签署请求的 TSIG 和 SIG(0) 密钥
- 在DNS客户端子网选项中编码的前缀地址

### ACL格式
```json
acl ACL_NAME {
    <address_match_element>;
    ...
};
```

范例如下所示：
```json
# 范例一：
acl innet {
    10.189.9.0/24;
    127.0.0.1/8;
    192.168.0.0/24;
};

options {
    directory "/var/named";
    allow-recursion { innet; };
};

# 范例二：
acl "someips" {                               //定义一个名为someips的ACL  
  10.0.0.1; 192.168.23.1; 192.168.23.15;      //包含3个单个IP  
};
 
acl "complex" {             //定义一个名为complex的ACL  
  "someips";                //可以嵌套包含其他ACL  
  10.0.15.0/24;             //包含10.0.15.0子网中的所有IP  
  !10.0.16.1/24;            //非10.0.16.1子网的IP  
  {10.0.17.1;10.0.18.2;};   //包含了一个IP组  
  localhost；               //本地网络接口IP（含实际接口IP和127.0.0.1）  
};
```

**named的内置ACL**：
- `none` : 没有一个主机
- `any` : 任意
- `local/localhost` : 本机
- `localnet` : 本机IP所属的网络

### 使用ACL
定义了 ACL 之后，可以在任何可以使用ACL匹配的子句中使用。
下面简单介绍常用的子句：

|子句|模块|说明|
|---|---|---|
|allow-query|options,zone|指定哪些主机或网络可以查询本服务器的区域，默认的是允许所有主机进行查询。|
|allow-transfer|options,zone|指定哪些主机允许和本地服务器进行域传输，默认值是允许和所有主机进行域传输。|
|allow-recursion|options|指定哪些主机可以进行递归查询。如果没有设定，缺省是允许所有主机进行递归查询的。注意禁止一台主机的递归查询，并不能阻止这台主机查询已经存在于服务器缓存中的数据。|
|allow-update|zone|指定哪些主机允许为主域名服务器提交动态 DNS 更新。默认为拒绝任何主机进行更新。|
|blackhole|options|指定不接收来自哪些主机的查询请求和地址解析。默认值是 none 。|

注意：上面列出的一些配置子句既可以出现在全局配置 **options** 语句里，又可以出现在 **zone** 声明语句里，当在两处同时出现时，**zone** 声明语句中的配置将会覆盖全局配置 **options** 语句中的配置。

### View 视图格式
​`view`语句定义了视图功能。视图是`BIND9`提供的强大的新功能，允许`DNS`服务器根据客户端的不同，有区别地回答`DNS`查询，每个视图定义了一个被特定客户端子集见到的`DNS`名称空间。

**视图语句的顺序是很重要的**。一位用户的请求将会在它所匹配的第一个视图中被解答。
在视图语句中定义的域只对匹配视图的用户是可用的。通过在多个视图中用相同名称定义一个域，不同域数据可以传给不同的用户。

视图的定义格式，如下所示：
```json
view view_name [ class ] {
    match-clients { address_match_list } ;
    match-destinations { address_match_list } ;
    match-recursive-only <boolean> ;
   [ view_option ; ... ]
   [ zone-statistics yes_or_no ; ]
   [ zone_statement ; ... ]
} ;
```

**字段解析**：
- `match_clients`和`matach-destinations`：根据用户的源地址("address_match_list")匹配视图定义的"match_clients"和用户的目的地址("address_match_list")匹配视图定义的"matach-destinations"。如果没有被指定，match-clients和match-destinations默认匹配所有地址。
- `match-recursive-only`：一个视图也可以做为match-recursive-only来指定，意思是来自匹配用户的递归请求将会匹配该视图。
- `class`：视图精确到类。如果没有给定任何类，就假设为IN类。如果在配置文件中没有view语句，在IN类中就会自动产生一个默认视图匹配于任何用户，任何指定在配置文件的zone配置被看作是此默认视图的一部分。

### ACL和View实现智能DNS
ACL及视图的配合使用，可实现智能DNS的实现：

**主配置文件中的内容**：
```json
# cat /etc/named.conf
    acl telecom {
        10.189.9.0/24;    
    }; 
                #此ACL定义了电信IP的列表
    acl unicom {
        10.189.8.0/24;
    };
                #此ACL定义了联通IP的列表
    options {
        directory "/var/named";
        allow-recursion {   localnet; };   #定义了允许递归请求的主机为所在网络的所有主机
        allow-query { any; };
    };

    view  telecom {
        match-clients { telecom; };
        zone "mageedua.com" IN {
            type master;
            file "telecom.mageedua.com.zone";
        };
    };

    view unicom {
        match-clients { unicom; };
        zone "mageedua.com" IN {
            type master;
            file "unicom.mageedua.com.zone";
        };
    };
```
在`named.conf`配置文件中，配置`view`功能，并在视图 `view` 区域定义`match-clients `参数，让`match-clients`引用`acl`列表，`acl`可以为电信IP列表，或者联通IP列表。
最后在`view`视图中定义需要智能DNS的区域。一般情况，可以将区域划分三类： 
1、内网视图  
2、电信视图 
3、联通视图

**不同视图的zone文件配置**：
电信视图的zone文件：
```json
# cat telecom.mageedua.com.zone
    $TTL 86400
    @   IN  SOA ns1.mageedua.com.   admin.mageedua.com. (
            2015061101
            1D
            1H
            7D
            1D )
        IN  NS  ns1
        IN  MX  10    mail
    ns1 IN  A   10.189.9.110
    mail    IN  A   10.189.9.111
    www IN  A   10.189.9.112   #电信www服务器为112的地址
```

联通视图的zone文件：
```json
# cat unicom.mageedua.com.zone
    $TTL 86400
    @   IN  SOA ns1.mageedua.com.   admin.mageedua.com. (
            2015061101
            1D
            1H
            7D
            1D )
        IN  NS  ns1
        IN  MX  10    mail
    ns1 IN  A   10.189.9.110
    mail    IN  A   10.189.9.111
    www IN  A   10.189.9.113  #联通www服务器为113的地址
```

测试（ #电信用户查询结果，如下所示）：
```
dig -t A www.mageedua.com @10.189.9.110  

    ; <<>> DiG 9.9.4-RedHat-9.9.4-14.el7_0.1 <<>> -t A www.mageedua.com @10.189.9.110
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 58349
    ;; flags: qr aa rd; QUERY: 1, ANSWER: 1, AUTHORITY: 1, ADDITIONAL: 2
    ;; WARNING: recursion requested but not available

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 4096
    ;; QUESTION SECTION:
    ;www.mageedua.com.      IN  A

    ;; ANSWER SECTION:
    www.mageedua.com.   86400   IN  A   10.189.9.112

    ;; AUTHORITY SECTION:
    mageedua.com.       86400   IN  NS  ns1.mageedua.com.

    ;; ADDITIONAL SECTION:
    ns1.mageedua.com.   86400   IN  A   10.189.9.110

    ;; Query time: 1 msec
    ;; SERVER: 10.189.9.110#53(10.189.9.110)
    ;; WHEN: 四 6月 11 10:26:21 CST 2015
    ;; MSG SIZE  rcvd: 95
```

## 主配置文件中的logging 块
### 背景
在默认情况下，bind 9把日志消息写到`/var/log/messages`文件中，而这些日志消息是非常少的，主要就是启动，关闭的日志记录和一些严重错误的消息，所以要详细记录服务器的运行状况，需要自己配置服务器的日志行为，也就是要在配置文件`named.conf`中使用`logging`语句来定制自己所需要的日志记录。

### 介绍
`logging` 部分的配置为DNS解析服务器提供了日志记录的功能，DNS服务器上的所有日志记录被存储到了指定的文件中。其通用的配置文件为:
```json
logging {
      category <string> { <channel_name_string>; ... };
      channel <string> {
              buffered <boolean>;
              file <quoted_string> [ versions ( unlimited | <integer> ) ]
                  [ size <size> ] [ suffix ( increment | timestamp ) ];
              null;
              print-category <boolean>;
              print-severity <boolean>;
              print-time ( iso8601 | iso8601-utc | local | <boolean> );
              severity <log_severity>;
              stderr;
              syslog [ <syslog_facility> ];
      };
};
```
从上边的通用配置格式可以看出来，`logging` 模块分为两个部分，`category` 和 `channel`。
- `channel`
`channel`的作用是指定输出的文件、日志格式和事件的严重性。
每一个`channel`可以指定一个 `category` 来指定记录的事件类型。

- `category`
 `category` 用来区分不同的类别或者场景，比如：客户端请求（`client request`）、配置文件解析处理（`Configuration file parsing and processing`）。

### 原理
首先需要创建一个 `channel` 来规定输出日志流的格式还以及日志文件名、文件版本。
每一个 `channel` 可以被多个 `category` 调用使用，每一个 `category` 相当于一个 `BIND9` 内嵌的服务模块，服务模块去调用日志配置模最后输出格式化日志。

### `channel` 的配置
**`channel`配置规则**：
`channel` 的配置规则：
- 所有的日志输出都需要 channel 来指定输出格式，BIND9 对于创建 channel 的数量没有限制。
- 每一个 channel 都需要为该通道的日志信息指定一个 `destination clause` - 目的句柄；
>目的句柄在 channel 阶段被配置，这个目的句柄用来区分： 
>1）输出到具体的文件的名字 - `file`；
>2）输出到具体的系统日志工具中（syslog/syslogd）- `syslog`；
>3）输出到终端显示- 标准错误流(standard error stream)；
>4）或者该错误消息直接被丢弃 - `null`。
- channel 的配置可以规定每一个错误日志消息的响应级别，默认的响应级别是`info`。
- channel 还可以控制输出错误日志消息的格式，可以包含：响应时间戳、category名字、严重等级等。

#### `channel` 的配置参数
- `buffered`: 用来规定是否刷新错误日志的文件，其参数值为`<boolean>`，在 BIND9 中 `<boolean>` 值的参数值为 `yes` / `no`，如果设置成为 `yes` 那么日志消息流(一般每一个错误日志消息都是一个 Log Entry)就不会刷新，而是被保存在缓冲区中了，不会刷新到文件中。
    
- `file`： 类似于Linux的通道概念，file 将日志输出流通过通道直接输出给文件，从上边的通用配置可以看出来可以为 file 指定文本文件的大小 - `size` ；指定 log 文件的版本号 - `version`；指定用于命名备份版本的格式 - `suffix`；
> - `size` ：用来限制log文件的大小，如果log文件的大小设置超过了设定的阈值，那么系统会自动停止输出内容到文件中；   
> - `versions`： 用于指定新创建的 log文件数存储到本地的上限值，默认的参数值为`unlimited`，当指定的文件的大小超过设定的`size`值得时候，如果没有指定 `versions`，那么系统就不会继续写进log；如果制定了`versions`，那么就会继续写入；比如指定5个版本（version 5），当access.log达到size的大小，则会保存为query.log.0，以此类推一直保存到access.log.4。


> - `suffix` ：设定用来命名`log`文件的方式；

- `syslog`：将通道定向到系统的日志文件流中； 常用的支持日志文件服务为：`dameon`、`syslog`、`local6`、`local7`；
- `severity`：用来指定记录消息的级别，相当于 `syslog` - `priorities`。
比如说定义了日志的严重级别为 `Debug`，那么会输出日志事件 `Debug` 以上的错误到文件中。按照严重性递减的顺序，如下所示：
```c
1. critical
2. error
3. warning
4. notice
5. info
6. debug 
7. dynamic
```

- `stderr`：将通道指向服务器的标准错误流。这是为了在服务器作为前台进程运行时使用；
- `print-time`： `yes` / `no` / `local` / `iso8601` / `iso8061-utc` 可以设定不同的输出到日志文件的时间格式；
- `print-category`：打印日志消息配置`category` 的信息到你设定的日志文件中；
- `print-severity`： 打印日志的严重等级


### `category` 的配置
`category`词组配置规则：`category <config_string> { <channel_name_string>; ... };`

|类别|说明|
|---|---|
|default|匹配所有未明确指定通道的类别，但是不匹配不属于任何类别的消息。这些不属于任何类别的消息属于下面列出的这些类别。|
|general|包括所有未明确分类的bind消息|
|client|处理客户端请求|
|database|同bind内部数据库相关的消息，用来存储区数据和缓存记录|
|dnssec|处理DNSSEC签名的响应|
|lame-servers|发现错误授权|
|network|网络操作|
|notify|异步区域变动通知|
|queries|查询日志|
|resolver|名字解析，包括对来自解析器的递归查询的处理|
|security|认可/非认可的请求|
|update|动态更新事件|
|xfer-in|从远程名称服务器到本地名称服务器的区传送|
|xfer-out|从本地名称服务器到远程名称服务器的区传送|


### 范例
```json
logging{
    channel default_logfile {
        file "/var/log/bind/example_1.log" versions 3 size 1m;
        print-category yes;
        print-severity yes;
        print-time yes;
        severity info;
    };

    channel default_querylogfile{
        file "/var/log/bind/query.log" versions 3 size 1m;
        print-category yes;
        print-severity yes;
        print-time yes;
        severity info;
    };

    channel default_databaselogfile{
        file "/var/cache/bind/data/database.log" versions 3 size 3m;
        print-time yes;
        print-severity yes;
        severity debug;
    };

    category default {
        default_logfile;
    };
    category queries {
        default_querylogfile;
    };
    category database {
        default_databaselogfile;
    };
    category unmatched { null; };
};
```

## 主配置文件中的 control 块
controls语句声明了系统管理员用于管理名称服务器远程操作的控制通道。
`rndc`使用这些控制通道向名称服务器发送命令，并从名称服务器检索非dns结果。

## 安全性配置
### 背景
DNS 的设计起始于上世纪 80 年代，当时互联网规模小得多，安全性并非首要设计考虑因素。当DNS的递归解析器(DNS Resolver )向权威域名服务器发送查询请求的时候，解析器无法验证响应的真实性。解析器只是检查响应的IP地址与解析器发送初始查询的IP地址是否相同。
但是， 这种检查机制存在一个非常严重的问题：权威域名服务器的请求包的源地址非常容易被伪造和仿冒，攻击者很容易伪装成解析器最初查询的权威域名服务器，因此可以很轻松将用户重定向到其他的恶意网站。因为需要考虑到递归解析器可能被入侵的场景。

域名解析器有一种缓存的机制用来加速解析流程，其原理是从权威域名服务器和根解析器时候收到的上次DNS数据存储到本地来加速下一次应答，避免查询一个或者多个权威服务器引发的服务延迟。然而，依靠缓存存在一个缺点，如果攻击者发送的假冒的DNS权威域名服务器响应被接受，那么其缓存就会被一直缓存在递归解析服务器中，因此之后所有的查询DNS Server 的主机都会接收到欺诈性DNS数据。

### 方案
####  `allow-query`
首先， BIND9 的 `options` 提供的 `allow-query` 参数可以对访问DNS解析服务器的客户端地址和网段进行限制，从而达到对陌生客户端和恶意IP地址的封禁。

#### `DNSSEC`
BIND 提供了一种名叫 `DNSSEC` 的安全扩展服务以及配套的密钥生成工具供网络人员使用。

DNSSEC 采用基于公钥加密的数字签名技术来加强 DNS的验证强度。 DNSSEC 并非对DNS查询和响应的过程进行传输加密，而是数据所有者对 DNS 数据进行签名加密。

DNSSEC 在 DNS协议中新增加的两个安全的功能：
- 数据来源验证 ：验证收到的数据是否确实来自其认定的数据传送区域。
- 数据完整性保护：验证自从区域所有者使用私钥第一次进行数字签名，数据在传输过程中并未遭到修改。

每个zone都会发布一个公钥，递归解析器可以检索公钥以验证区域中的数据，区域的公钥也必须经过签名。
>每一个 DNS Zone 在配置的时候 BIND 服务会为其创建一个公钥和私钥的配对的秘钥对。 DNS zone 的所有者使用该区域的私钥对区域内的DNS数据进行加密，并为这些数据生成签名。 DNS 解析器生成的 公钥 对该区域内所有的 DNS数据验证真实性。如果有效，证明DNS数据合法。

# DNS测试工具
常见的DNS测试工具为 `dig`, `host`, `nslookup` 等。

# 参考
```c
# BIND9 - 最详细、最认真的从零开始的 BIND 9
https://www.cnblogs.com/doherasyang/p/14464999.html#11-dns

# DNS基础解析(3)
https://zhuanlan.zhihu.com/p/590032092

# Linux之DNS服务Bind
https://www.jianshu.com/p/149057c56344

# bind9 的官方英文文档
https://bind9.readthedocs.io/en/latest/

# bind9 
https://kb.isc.org/docs/aa-00726
```