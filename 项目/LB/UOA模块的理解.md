```table-of-contents
```
# 为什么需要获取客户端真实ip
## 基于client的ip地址可以做什么
ip地址是按地域分布的，服务器获取到客户端ip后可以做流量统计和分析、计费等；
服务器也可以针对客户端ip做一些定制化的功能，比如限流和黑白名单。

## 获取client的ip地址的难点
网络环境十分复杂，客户端发出的一个请求至少要经过cdn和负载均衡后才可以到达服务器。
报文经过负载均衡的时候往往会做FNAT，所以报文经过lb后，源地址就被删除了，这样服务器看到的源地址都是lb的地址，无法获取到客户端的真实ip了。


# 获取客户端真实ip的方式方法
各大lb厂商都有自己的办法来实现这个功能，具体的实现方式需要区分四层lb和七层lb。

## 四层负载均衡
四层负载均衡支持的协议一般是tcp和udp，四层负载均衡实现获取客户端真实ip功能的方法一般是`toa/uoa`和`proxy protocol`。

### toa/uoa

### proxy protocol
#### 原理
**tcp**
通过在 tcp载荷中添加一个很小的头信息，来方便的传递客户端信息（协议栈、源IP、目的IP、源端口、目的端口等)。其本质就是在三次握手后的第一个数据包中的tcp头部后面的payload添加了proxy protocol协议规定的内容，来向服务端传送客户端信息。

**udp**
udp当然也可以支持proxy protocol，只需将同一条流的第一个udp报文进行改造即可。

#### 特点
proxy protocol的特点如下：

（1）需要对后端服务代码进行改造
nginx天然支持proxy protocol，但是如果是自己实现的服务，则需要自己来实现解析proxy protocol的功能。



## 七层负载均衡
七层负载均衡支持的主要服务是http，七层负载均衡支持获取客户端真实ip的方法就比较简单且灵活，一般是通过插入一个http头部来实现，比如`X-Forwarded-For`字段，服务端可以直接通过这个http头部获取到客户端真实ip。

# 依赖uoa的转发模式
Fullnat模式才依赖uoa、toa；
如果是nat模式，dr模式，tunnel模式，则不依赖于uoa。

## 依赖于fullnat转发模式的场景
那么，哪些特性，又是依赖于fullnat转发模式的呢？
synproxy 特性
fnat64/fnat46(即ipv6到ipv4的转换)
client和 sever 同设备(使用相同的ip)


# UOA的原理
## dpvs中 uoa option 信息添加

dpvs 中 udp client 信息，又称之为 uoa option；下面主要从 dpvs 将 uoa option 信息插入到何处，插入的时机，插入的数据格式等几个方面进行展开讨论；

### dpvs 将  uoa option 信息插入到何处

dpvs  将  uoa option  插入到 ip 头 以及 udp 头之间。
为了方面 toa 模块识别，会：
1》ip头：更改 ip 头中proto = 248，ipHdr len长度保持不变；
2》udp头：udp的csum保持不变。

```text
注：
UDP的校验和需要计算UDP首部加数据荷载部分，加上UDP伪首部。
这个伪首部指，源地址、目的地址、UDP数据长度、协议类型（0x11），协议类型就一个字节，但需要补一个字节的0x0，构成12个字节。
```

### uoa 的格式

在 `ipv4/ipv6 hdr` 以及 udp 头之间 插入 `uoa option protocol` 及其载荷; 
拿ipv4举例: 变成了 `ipv4 Hdr[ipv4 option]` + `uoa option proto hdr` + `uoa 信息` + `udp 头及其载荷`;

具体变化如下：
**(1) Ipv4 hdr的变化**：
ipv4 hdr len 不变, 
ipv4 proto 更改为 IPPROTO_OPT(248), 
ipv4 total len 也会更改;
ipv4 csum 也发生变化

**(2) uoa option protocol hdr**：
4B: version(1B)+proto(1B)+len(2B) ：
vesion = ipv4, proto = udp(17/0x11), len = opp_hdr_len(4) + opp实体(8B)= 12(0x0C);

**(3) uoa 信息/opp实体：**
ipv4 uoa信息: 8B
ipv4 uoa 信息 = code(1B):len(1B):cport(2B):cip(4B)

**(4) udp 头及其载荷**：
不发生改变。

### 插入的时机
dpvs 只会在一个udp 连接的前n个数据包中插入 `uoa option` 相关信息，默认n为3；


## 后端Server 中 uoa 模块处理

### uoa 模块如何识别插入了client 信息的报文
收到 `ip proto = 248` 的报文，则认为是添加了 `uoa option`的 报文；

### uoa 模块对插入了 client 信息的报文的处理
经过 uoa 之后，将 `ip_hdr_len += 12`, `ip_proto = udp`, 即将 `option proto hdr` 以及 uoa 信息 共 12B信息放入到ip option中并设置为 0，重新计算 `iphdr checksum`。
另外会建立会话保存`client ip`，`client port` 以及 当前报文的五元组信息。

### 后端 server 如何获取 client 信息
后端server通过调用 如下API 接口 获取 `client ip`，`client port` 信息。
`getsockopt(fd, IPPROTO_IP, UOA_SO_GET_LOOKUP, &map, &mlen)`；`UOA_SO_GET_LOOKUP` 默认为 2048；
uoa中会基于当前的五元组查询会话，从会话取出 `client ip`，`client port` 返回给 server。

### uoa表项的刷新以及超时
每个服务程序调用 `getsockopt` 时，会查询 uoa表项，查询到之后，会将uoa表项的超时时间进行刷新。



# UOA存在的问题
## 低概率获取不到cip的问题

服务端的程序，在获取cip的时候，可能先收到了udp报文，而不是proto 248协议报文，此时就会无法获取到cip信息。

（1）proto 248报文和后续的udp报文到达server的时间的先后顺序
可能udp报文先到达，proto 248后到达，这个是可能的；因为这2种报文，被中间设备转发，也是放入到不同的队列中，可能存在不同的优先级处理登记。

（2）proto 248报文 和 后续的 udp报文被放入到server的不同的网卡的接收队列

（3）proto 248协议报文在网络中可能丢失，而udp不存在重传的问题
即使是quic协议，支持了重传，重传后的报文，也是无法识别是重传的，也是可能被按照普通的 udp来处理转发，而不会再次添加uoa信息。

## uoa表项的刷新机制和lb中conn的刷新不一致的问题

LB中的conn 的刷新是：无论 inbound 还是  outbound 方向 存在报文，都会刷新 conn。

uoa表项的刷新：
	调用getsockopt 获取cip的时候刷新
	 收到udp包的时候刷新
	 发送udp包的时候，暂时没有刷新


存在一种特殊情况：比如发送了一个udp的请求，后续服务器长时间给client发生响应，那么一直可以刷新LB中的conn的超时时间，但是Uoa表项的超时时间得不到刷新，进而导致超时。然后，client再次发送请求，由于可以在LB中查找到conn，不会再次添加uoa信息，但是 uoa中的表现超时了，报文中不存在uoa信息，无法再次构建uoa表项。


## uoa表项的超时时间 小于 lb中conn的超时时间的问题

如果uoa的表项的超时时间小于 LB中的conn的超时时间，则可能存获取CIP失败的问题。
比如：uoa中的 表项长期无法刷新，而被删除，但是LB中的conn依然存在，此时有流量到达LB，匹配到conn，则不会添加uoa信息(携带cip:cport信息)；然后流量到达后端RS，此时由于没有携带uoa信息，无法创建uoa表项，服务程序无法获取到 CIP:cport信息。

### 解决方法
（1）网路端：调整LB的vs下的conn的超时时间

（2）业务端：设置 heartbeat 、 keepalive 机制
对于一些重要的业务的长连接的流量，短期内没有流量，然后过一段时间，又存在了流量。
可以考虑，在业务端设置 heartbeat 、 keepalive 机制。定时刷新 uoa中表项，以及 LB的conn。

## uoa升级的问题

uoa升级的时候，由于 uoa 表项都是存放于内存中，升级过程中，需要卸载uoa，则uoa表项都会丢失。
uoa完成升级之后，存量的uoa表项都消失了，但是LB再次转发流量时，不会再次添加uoa信息，进而导致业务程序后续无法获取到 CIP:Cport信息。

### 解决
#### uoa侧 将内存中的uoa表项落盘
升级uoa之前，将uoa表项落盘；然后卸载旧的uoa，安装新的uoa，并从磁盘中加载uoa表项。

#### uoa和LB联动：
**（1）添加UOA的机制**：
LB转发流量时是否一直添加uoa信息，在于UOA是否给LB发送回复了 UOA ack消息，如果不存在uoa ack，则后续转发包时一直添加 uoa 信息。

**（2）升级流程**：
1> 在UOA升级之前，则通过扫描 uoa 表项，给LB发送 unack的消息；
同时，旧的uoa不会再给 LB发送  uoa ack消息；
2> LB收到 unack 消息之后，则转发包时添加uoa信息。
3> 卸载旧的uoa，安装新的uoa。
只有新的uoa，才会给LB回复uoa ack消息。



    
# 其他
## UOA支持容器

容器共享宿主机的内核「容器可以理解为一个进程」，容器在资源隔离方面主要依赖于命名空间（namespaces）和控制组（cgroups）。
因此，`UOA`支持容器，即==`UOA`支持`network namespace`,  是将`UOA`作为内核模块安装到宿主机上==。

### 内核模块的安装位置
#### 标准做法：在宿主机安装内核模块
- **内核共享性**：容器（包括 Kubernetes Pod）共享宿主机的内核，宿主机加载的模块对所有容器/Pod 直接生效。
- **权限与安全**：
    - 容器默认无权限加载内核模块（需要 `--privileged` 或 `CAP_SYS_MODULE` 权限）。
    - 在宿主机统一管理内核模块更安全，避免容器因权限过高导致的安全风险。
- **维护性**：宿主机统一管理模块，避免容器销毁后模块丢失。

#### 容器内安装内核模块的注意事项
一般，==不建议在容器内安装内核模块==。理由如下：
（1）容器需要特殊权限
容器默认无权限加载内核模块（需要 `--privileged` 或 `CAP_SYS_MODULE` 权限）。

（2）风险
- **破坏宿主机稳定性**：容器内加载的劣质模块可能导致宿主机内核崩溃。
- **安全漏洞**：特权容器可能被攻击者利用，通过内核模块提权。

### 宿主机上的内核模块对容器(Pod) 的影响
#### 内核模块全局可见
宿主机加载的模块会直接影响所有容器/Pod，因为容器共享宿主机内核.
```bash
# 宿主机加载网络驱动模块（如 veth、bridge） 
sudo modprobe bridge 

# 容器/Pod 的网络栈将自动使用该模块
```
#### 典型场景
- **网络功能**：宿主机加载 `veth`、`bridge`、`ip_tables` 等模块，容器/Pod 依赖这些模块实现网络通信。
- **存储功能**：宿主机加载文件系统模块（如 `overlayfs`、`ext4`），容器/Pod 的存储卷依赖这些模块。
- **硬件访问**：宿主机加载设备驱动模块（如 GPU 驱动 `nvidia`），容器/Pod 可通过设备映射（如 `--device=/dev/nvidia0`）使用硬件。


### UOA支持虚拟机
容器通常被认为是轻量级的虚拟化技术，但它们实际上并不是完全的虚拟机。容器共享宿主机的内核，这意味着容器内的进程直接运行在宿主机的内核上。而虚拟机则有自己的独立内核。因此，容器在资源隔离方面主要依赖于命名空间（namespaces）和控制组（cgroups），而不是硬件虚拟化。

因此，可以在虚拟机上安装内核模块，因为**虚拟机是硬件虚拟化，虚拟机存在独立的操作系统，可以安装内核模块**。
而**容器共享宿主机的内核「容器可以理解为一个进程」，不要在容器上安装内核模块**。容器在资源隔离方面主要依赖于命名空间（namespaces）和控制组（cgroups）。

## wireshark 解析含有uoa的报文
mac的 `wireshark`，在`/Applications/Wireshark.app/Contents/Resources/share/wireshark/init.lua` 文件最后追加一行 `dofile(USER_DIR.."uoa.lua")`，然后在`/Users/<用户名>/.config/wireshark/目录下创建uoa.lua` 文件，写入下面的代码

```lua
local p_uoa = Proto("uoa", "UDP option address");

p_uoa.fields.ver = ProtoField.uint8("uoa.ver", "uoa version")
p_uoa.fields.proto = ProtoField.uint8("uoa.proto", "uoa next protocol")
p_uoa.fields.tlen = ProtoField.uint8("uoa.tlen", "uoa total length")
p_uoa.fields.op = ProtoField.uint8("uoa.opcode", "uoa option opcode")
p_uoa.fields.olen = ProtoField.uint8("uoa.olen", "uoa option length")
p_uoa.fields.cport = ProtoField.uint16("uoa.cport", "client port")
p_uoa.fields.cip = ProtoField.ipv4("uoa.cip", "client ip")
p_uoa.fields.cip6 = ProtoField.ipv6("uoa.cip6", "client ip6")

function p_uoa.dissector(buf, pkt, tree)
    if (buf:len() < 12) then return end
    if (buf(1,1):uint() ~= 17) then return end
    local udp_offset = 12;
    local subtree
    if (buf(2,2):uint() == 12) then
        if (buf(4,1):uint() ~= 31) then return end
        if (buf(5,1):uint() ~= 8) then return end
        subtree = tree:add(p_uoa, buf(0, 12))
        subtree:add(p_uoa.fields.cip, buf(8, 4))
    elseif (buf(2,2):uint() == 24) then
        if (buf(4,1):uint() ~= 31) then return end
        if (buf(5,1):uint() ~= 20) then return end
        subtree = tree:add(p_uoa, buf(0, 24))
        subtree:add(p_uoa.fields.cip6, buf(8, 16))
        udp_offset = 24
    else
        return
    end
    subtree:add(p_uoa.fields.ver, buf(0, 1))
    subtree:add(p_uoa.fields.proto, buf(1, 1))
    subtree:add(p_uoa.fields.tlen, buf(2, 2))
    subtree:add(p_uoa.fields.op, buf(4, 1))
    subtree:add(p_uoa.fields.olen, buf(5, 1))
    subtree:add(p_uoa.fields.cport, buf(6, 2))
    local dissector = Dissector.get("udp")
    dissector:call(buf(udp_offset):tvb(), pkt, tree)
end

local uoa_encap_table = DissectorTable.get("ip.proto")
uoa_encap_table:add(248, p_uoa)
```


# 参考
```bash
# 获取客户端真实ip的方法
https://blog.csdn.net/yuubeka/article/details/124957150
```