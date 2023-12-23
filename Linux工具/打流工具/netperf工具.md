```table-of-contents
```
# 介绍
（1）Netperf是由惠普公司开发的一种网络性能测量工具，主要针对基于TCP或UDP的传输。
（2）Netperf根据应用的不同，可以进行不同模式的网络性能测试，即批量数据传输（bulk data transfer）模式和请求/应答（request/reponse）模式。
（3）Netperf测试结果所反映的是一个系统能够以多快的速度向另外一个系统发送数据，以及另外一个系统能够以多块的速度接收数据。

# 工作方式
Netperf 工具以client/server方式工作。
server端是netserver，用来侦听来自client端的连接。
client端是netperf，用来向server发起网络测试。

# 工作原理
在client与server之间，首先建立一个控制连接，用于传递有关测试配置的信息，以及测试的结果。在控制连接建立并传递了测试配置信息以后，client与server之间会再建立一个测试连接，用来来回传递着特殊的流量模式，以测试网络的性能。

# 工作流程
1> 建立控制连接：

server端netserver启动监听，监听来自client端netperf 的连接请求；
client端向server端发送控制连接请求，server端发现连接请求，建立控制连接。
控制连接创建完成，使用BSD socket传输信息，属于TCP连接。

2> 建立测试连接

lient端通过控制连接向server端传递测试配置信息。
server端获取测试配置信息，建立测试连接。
测试连接用于传输各种模式的流量测试网络的性能。

3> 测试网络性能

client端通过测试连接向server端发送Bulk模式流量模式的数据。
server端接受Bulk模式流量模式的数据并产生测试结果1。

client端通过测试连接向server端发送request/response流量模式的数据。
server端接受request/response流量模式的数据并产生测试结果2。

4> 输出测试结果

server端通过控制连接向client端发送测试结果。
client端接受到测试结果并显示或保存。

# 安装
```bash
wget -c "https://codeload.github.com/HewlettPackard/netperf/tar.gz/netperf-2.5.0" -O netperf-2.5.0.tar.gz
tar -zxvf netperf-2.5.0.tar.gz
cd netperf-netperf-2.5.0
./configure
make && make install
```

# 使用
#  网络性能测试分类
## 批量(bulk)网络流量的性能测试
根据使用传输协议的不同，批量数据传输又分为TCP批量传输和UDP批量传输。
### TCP_STREAM
Netperf缺省情况下进行TCP批量传输，即-t TCP_STREAM，用来测试进行TCP批量传输时的网络性能。
测试过程中，netperf向netserver发送批量的TCP数据分组，以确定数据传输过程中的吞吐量。

### UDP_STREAM
UDP_STREAM用来测试进行UDP批量传输时的网络性能。
## 请求/应答(request/response)网络流量的性能测试
在client/server结构中的request/response模式。在每次交易（transaction）中，client向server发出小的查询分组，server接收到请求，经处理后返回大的结果数据。

### TCP_RR
TCP_RR：测试**同一个TCP连接**中的多次TCP request和response的响应效率。
这种模式常常出现在数据库应用中。数据库的client程序与server程序建立一个TCP连接以后，就在这个连接中传送数据库的多次交易过程。
```javascript
[root@Netperf-test ~]# netperf -t TCP_RR -H 192.168.0.128
TCP REQUEST/RESPONSE TEST to 192.168.0.128
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size Size Time Rate
bytes  Bytes  bytes bytes   secs. per sec
16384  87380  1 1 10.00 9502.73
16384  87380
```

通过使用-r参数设置request和reponse分组的大小，可以进行更有实际意义的测试。
```javascript
[root@Netperf-test ~]# netperf -t TCP_RR -H 192.168.0.128 -- -r 32,1024
TCP REQUEST/RESPONSE TEST to 192.168.0.128
Local /Remote
Socket Size   Request  Resp.   Elapsed  Trans.
Send   Recv   Size Size Time Rate
bytes  Bytes  bytes bytes   secs. per sec
16384  87380  32 1024 10.00 4945.97
16384  87380
```
### TCP_CRR
TCP_CRR：TCP_CRR与TCP_RR不同,测试多个TCP连接中的request和response的响应效率，每个TCP请求、响应都建立一个新的TCP连接。
最典型的应用就是HTTP，每次HTTP交易是在一条单独的TCP连接中进行的。因此，由于需要不停地建立新的TCP连接，并且在交易结束后拆除TCP连接，交易率一定会受到很大的影响。
### UDP_RR
UDP_RR方式使用UDP分组进行request/response的交易过程。
# 参考
```c
https://bbs.huaweicloud.com/blogs/228744
```