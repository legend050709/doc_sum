```table-of-contents
```
# 背景
调试过网络程序的人大多使用过`tcpdump`，但你知道`tcpdump`是如何工作的吗？ `tcpdump`这类工具也被称为`Sniffer`，它可以在不影响应用程序正常报文的情况下，将流经网卡的报文（二层+三层+四层+数据）呈现给用户。
本文不分析`tcpdump`的具体实现，而只是借`tcpdump`来揭示一些网络编程中一个大多数人都容易忽略的一个主题：`Socket`参数对用户接收报文的影响。

# socket API
相信所有接触过`Socket`编程的人都应该认识下面这个`API`
```
#include <sys/socket.h>
sockfd = socket(int socket_family, int socket_type, int protocol);
```
 对于`TCP`或者`UDP`应用的开发者来说，他们可以很容易地从互联网上找(抄)到这样的例子：
```
/* 创建TCP socket*/
sockfd = socket(AF_INET, SOCK_STREAM, 0);

/* 创建UDP socket*/
sockfd = socket(AF_INET, SOCK_DGRAM, 0)
```

为什么第一个参数要使用`AF_INET`,为什么第二个参数要使用`SOCK_STREAM`或者`SOCK_DGRAM`，为什么第三个参数要填`0` ? 

## socket_family
第一个参数表示创建的`socket`所属的`地址簇`或者`协议簇`，取值以`AF`或者`PF`开头定义在(`include\linux\socket.h`)，实际使用中并没有区别(有两个不同的名字只是因为是历史上的设计原因)。
> 最常用的取值有`AF_INET`,`AF_PACKET`,`AF_UNIX`等。`AF_UNIX`用于主机内部进程间通信，本文暂且不谈。

`AF_INET`与`AF_PACKET`的区别在于使用**前者只能看到`IP`层以上的东西，而后者可以看到链路层的信息**。
为了说明这个问题，我们需要知道网络报文的分类。如下图所示：
`Ethernet II`帧是应用最为广泛的帧类型(当然也有像`PPP`这样的其他链路帧类型)。`Ethernet II`帧内部，又可大致分为`IP`报文和其他报文。
我们熟悉的`TCP`或者`UDP`报文都属于`IP`报文。
![](attachments/Pasted%20image%2020231204144740.png)
`AF_INET`是与`IP`报文对应的，而`AF_PACKET`则是与`Ethernet II`报文对应的。
`AF_INET`创建的套接字称为`inet socket`，而`AF_PACKET`创建的套接字称为`packet socket`。
![](attachments/Pasted%20image%2020231204144817.png)

## socket_type & protocol
第一个参数`family`会影响第二个参数`socket_type`和第三个参数`protocol`取值范围。
### socket_type
第二个参数`socket_type`表示套接字类型。它的取值不多，常见的就以下三种

```
enum sock_type {
	SOCK_STREAM	= 1,     /* stream (connection) socket  */
	SOCK_DGRAM	= 2,     /* datagram (conn.less) socket */
	SOCK_RAW	= 3,     /* raw socket                  */
};

所有的 sock type:
enum sock_type {
    SOCK_STREAM = 1,
    SOCK_DGRAM  = 2,
    SOCK_RAW    = 3,
    SOCK_RDM    = 4,
    SOCK_SEQPACKET  = 5,
    SOCK_DCCP   = 6,
    SOCK_PACKET = 10,
};
```
SOCK_STREAM 是面向数据流的，协议 IPPROTO_TCP 属于这种类型。
SOCK_DGRAM 是面向数据报的，协议 IPPROTO_UDP 属于这种类型。如果在内核里面看的话，IPPROTO_ICMP 也属于这种类型。
SOCK_RAW 是原始的 IP 包，IPPROTO_IP 属于这种类型。

### protocol
第三个参数`protocol`表示套接字上报文的协议。

- 对于`AF_INET`地址簇
`protocol`的取值范围:如 **IPPROTO_TCP**、 **IPPROTO_UDP**、 **IPPROTO_ICMP** 这样的`IP`报文协议类型，或者 **IPPROTO_IP = 0** 这个特殊值。
- 对于`AF_PACKET`地址簇
`protocol`的取值范围是 **ETH_P_IP**、 **ETH_P_ARP**这样的以太帧协议类型。

# af inet 域 socket
## inet socket的协议开关表
每一个`inet socket`只能收发一种`IP`协议类型的报文，这是在套接字创建的时候就决定的(`protocol`参数)，比如`TCP`套接字是不能收发`UDP`报文的，反之也是一样。
> 注：`protocol`的值还受到`socket_type`的限制，不匹配的取值会导致套接字创建操作会返回失败。


```
/* 错误取值，返回失败 */
sockfd = socket(AF_INET, SOCK_DGRAM, IPPROTO_TCP);
```

内核通过`协议开关表`记录了哪些哪些取值是有效的，`inet`在初始化时会将支持的协议注册在`协议开关表`中的以`socket_type`为`KEY`的链表上：
![](attachments/Pasted%20image%2020231204150216.png)
而在创建套接字时，`inet_create`会在协议开关表中根据`socket_type`和`protocol`进行匹配
```
list_for_each_entry_rcu(answer, &inetsw[sock->type], list) {
		err = 0;
		/* Check the non-wild match. */
		if (protocol == answer->protocol) {
			if (protocol != IPPROTO_IP)
				break;
		} else {
			/* Check for the two wild cases. */
			if (IPPROTO_IP == protocol) {
				protocol = answer->protocol;
				break;
			}
			if (IPPROTO_IP == answer->protocol)
				break;
		}
		err = -EPROTONOSUPPORT;
	}
```
`IPPROTO_IP`的值为`0`, 在用户使用`0`作为创建套接字的第三个参数时，会匹配到该链表上的第一个协议，这正是创建`TCP`或者`UDP`套接字时，第三个参数可以为`0`的原因, `0`表示由内核自动选择。
```
/* 创建TCP socket*/
sockfd = socket(AF_INET, SOCK_STREAM, 0);

/* 创建UDP socket*/
sockfd = socket(AF_INET, SOCK_DGRAM, 0)
```

## raw inet socket
对于`inet socket`来说，一个`TCP`报文可以这样分解：

```
packet = IP Header + TCP Header +  Payload
```
如果我们是使用`SOCK_STREAM`创建的`TCP`套接字，应用程序在通过`send`发送数据时，只需要提供`Payload`就行了，而`IP Header`和`TCP Header`则由内核组装完成。接收方向，应用程序通过`recv`也只能收到`payload`。

### 手动组装四层头以及载荷 
`RAW inet socket`套接字则为应用提供了更底层的控制能力
```
int s = socket (AF_INET, SOCK_RAW, IPPROTO_TCP);
```
使用上面的接口可以创建一个更原始的`TCP`套接字，当我们使用这个套接字发送数据时，需要提供`Payload`和`TCP Header`，而`IP Header`依然由内核协议栈自动组装。

### 手动组装三层头以及载荷
如果希望手动组装`IP Header`，有两个方法：

第一种是`protocol`使用`IPPROTO_RAW`
```
int s = socket (AF_INET, SOCK_RAW, IPPROTO_RAW);
```

第二种是置位`IP_HDRINCL`的套接字选项。
```
int s = socket (AF_INET, SOCK_RAW, IPPROTO_TCP);

int one = 1;
const int *val = &amp;one;
if (setsockopt (s, IPPROTO_IP, IP_HDRINCL, val, sizeof (one)) &lt; 0)
{
	printf (&quot;Error setting IP_HDRINCL. Error number : %d . Error message : %s \n&quot; , errno , strerror(errno));
	exit(0);
}
```
# af packet 域 socket
## 背景
Linux下有两种方式接收数据链路层的数据包：  
    （1）原始的方法，即创建一个类型为SOCK_PACKET的**af inet 域 socket**，该方法很普遍，但是缺乏灵活性；  
    （2）最新的方法，引入了帧过滤功能和性能上的提升，即创建一个指定协议簇为 PF_PACKET的socket，这需要root权限（类似于创建一个raw socket），并且socket的第三个参数必须指定一个以太网帧类型(Ethernet frame type)。
```c
socket(PF_PACKET, SOCK_RAW, htons(ETH_P_XXX))    // 发送、接收数据链路层数据帧（目前只有Linux支持）
socket(AF_INET, SOCK_PACKET, htons(ETH_P_XXX))   // 过时了
```

## af packet  域 的socket 和 af inet 域socket区别
![](attachments/Pasted%20image%2020231204154034.png)
**`inet socket`的控制范围是`IP`报文，而`packet socket`的控制范围扩大到了以太层报文**。

对于`af packet`域的socket , 第二个参数`socket_type`只能选择`SOCK_DGRAM`、`SOCK_RAW`, `protocol`则表示支持的网络层的协议类型。
> 注：`SOCK_DGRAM`和`SOCK_RAW`的区别就在于，在接收方向，使用`SOCK_DGRAM`套接字的应用程序收到的报文已经去除了`Ethernet Header`，而`SOCK_RAW`套接字则会保留。

## Protocol Handler
### 介绍
对**以太帧**来说，不同的网络层协议类型(比如`IP` `ARP` `PPPoE`)有不同的接收处理函数。在内核中，这就是`Protocol Handler`。
内核中的`Protocl Handler`是这样组织的：
![](attachments/Pasted%20image%2020231204151841.png)

`Protocl Handler`在`dev`下存在`ptype_all`链表和`ptype_base`链表。
无论网卡是否采用`NAPI`，内核最终都会调用到`__netfi_receive_skb`接收报文，这个函数会遍历`ptype_all`链表上已注册的`handler`，然后再遍历`ptype_base`特定协议链上的所有已注册的`handler`。

### `Protocol handler`的注册
`handler`的注册是通过`dev_add_pack`完成的,如果没有指定协议(即`proto=ETH_P_ALL`)，该`handler`就会注册在`ptype_all`上(`tcpdump`默认就会注册在这里)，否则根据协议注册在`ptype_base`的某条链表上。

> 注：在报文接收过程中，**同一个`skb`会被`deliver_skb`到多个`handler`**(至少将`ptype_all`链表上的`handler`走一遍)。

#### AF_INET域套接字的注册
内核启动时，`inet`会注册一个`handler`，它支持`IP`协议，**所有`AF_INET`套接字实际上是共用这样一个`handler`**，对应的接收函数是`ip_rcv`，区分是哪一个套接字的报文是之后的工作。

```
/* net/ipv4/af_inet.c */
static struct packet_type ip_packet_type __read_mostly = {
	.type = cpu_to_be16(ETH_P_IP),
	.func = ip_rcv,
};

static int __init inet_init(void)
{
    // code omitted
    dev_add_pack(&ip_packet_type);
    // code omitted
}
```

#### AF_PACKET域套接字的注册
对于`AF_PACKET`，`handler`是在`packet_create`中单独注册的，也就是说，**每个`AF_PACKET`套接字拥有独立的`handler`**。
```
static int packet_create(struct net *net, struct socket *sock, int protocol,
			 int kern)
{
    // code omitted
    po->prot_hook.func = packet_rcv;   
    // code omitted
    register_prot_hook(sk);  // 这里面去 dev_add_pack
}
```
单独的`handler`，使得在接收函数`packet_rcv`的时候，就已经可以知道这是属于哪一个套接字的数据了。

## raw packet socket对报文的二层头以及载荷处理
对于**`AF_PACKET`域 socket**, 来说，一个报文可以这样分解：
```
packet = Ethernet Header + Payload
```
而`SOCK_DGRAM`和`SOCK_RAW`的区别就在于，在接收方向，使用`SOCK_DGRAM`套接字的应用程序收到的报文已经去除了`Ethernet Header`，而`SOCK_RAW`套接字则会保留。

接收完整的数据链路层数据包,可以这样写：
```c
fd = socket(AF_INET, SOCK_PACKET, htons(ETH_P_ALL)); 
/* older systems use af inet 域 socket */
或者
fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL)); 
/* newer systems use af packet 域 socket */
```

如果我们只需要IPv4帧，可以这样写：
```
fd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_IP)); /* new systems */
或者
fd = socket(AF_INET, SOCK_PACKET, htons(ETH_P_IP)); /* older systems */
```
其它数据链路层类型定义的常量参数有 ETH_P_ARP 和 ETH_P_IPV6；
指定协议(protocol)为ETH_P_XXX即告诉数据链路层我只接收该类型的帧(frame)，socket会自动过滤。


## raw packet socket的创建流程
应用层使用：
```c
socket(AF_PACKET, SOCK_RAW, ETH_P_ALL)
```


内核中的流程：
```c
SYSCALL_DEFINE3(socket, int, family, int, type, int, protocol)
{
......
    retval = sock_create(family,  type, protocol, &sock);
......
}


int sock_create(int family, int type, int protocol, struct socket **res)
{
    return __sock_create(current->nsproxy->net_ns, family, type, protocol, res, 0);
}


int __sock_create(struct net *net, int family, int type, int protocol,
struct socket **res, int kern)
{
......
    pf = rcu_dereference(net_families[family]);
......
    err = pf->create(net, sock, protocol, kern);
......
}

static const struct net_proto_family packet_family_ops = {
    .family =    PF_PACKET,
    .create =    packet_create,
    .owner    =    THIS_MODULE,
};

static int packet_create(struct net *net, struct socket *sock, int protocol,
             int kern)
{
    struct sock *sk;
    struct packet_sock *po;
    __be16 proto = (__force __be16)protocol; /* weird, but documented */
......
    sk = sk_alloc(net, PF_PACKET, GFP_KERNEL, &packet_proto);
......
    sock->ops = &packet_ops;
......
    po = pkt_sk(sk);
    sk->sk_family = PF_PACKET;
    po->num = proto;
......
    po->prot_hook.func = packet_rcv;
......
    po->prot_hook.af_packet_priv = sk;
    if (proto) {
        po->prot_hook.type = proto;
        register_prot_hook(sk);
    }
......
}
```

## af packet 域 socket对报文的匹配
我们知道，一个 `Skb` 匹配 `AF INET 类型的 TCP protocol  socket` 的 TCP protocol ，则需要 `skb` 的四元组 先和 `connected` 状态的 `tcp inet socket` 进行匹配。
如果无法匹配上，则二元组`（dip, dport）` 和 `listen` 状态的  `tcp inet socket` 进行匹配。
如果还是无法匹配上，则`(INADDR_ANY)` 1-tuple，寻找有没有 listening 状态的 socket。

但是对于`af packet` 域 socket，只要报文和 设置的socket的proto 匹配。那么就可以匹配到对应的 socket，然后进行对应的处理。这也是tcpdump，可以抓取到本机的所有流的以太网包的原因。

## af packet 域 socket 与 tcpdump

`tcpdump`是如何完成嗅探工作的呢？ 没错！它正是使用的`af packet`域的 socket：
- `tcpdump`作为`Sniffer`，它不能影响正常的报文收发，因此它需要单独的`protocol handler`，这样内核接收的报文会复制一份后，交给`tcpdump`
- `tcpdump`不止能抓取`IP`报文, 它还可以抓起链路层信息或者其他一些非`IP`报文。

# SOCK_RAW type的套接字
SOCK_RAW type 套接字编程可以接收到本机网卡上的数据帧或者数据包，对与监听网络的流量和分析是很有作用的。
一共可以有 `3` 种方式创建这种socket：
```c
socket(AF_INET, SOCK_RAW, IPPROTO_XXX)           // 发送、接收网络层IP数据包
socket(PF_PACKET, SOCK_RAW, htons(ETH_P_XXX))    // 发送、接收数据链路层数据帧（目前只有Linux支持）
socket(AF_INET, SOCK_PACKET, htons(ETH_P_XXX))   // 过时了
```


## af inet 域的 SOCK_RAW type socket的使用
```c
socket(AF_INET, SOCK_RAW, IPPROTO_XXX)
```
其中，`IPPROTO_XXX` 是`/etc/protocols`文件中描述的protocol，指明要接收的包含在IP数据包中的协议包。
### 接收数据
这个SOCKET接收的数据是包含IP头的IP数据包（不管protocol是任何协议），并且是已经重组好的IP包。

### 发送数据
默认情况下，发送的数据是不包含IP头的IP数据包负载部分（如TCP或UDP包），网络层会自动添加IP头；如果使用`setsocketopt`函数设置了`IP_HDRINCL`选项后，写入的数据就必须包含IP头，即IP头在用户层由使用者自己构建。例子：
```c
int on = 1;
if (setsockopt(sockfd, IPPROTO_IP, IP_HDRINCL, &on, sizeof(on)) < 0) {
    printf("Set IP_HDRINCL failed\n");
}
```

### 其他
（1）如果protocol是`IPPROTO_RAW(255)`，这时候，这个socket只能用来发送IP包，而不能接收任何的数据。发送的数据需要自己填充IP包头，并且自己计算校验和。

（2）对于protocol为0（`IPPROTO_IP`)的raw socket。用于接收任何的IP数据包。其中的校验和和协议分析由程序自己完成。
> 另一种说法：对于domain为`AF_INET`，type为`SOCK_RAW`的socket来说，protocol不能为0（即`IPPROTO_IP`）。 经过在Go语言下测试（通过系统调用syscall.Socket），对protocol参数使用0时，报错：`protocol not supported`。

（3）如果protocol既不是0也不是255，那么sock_raw既可以接收数据包，也可以发送数据包。

（4）虽然设置 `IP_HDRINCL` 选项，可以由使用者自行指定IP头，但 `IP_HDRINCL` 选项还是会修改使用者指定的IP头，规则如下：
> A. IP 头校验和（CheckSum）：总是填充。换句话说就是，使用者在指定IP头，不需要处理IP校验和。
> B. 源IP地址：如果为 0，则被自动填充为本机的 IP 地址。
> C. 包ID（packet ID）：如果为 0，则被自动填充。
> D. IP包的总长度：总是被填充。换句话说就是，IP头部的关于IP数据包总长度字段不需要使用者来处理。

（5）如果 raw socket 没有使用 connect 函数绑定对方地址时，则应使用 `sendto` 或 `sendmsg` 函数来发送数据包，在函数参数中指定对方地址。如果使用了 connect 函数，则可以直接使用 `send`、`write` 或 `writev` 函数来发送数据包。

（6）如果 raw socket 调用了 `bind` 函数，则发送数据包的源 IP 地址将是 `bind` 函数指定的地址。否则，内核将以发接口的主 IP 地址填充；如果设置了 `IP_HDRINCL` 选项，则必须手工填充每个发送数据包的源 IP 地址。

这种套接字用来写个ping程序比较适合。

## af packet 域的SOCK_RAW type socket使用
```c
socket(PF_PACKET, SOCK_RAW, htons(ETH_P_XXX))
```
这种套接字可以监听网卡上的所有数据帧；
> 最后的以太网CRC从来都不算进来的，因为网卡已经判断过了，对程序来说没有任何意义了。

### 接收数据
A. 发往本地mac的数据帧
B. 从本机发送出去的数据帧(第3个参数需要设置为 ETH_P_ALL)
C. 非发往本地mac的数据帧(网卡需要设置为 promisc 混杂模式)

协议类型一共有四个：
```c
ETH_P_IP   0x800     只接收发往本机mac的IP类型的数据帧
ETH_P_ARP  0x806     只接收发往本机mac的ARP类型的数据帧
ETH_P_RARP 0x8035    只接收发往本机mac的RARP类型的数据帧
ETH_P_ALL  0x3       接收发往本机mac的所有类型（IP、ARP、RARP）的数据帧，并接收从本机发出的所有类型的数据帧
					（混杂模式打开的情况下，会接收到非发往本地mac的数据帧）
```

### 发送数据
需要自己组织整个以太网数据帧，所有相关的地址使用`struct sockaddr_ll` 而不是 `struct sockaddr_in`（因为协议簇是 `PF_PACKET` 不是 AF_INET 了）

### 范例
```c
    int sockfd = socket(PF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    struct sockaddr_ll sll;
    memset( &sll, 0, sizeof(sll) );
    sll.sll_family = AF_PACKET;
    struct ifreq ifstruct;
    strcpy(ifstruct.ifr_name, "eth0");
    ioctl(sockfd, SIOCGIFINDEX, &ifstruct);
    sll.sll_ifindex = ifstruct.ifr_ifindex;
    sll.sll_protocol = htons(ETH_P_ALL);
    if(bind(fd, (struct sockaddr *) &sll, sizeof(sll)) == -1 )
        perror("bind()");

    int set_promisc(char *interface, int fd) {
        struct ifreq ifr;
        strcpy(ifr.ifr_name, interface);
        if(ioctl(fd, SIOCGIFFLAGS, &ifr) == -1) {
            perror("iotcl()");
            return -1;
        }
        ifr.ifr_flags |= IFF_PROMISC;
        if(ioctl(fd, SIOCSIFFLAGS, &ifr) == -1) {
            perror("iotcl()");
            return -1;
        }
        return 0;
    }

    int unset_promisc(char *interface, int fd) {
        struct ifreq ifr;
        strcpy(ifr.ifr_name, interface);
        if(ioctl(fd, SIOCGIFFLAGS, &ifr) == -1) {
            perror("iotcl()");
            return -1;
        }
        ifr.ifr_flags &= ~IFF_PROMISC;
        if(ioctl(fd, SIOCSIFFLAGS, &ifr) == -1) {
            perror("iotcl()");
            return -1;
        }
        return 0;
    }
```

## af packet 域的SOCK_RAW type socket接收数据包的原理

### 1. 网卡对该数据帧进行硬数据链路层过滤

首先进行数据链路层校验和处理，如果校验和出错，直接仍掉；然后根据网卡的模式不同会有不同的动作：如果设置了promisc混杂模式的话，则不做任何过滤直接交给下一层输入例程，否则非本机mac或者广播mac会被直接丢弃。

### 2. 向用户层递交数据链路层数据帧———— SOCK_RAW捕获数据链路层数据帧

在进入网络层之前，系统会检查系统中是否有通过`socket(AF_PACKET, SOCK_RAW, ...)`创建的套接字；如果有并且与指定的协议相符的话，系统就给每个这样的socket接收缓冲区发送一个数据帧拷贝。然后进入网络层。

## af inet 域的 SOCK_RAW type socket 接收数据包的原理

### 1.网卡对该数据帧进行数据链路层过滤

首先进行数据链路层校验和处理，如果校验和出错，直接仍掉；然后根据网卡的模式不同会有不同的动作：如果设置了promisc混杂模式的话，则不做任何过滤直接交给下一层输入例程，否则非本机mac或者广播mac会被直接丢弃。

### 2. 进入网络层IP层过滤

IP层会对该数据包进行软过滤————就是检查校验或者丢弃非本机IP或者广播IP的数据包等。

### 3. 向用户层递交网络层数据包 ———— SOCK_RAW捕获网络层IP数据包

在进入运输层（如TCP、UDP例程）之前，系统会检查系统中是否有通过`socket(AF_INET, SOCK_RAW, ...)`创建的套接字；如果有的话并且协议相符，系统就给每个这样的`socket`接收缓冲区发送一个数据包拷贝（会包含IP数据包头）。


## 应用
SYN flooder 程序，比如：hping的程序。

## 其他
**程序中使用 Inet sock_raw type +  tcp proto 的socket 为什么只写发SYN包？**

因为你发了SYN包，对方发回来SYN/ACK包，你的操作系统内核先于你的程序接收到这个包，它检查内核里的socket，发现没有一个socket对应于这个包（因为raw tcp socket没有保存ip和端口等信息，所以内核不能识别这个包），所以发了一个RST包给对方，于是对方的tcp socket关闭了。
接下来你的raw tcp socket最终收到这个SYN/ACK包，你的程序做了处理后，再发ACK包给对方时，对方的tcp socket已经关闭，所以对方就发了一个RST回来。

> 因此，要写一个SYN flooder就只能用 inet raw + raw  proto socket，因为raw tcp不能自己控制IP头，所以不能写SYN flooder，除非用了IP_HDRINCL选项和自己构造IP头部。

## af inet raw socket 范例
一个ping程序样例
```c
#include "stdio.h"
#include "stdlib.h"
#include "string.h"

#include "unistd.h"
#include "sys/types.h"
#include "sys/socket.h"
#include "netinet/in.h"
#include "netinet/ip.h"
#include "netinet/ip_icmp.h"
#include "netdb.h"
#include "errno.h"
#include "arpa/inet.h"
#include "signal.h"
#include "sys/time.h"

extern int errno;

int sockfd;
struct sockaddr_in addr; //peer addr
char straddr[128];       //peer addr ip(char*)
char sendbuf[2048];
char recvbuf[2048];
int sendnum;
int recvnum;
int datalen = 30;


unsigned short my_cksum(unsigned short *data, int len) {
    int result = 0;
    for(int i=0; i<len/2; i++) {
        result += *data;
        data++;
    }
    while(result >> 16)result = (result&0xffff) + (result>>16);
    return ~result;
}
void tv_sub(struct timeval* recvtime, const struct timeval* sendtime) {
    int sec = recvtime->tv_sec - sendtime->tv_sec;
    int usec = recvtime->tv_usec - sendtime->tv_usec;
    if(usec >= 0) {
        recvtime->tv_sec = sec;
        recvtime->tv_usec = usec;
    } else {
        recvtime->tv_sec = sec-1;
        recvtime->tv_usec = -usec;
    }
}

void send_icmp() {
    struct icmp* icmp = (struct icmp*)sendbuf;
    icmp->icmp_type = ICMP_ECHO;
    icmp->icmp_code = 0;
    icmp->icmp_cksum = 0;

    // needn't use htons() call, because peer networking kernel didn't handle
    // this data and won't make different meanings(bigdian litteldian)
    icmp->icmp_id = getpid();
    icmp->icmp_seq = ++sendnum;   //needn't use hotns() call too.
    gettimeofday((struct timeval*)icmp->icmp_data, NULL);
    int len = 8+datalen;
    icmp->icmp_cksum = my_cksum((unsigned short*)icmp, len);
    int retval = sendto(sockfd, sendbuf, len, 0, (struct sockaddr*)&addr, sizeof(addr));
    if(retval == -1){
        perror("sendto()");
        exit(-1);
    }
}

void recv_icmp() {
    struct timeval *sendtime;
    struct timeval recvtime;

    for(;;) {
        int n = recvfrom(sockfd, recvbuf, sizeof(recvbuf), 0, 0, 0);
        if(n == -1) {
            if(errno == EINTR)continue;
            else {
                perror("recvfrom()");
                exit(-1);
            }
        } else {
            gettimeofday(&recvtime, NULL);
            struct ip *ip = (struct ip*)recvbuf;
            if(ip->ip_src.s_addr != addr.sin_addr.s_addr) {
                continue;
            }
            struct icmp *icmp = (struct icmp*)(recvbuf + ((ip->ip_hl)<<2));
            if(icmp->icmp_id != getpid()) {
                continue;
            }
            recvnum++;
            sendtime = (struct timeval*)icmp->icmp_data;
            tv_sub(&recvtime, sendtime);
            printf("imcp echo from %s(%dbytes)\tttl=%d\tseq=%d\ttime=%d.%06d s\n",
                   straddr, n, ip->ip_ttl, icmp->icmp_seq, recvtime.tv_sec, recvtime.tv_usec);
        }
    }
}

void catch_sigalrm(int signum) {
    send_icmp();
    alarm(1);
}

void catch_sigint(int signum) {
    printf("\nPing statics:send %d packets, recv %d packets, %d%% lost...\n",
           sendnum, recvnum, (int)((float)(sendnum-recvnum)/sendnum)*100);
    exit(0);
}

int main(int argc, char **argv) {
    if(argc != 2) {
        printf("please use format: ping hostname\n");
        exit(-1);
    }

    sockfd = socket(AF_INET, SOCK_RAW, IPPROTO_ICMP);
    if(sockfd == -1) {
        perror("socket()");
        return -1;
    }

    /*
    int sendbufsize = 180;
    socklen_t sendbufsizelen = sizeof(sendbufsize);
    if(setsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &sendbufsize, sendbufsizelen) == -1)
        perror("setsockopt()");
    int recvbufsize;
    socklen_t recvbufsizelen;
    if(getsockopt(sockfd, SOL_SOCKET, SO_RCVBUF, &recvbufsize, &recvbufsizelen) == -1)
        perror("getsockopt()");
    */

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    int retval = inet_pton(AF_INET, argv[1], &addr.sin_addr);
    if(retval == -1 || retval == 0) {
        struct hostent* host = gethostbyname(argv[1]);
        if(host == NULL) {
            fprintf(stderr, "gethostbyname(%s):%s\n", argv[1], strerror(errno));
            exit(-1);
        }

        /*
        if(host->h_name != NULL)printf("hostent.h_name:%s\n", host->h_name);
        if(host->h_aliases != NULL && *(host->h_aliases) != NULL)
            printf("hostent.h_aliases:%s\n", *(host->h_aliases));
        printf("hostent.h_addrtype:%d\n", host->h_addrtype);
        printf("hostent.h_length:%d\n", host->h_length);
        */

        if(host->h_addr_list != NULL && *(host->h_addr_list) != NULL) {
            strncpy((char*)&addr.sin_addr, *(host->h_addr_list), 4);
            inet_ntop(AF_INET, *(host->h_addr_list), straddr, sizeof(straddr));
        }
        printf("Ping address:%s(%s)\n\n", host->h_name, straddr);
    } else {
        strcpy(straddr, argv[1]);
        printf("Ping address:%s(%s)\n\n", straddr, straddr);
    }

    struct sigaction sa1;
    memset(&sa1, 0, sizeof(sa1));
    sa1.sa_handler = catch_sigalrm;
    sigemptyset(&sa1.sa_mask);
    sa1.sa_flags = 0;
    if(sigaction(SIGALRM, &sa1, NULL) == -1)
        perror("sigaction()");

    struct sigaction sa2;
    memset(&sa2, 0, sizeof(sa2));
    sa2.sa_handler = catch_sigint;
    sigemptyset(&sa2.sa_mask);
    sa2.sa_flags = 0;
    if(sigaction(SIGINT, &sa2, NULL) == -1)
        perror("sigaction()");

    alarm(1);
    recv_icmp();

    return 0;
}
```

# 参考
```c
# inet socket 与 packet socket
https://switch-router.gitee.io/blog/af_packet/

# Raw Socket 接收和发送数据包
https://github.com/xgfone/snippet/blob/master/snippet/docs/linux/program/raw-socket.md
```