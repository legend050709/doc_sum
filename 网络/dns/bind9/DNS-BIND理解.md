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
## BIND的工作端口
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
负责至少解析一个域内的域名，维护所负责解析的域数据库，可对该域数据进行读写操作
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

## 主从DNS之间的区域传送机制

- `axfr` : 全量传送，一般为从服务器第一次读取，将会传送整个数据库。
- `ixf` : 增量传送，仅传送变化的数据。


# bind配置文件
## 配置总体概述
`/var/named/` : 资源记录文档的存放位置
`/var/named/named.ca` ：13个根服务器存放文件
`/var/named/named.empty`：
`/var/named/named.localhost`：
`/var/named/named.loopback`：

`/etc/logrotate.d/named` ：日志切割配置文件

`/etc/named.conf`： 主配置文件
`/etc/named.rfc1912.zones`： 区域配置文件（用include指令包含在主配置文件）
`/etc/named.root.key`： 根区域的key文件以实现事务签名；
`/etc/rndc.key`： rndc加密密钥

`/etc/rndc.conf`： rndc（远程名称服务器控制器）配置文件

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


## 主配置文件中的acl 块
一般来说，ACL模块用来控制哪些主机可以访问域名解析服务器，其设置不会让控制文件的配置非常冗余和庞大。
采用这个配置可以有效防范DOS以及Spoofing攻击。

ACL匹配客户端是否能够接入到域名服务器基于三个基本的特征:
- 客户端的IPv4或者IPv6地址
- 用于签署请求的 TSIG 和 SIG(0) 密钥
- 在DNS客户端子网选项中编码的前缀地址


## 主配置文件中的logging 块
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
`channel`的作用是指定输出的方式、日志格式的选项和事件的严重性，每一个`channel`可以指定一个 `category` 来指定记录的事件类型。

- `category`
 `category` 用来区分不同的事件产生的类别或者场景，比如：客户端请求（`client request`）、配置文件解析处理（`Configuration file parsing and processing`）。

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

`named.rfc1912.zones`主配置文件的区域配置
```c
zone "ZONE_NAME" IN {
    type {master|slave|hint|forward};
    file "ZONE_NAME.zone";
};
```

-  `zone "ZONE_NAME“ `：定义解析库名字
通常和解析库文件前缀对应起来。
-  `type `：
	-  `master `：指的是主dns解析
	-  `slave `：指的是从dns解析
	-  `hint `：指的是根域名解析（根提示域）
	-  `forward `：指的是转发，转发不使用file
-  `file ` ：指定存放dns记录的数据文件名称（位置默认在/var/named下面）
file的前缀通常和zone的名字通常对应起来，然后加一个.zone的后缀。
-  `allow-update `：是否允许客户主机或服务器自行更新dns记录。
- `allow-transfer`: 用来给出 Failover 或者是 递归查询DNS服务器的IP地址，如果之前在 `options` 里配置的`allow-transfer` 如果设置成了参数 `yes`， 那么需要在这里指出递归查询服务器的IP地址；


>注：自定义的配置解析库文件（Zone files）,一般是在/var/named下写，文件名格式一般写为ZONE_NAME.zone。


### 区域文件(Zone文件)
zone文件：保存 RR (Record Resource) 信息的文件。
zone文件包括正向Zone文件和反向Zone文件。
> 注：在 zone file 中，注释的符号是`;`。

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
`@`：代表当前的区域；
`$ORIGIN [REGION NAME]` 设定了当前的解析域名区域；
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

注意：
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
反解域也是要有SOA记录的，在反解域中ns记录就不用在写A记录了，因为反解区域文件只可以有PTR，不可以有A记录。
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

检查配置文件：
```c
named-checkzone  zhangqifei.top  /var/named/zhangqifei.top.zone
```

### 反向区域文件的配置范例

例子如下：
```c

zone "111.168.192.in-addr.arpa" IN {
    type master;
    file "192.168.111.zone";
};
配置/var/named/ZONE_NAME.zone
不需要MX、A、AAAA，要有SOA、NS记录，PTR记录。
```
配置192.168.111.zone：
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
```