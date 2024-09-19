```table-of-contents
```
# 介绍
DNSTAP是一种DNS报文日志格式协议，可以被用于诊断DNS服务器上异常解析问题。

多款DNS程序，支持了dnstap， 比如 bind9, coredns 等。
![](attachments/Pasted%20image%2020240304102335.png)



## 前提条件
`bind-9.11.26-2` 及其以上的版本软件才开始支持`dnstap`。
需要在编译`bind`时，使能`dnstap`选项。同时在配置`bind`时，也需要配置`dnstap`。
`dnstap`的输出数据，可以直接在`bind`中输出到文件中。更好的是，通过比如`unix socket`发送给`dnstap-server`进行，通过`dnstap-server`进行解析，然后写入到文件中。
![](attachments/Pasted%20image%2020240304151348.png)

## 编译
```bash
--enable-dnstap
```
`dnstap sender` 需要几个库，也需要在编译机器上同步进行安装。
```bash
# 下载
git clone https://github.com/google/protobuf
git clone https://github.com/protobuf-c/protobuf-c
git clone https://github.com/farsightsec/fstrm


# 编译和安装
autoreconf -i
configure
make; make install
```

## 作用
利用dnstap 可以查询 来自dns clinet的dns查询，经过 reslover， 转发到 dns forwarder，从dns forwarder 收到请求，以及响应给 dns client的 报文的流程。


# 原理

![](attachments/Pasted%20image%2020240304123313.png)
![](attachments/Pasted%20image%2020240304102829.png)

DNSTap 是一个非常有意思的插件。DNSTap 本身是一种灵活的二进制日志的协议，专门为 DNS 报文所设计的。左图中共有四条报文链路：

1. 客户端请求 CoreDNS
2. CoreDNS 请求上游 DNS 服务器
3. 上游 DNS 服务器响应 CoreDNS
4. CoreDNS 响应客户端

这四条报文均可以被 CoreDNS dnstap 插件以该二进制日志的方式投递到远程的 dnstap server 中，dnstap server 可以完成存储、分析、上报异常等动作。
dnstap 在发送这些报文的过程中采用了异步 IO 设计，**避免了本地磁盘日志写入和后续处理过程中报文反序列化流程**，可以实现低性能开销、高效的**报文采集**和异常诊断流程。

那么我们如何在 dnstap server 中实现报文的异常诊断呢？
dnstap server 是可以解析出原始的 DNS 报文的，从图中的 DNS 报文的 RCODE 状态码字段可以提取出 DNS 解析请求的响应状态码。不同的状态码体现了不同的异常类型，在实际问题排查过程中可以根据下表进行问题定位。
![](attachments/Pasted%20image%2020240304121243.png)

除了报文本身的 RCODE 字段外，我们可以根据 dnstap 报文的 Message Type 做一些异常判断。
![](attachments/Pasted%20image%2020240304121304.png)
当一个相同的 DNS Message ID 的报文仅产生了 ClientQuery 和 ForwarderQuery 时，说明上游并没有响应该 DNS 请求，CoreDNS 也没有响应给客户端。针对这样不同 Message Type 出现的组合，可以诊断出不同的场景。

# 配置
## 编辑 `options` 块启用 `dnstap`
```bash
options
{
	# ...
	dnstap { ( all | auth | client | forwarder |
          resolver | update ) [ ( query | response ) ];
          ... };
	dnstap-identity ( <quoted_string> | none |
          hostname );
    dnstap-output ( file | unix ) <quoted_string> [
          size ( unlimited | <size> ) ] [ versions (
          unlimited | <integer> ) ] [ suffix ( increment
          | timestamp ) ];      
	# dnstap输出，可以是一个文件，也可以是一个unix-socket，另外一个dnstap-server从这里读取，然后落盘。
	
	dnstap-version ( <quoted_string> | none );
	# ...
};
# end of options
```


范例，如下所示：
```none
dnstap {auth; resolver query;};
dnstap-output unix "/var/run/bind/dnstap.sock";
dnstap-identity "hostname";
dnstap-version "9.9.8-S5";
```

# 输出
## 输出到文件 
```

### DNS过滤器
指定您要记录的 DNS 流量类型，请将 `dnstap` 过滤器添加到 `/etc/named.conf` 文件中的 `dnstap` 块中。
可以使用以下过滤器：
- `auth` - 权威区域响应或回答。
- `client` - 内部客户端查询或回答。
- `forwarder` - 转发的查询或来自它的响应。
- `resolver` - 迭代的解析查询或响应。
- `update` - 动态区域更新请求。
- `all` - 以上选项中的任何一个。
- `query` 或 `response` - 如果您没有指定 `query` 或 `response` 关键字，则 `dnstap` 两个都记录。


`dnstap` 过滤器包含多个由 `;` 分隔的定义，`dnstap {}` 块的语法如下：`dnstap{(all | auth | client | forwarder | resolver | update)[(query | response)]; …​ };`

### 日志配置定期回滚
在以下示例中，`cron` 调度程序每天运行一次用户编辑的脚本的内容。
值为 `3` 的 `roll` 选项指定 `dnstap` 最多可以创建三个备份日志文件。
值 `3` 覆盖 `dnstap-output` 变量的 `version` 参数，并将备份日志文件数限制为三个。

二进制日志文件被移到另一个目录并被重命名，并且永远不会达到 `.2` 后缀，即使三个备份文件已存在。
如果根据大小限制自动回滚二进制日志足够了，则您可以跳过这一步。

```bash
Example:

sudoedit /etc/cron.daily/dnstap

#!/bin/sh
rndc dnstap -roll 3
mv /var/named/data/dnstap.bin.1 /var/log/named/dnstap/dnstap-$(date -I).bin

# use dnstap-read to analyze saved logs
sudo chmod a+x /etc/cron.daily/dnstap
```
## 输出到dnstap-server


```
fstrm_capture： farsightsec/fstrm 包编译生成的程序。
fstrm_capture： Receive and save Frame Streams data from a socket.
fstrm_dump ：   Display metadata and contents of Frame Streams file.

# fstrm_capture -t protobuf:dnstap.Dnstap -u /var/run/bind/dnstap.sock -w /var/tmp/example.dnstap

```
![](attachments/Pasted%20image%2020240304153928.png)

### fstrm 介绍
```bash
`fstrm`, a C implementation of the Frame Streams data transport protocol.
```
C 语言实现的传输数据帧的一个传输协议，开销很小，只有一个4B的头。可以在应用程序中，使用`libfstrm` 库进行传输数据帧，然后在 在 server端，比如`fstrm_capture`, `fstrm_dump`， 得到数据帧，或者打印数据帧的内容。
相比，直接在应用程序中打印日志，这种方式，通过网络通信（比如：unix socket）来减少日志落盘在这个应用程序上的CPU损耗。

# 读取
![](attachments/Pasted%20image%2020240304145521.png)

## `dnstap-read`
![](attachments/Pasted%20image%2020240304150401.png)

范例，如下所示：
![](attachments/Pasted%20image%2020240304145853.png)
# 性能
`dnstap`对性能的影响比较小，关闭`querylog`后 `dnstap` 关闭/开启的性能对比大概是`13W/S` VS `10W/S`。相比传统的`querylog`的性能影响实在好太多了。但是因为是二进制的文件，查看需要用`dnstap-read`还是非常不方便的。


# 参考
```c
# 使用 dnstap 记录 DNS 查询
https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/managing_networking_infrastructure_services/proc_recording-dns-queries-using-dnstap_assembly_setting-up-and-configuring-a-bind-dns-server

# dnstap 英文介绍
https://dnstap.info/
https://www.slideshare.net/MenandMice/dnstap


# bind 中 使用dnstap 
https://kb.isc.org/docs/aa-01342
https://bind9.readthedocs.io/en/v9.16.27/reference.html


# coredns中 dnstap 使用
https://zhuanlan.zhihu.com/p/474131897

# dnstap的pb格式
https://docs.bluecatnetworks.com/r/%E7%AE%A1%E7%90%86%E6%8C%87%E5%8D%97/DNS-%E6%9B%B4%E6%96%B0%E5%93%8D%E5%BA%94%E4%BA%8B%E4%BB%B6/9.3.0
```