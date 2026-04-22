```table-of-contents
```
# 背景
在主机上通过 nc 监听 56789 端口，然后在容器里使用 `nc` 发数据。
在主机上运行 nc UDP 服务器（`-u` 表示 UDP 协议，`-l` 表示监听的端口）
```
nc -ul 56789
```
然后启动一个容器，运行客户端：
```
$ docker run -it apline sh
/ # nc -u 172.16.13.13 56789
```


容器使用的是 docker 的默认网络，容器的 ip 是 172.17.0.3，通过 veth pair（图中没有显示）连接到虚拟网桥 docker0（ip 地址为 172.17.0.1），docker0所在的宿主机本身的网络为 eth0，其 ip 地址为 172.16.13.13。
```
 172.17.0.3
+----------+
|   eth0   |
+----+-----+
     |
     |
     |
     |
+----+-----+          +----------+
| docker0  |          |  eth0    |
+----------+          +----------+
172.17.0.1            172.16.13.13
```


- **问题**

在 docker0 上抓包，因为这是报文必经过的地方。通过过滤容器的 ip 地址，很容器找到感兴趣的报文：
```
tcpdump -i docker0 -nn host 172.17.0.3
```
抓包的结果如下：
可以发现第一个报文发送出去没有任何问题（因为 UDP 是没有 ACK 报文的，所以客户端无法知道对方有没有收到，这里说的没有问题是指没有对应的 ICMP 报文），但是第二个报文从服务端发送的报文，Client会返回一个 `ICMP` 告诉端口 38908 不可达；第三个报文从客户端发送的报文也是如此。以后的报文情况类似，双方再也无法进行通信了。
```
11:20:43.973286 IP 172.17.0.3.38908 > 172.16.13.13.56789: UDP, length 6
11:20:50.102018 IP 172.17.0.1.56789 > 172.17.0.3.38908: UDP, length 6
11:20:50.102129 IP 172.17.0.3 > 172.17.0.1: ICMP 172.17.0.3 udp port 38908 unreachable, length 42
11:20:54.503198 IP 172.17.0.3.38908 > 172.16.13.13.56789: UDP, length 3
11:20:54.503242 IP 172.16.13.13 > 172.17.0.3: ICMP 172.16.13.13 udp port 56789 unreachable, length 39
```


- **分析**
从网络报文的分析中可以看到服务端返回的报文源地址不是我们预想的 eth0 地址，而是 docker0 的地址，而客户端直接认为该报文是非法的，返回了 ICMP 的报文给对方。
那么问题的原因也可以分为两个部分：
1. 为什么应答报文源地址是**错误的**？
2. 既然 UDP 是无状态的，内核怎么判断源地址不正确呢？

# 接口多IP时SIP的选择机制
源地址和目的地址是 IP 首部中最重要的两个字段，而我们总是习惯于关注报文的目的地址，而忽略源地址。这完全是情有可原的！因为作为开发者来说，关心的是数据往哪发，作为运维者来说，配置路由规则也是按目的地址进行配置。至于源地址？就让内核自己去搞定好了，而内核似乎也真的能搞定。
源地址选择不重要吗？当然不是，源地址在绝大多数情况下就是对端的目的地址，它的选择是否正确决定了对端能不能将该响应正确地送回来。对于只有一个 IP 地址(排除 loopback )的主机来说，那没得说，只能选它；但如果有多个 IP 地址呢？ 要知道一台主机可以有多个网卡，每个网卡都可以配置多个 IP 地址，这时源地址该如何选择呢？
## SIP的选择顺序
内核按照以下顺序尝试选择：

1. socket 层次，用户使用 bind() 绑定的 IP 地址
> 注意：使用bind方式，会分配Sport，即在connect之前使用bind绑定，会分配Sport。Sport的选择范围受限于内核参数port-range。如果机器上存在大量的会话（比如time_wait）状态，那么下次bind分配sport可能失败，导致单机支持的会话个数受限于port-range。可以使用
2. 查询路由时，如果匹配的路由带有 src 关键字，则使用其指定的地址
3. 查询路由时，得到的出接口上配置的同网段的第一个 primary 地址
4. 查询路由时，其他接口上配置的 primary 地址

> 注：看上去源地址是完全在网络层完成的事，但实际上，不同的传输层( TCP 和 UDP )也会对源地址选择产生影响。
### TCP 源地址选择 
TCP 是面向连接的，这里的面向连接隐含一层意思：连接的四元组一旦建立，就不会变了。

对于发起端，它会在发送第一个 SYN 报文时(也就是用户使用 connect() 时)就进行源地址选择。
对于被动端则很简单，直接使用 SYN 报文中的目的地址。它需要重新源地址选择吗？不需要，或者说不能！原因是四元组唯一确定一条 TCP 连接，既然发起端已经在 SYN 中定下了<源IP，目的IP>，那么被动端只能默默接受，否则就无法算一条连接了。

-**内核相关实现**
主动端：
```
int tcp_v4_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    ......
    if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr; 
}
```

被动端：
```
// 收到 SYN 请求时
static void tcp_v4_init_req(struct request_sock *req,
			    const struct sock *sk_listener,
			    struct sk_buff *skb)
{
    ......
    sk_daddr_set(req_to_sk(req), ip_hdr(skb)->saddr);
}

// 收到 ACK 时
struct sock *tcp_v4_syn_recv_sock(const struct sock *sk, struct sk_buff *skb,
				  struct request_sock *req...
{
    ....
    newinet->inet_saddr	      = ireq->ir_loc_addr;
}                  
```

### UDP 源地址选择
UDP 在很多方面都没有 TCP 复杂，但源地址选择是个例外。UDP 没有数据流或者连接的概念，因此他不必听命于对端的报文的目的地址作为源地址，而是可以另起炉灶，重新选择。
比如在下面的拓扑中：
![](attachments/Pasted%20image%2020231120110358.png)

HOST 2 作为 UDP Server, 当 HOST 1 发送一个源地址 为 10.0.0.1 且目的地址为 192.168.2.1 的 UDP 报文后，HOST 2 查询路由发现回复的报文应该走 eth1，因此回复的报文源地址为 192.168.3.1。

**有什么办法可以避免 UDP 每个报文都去进行路由查找然后源地址选择呢？答案是 bind() 或者 connect()**。
```c
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>

int connect(int sockfd, const struct sockaddr *addr,
           socklen_t addrlen);
```
connect() 的相关代码如下：
```
int __ip4_datagram_connect(struct sock *sk, struct sockaddr *uaddr, int addr_len)
{
    ......
    if (!inet->inet_saddr)
		inet->inet_saddr = fl4->saddr;	/* Update source address */
}
```
# 相关问题
## UDP接口多IP的SIP的选择问题
UDP 和多网络接口。因为如果主机上只有一个网络接口，发出去的报文源地址一定不会有错；
通过搜索，发现这确实是个已知的问题。在 UNP（） 这本书中，已经描述过这个问题，下面是对应的内容：
![](attachments/Pasted%20image%2020231106115338.png)

> 小结：UDP 在多网卡的情况下，可能会发生服务器端源地址不对的情况，这是内核选路的结果。
## udp和tcp对比
为什么 UDP 和 TCP 有不同的选路逻辑呢？
因为 UDP 是无状态的协议，内核不会保存连接双方的信息，因此每次发送的报文都认为是独立的，socket 层每次发送报文默认情况不会指明要使用的源地址，只是说明对方地址。
因此，内核会为要发出去的报文选择一个 ip，这通常都是报文路由要经过的设备 ip 地址。

## 为什么 dnsmasq 服务没有这个问题呢
使用 `strace` 工具抓取了 dnsmasq的系统调用。
dnsmasq 在启动阶段监听了 UDP 和 TCP 的 54 端口
（因为是在本地机器上测试的，为了防止和本地 DNS 监听的 DNS端口冲突，我选择了 54 而不是标准的 53 端口）：
```c
socket(PF_INET, SOCK_DGRAM, IPPROTO_IP) = 4
setsockopt(4, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(4, {sa_family=AF_INET, sin_port=htons(54), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
setsockopt(4, SOL_IP, IP_PKTINFO, [1], 4) = 0

socket(PF_INET, SOCK_STREAM, IPPROTO_IP) = 5
setsockopt(5, SOL_SOCKET, SO_REUSEADDR, [1], 4) = 0
bind(5, {sa_family=AF_INET, sin_port=htons(54), sin_addr=inet_addr("0.0.0.0")}, 16) = 0
listen(5, 5)                            = 0
```

比起 TCP，UDP 部分少了 `listen`，但是多个 `setsockopt(4, SOL_IP, IP_PKTINFO, [1], 4)` 这句。
dnsmasq 收包和发包的系统调用，直接使用 `recvmsg` 和 `sendmsg` 系统调用：
```c
recvmsg(4, {msg_name(16)={sa_family=AF_INET, sin_port=htons(52072), sin_addr=inet_addr("10.111.59.4")}, msg_iov(1)=[{"\315\n\1 \0\1\0\0\0\0\0\1\fterminal19-0\5u5016\3"..., 4096}], msg_controllen=32, {cmsg_len=28, cmsg_level=SOL_IP, cmsg_type=, ...}, msg_flags=0}, 0) = 67

sendmsg(4, {msg_name(16)={sa_family=AF_INET, sin_port=htons(52072), sin_addr=inet_addr("10.111.59.4")}, msg_iov(1)=[{"\315\n\201\200\0\1\0\1\0\0\0\1\fterminal19-0\5u5016\3"..., 83}], msg_controllen=28, {cmsg_len=28, cmsg_level=SOL_IP, cmsg_type=, ...}, msg_flags=0}, 0) = 83
```

而出问题的应用 `strace` 结果如下：
```c
[pid   477] socket(PF_INET6, SOCK_DGRAM, IPPROTO_IP) = 124
[pid   477] setsockopt(124, SOL_IPV6, IPV6_V6ONLY, [0], 4) = 0
[pid   477] setsockopt(124, SOL_IPV6, IPV6_MULTICAST_HOPS, [1], 4) = 0
[pid   477] bind(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 0

[pid   477] getsockname(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 0
[pid   477] getsockname(124, {sa_family=AF_INET6, sin6_port=htons(6088), inet_pton(AF_INET6, "::", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 0

[pid   477] recvfrom(124, "j\201\2450\201\242\241\3\2\1\5\242\3\2\1\n\243\0160\f0\n\241\4\2\2\0\225\242\2\4\0"..., 2048, 0, {sa_family=AF_INET6, sin6_port=htons(38790), inet_pton(AF_INET6, "::ffff:172.17.0.3", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, [28]) = 168

[pid   477] sendto(124, "k\202\2\0210\202\2\r\240\3\2\1\5\241\3\2\1\v\243\5\33\3TDH\244\0220\20\240\3\2"..., 533, 0, {sa_family=AF_INET6, sin6_port=htons(38790), inet_pton(AF_INET6, "::ffff:172.17.0.3", &sin6_addr), sin6_flowinfo=0, sin6_scope_id=0}, 28) = 533
```
其对应的逻辑是这样的：使用 ipv6 绑定在 `0.0.0.0` 和 6088 端口，调用 `getsockname` 获取当前 socket 绑定的端口信息，数据传输过程使用的是 `recvfrom` 和 `sendto`。

对比下来，两者的不同有几点：

- 后者使用的是 ipv6，而前者是 ipv4
- 后者使用 `recvfrom` 和 `sendto` 传输数据，而前者是 `sendmsg` 和 `recvmsg`
- 前者有调用 `setsockopt` 设置 `IP_PKTINFO` 的值，而后者没有

## 为什么内核会把源地址和之前不同的报文丢弃
我们前面已经说过，UDP 协议是无连接的，默认情况下 socket 也不会保存双方连接的信息。即使服务端发送报文的源地址有误，只要对方能正常接收并处理，也不会导致网络不通。

因为 conntrack，内核的 netfilter 模块会保存连接的状态，并作为防火墙设置的依据。它保存的 UDP 连接，只是简单记录了主机上本地 ip 和端口，和对端 ip 和端口，并不会保存更多的内容。

在找到根源之前，我们曾经尝试过在服务器的主机上使用 SNAT 来修改服务端应答报文的源地址，期望能够修复该问题。但是却发现这种方法行不通，为什么呢？

因为 SNAT 是在 netfilter 最后做的，在之前 netfilter 的 conntrack 因为不认识该 connection，直接丢弃了，所以即使添加了 SNAT 也是无法工作的。

那能不能把 conntrack 功能去掉呢？比如解决方案：
```
iptables -I OUTPUT -t raw -p udp --sport 5060 -j CT --notrack
iptables -I PREROUTING -t raw -p udp --dport 5060 -j CT --notrack
```
答案也是否定的，因为 NAT 需要 conntrack 来做翻译工作，如果去掉 conntrack 等于 SNAT 完全没用。

# 解决方法
## UDP接口多IP使用 IP_PKTINFO 的sockopt
 获取 TCP socket 的目的地址很容易，通过 getsockname即可。但是对于无连接状态的 UDP socket 获取目的地址比较麻烦。
 
通过查找，发现 `IP_PKTINFO` 这个选项就是让内核在 socket 中保存 IP 报文的信息，当然也包括了报文的源地址和目的地址。
而 `man 7 ip` 文档中也说明了 `IP_PKTINFO` 是怎么控制源地址选择的：
![](attachments/Pasted%20image%2020231106112432.png)

也就是说，通过设置 `IP_PKTINFO` socket 选项为 1，然后使用 `recvmsg` 和 `sendmsg` 传输数据就能保证源地址选择符合我们的期望。这也是 dnsmasq 使用的方案。

### 限制
上面的方法固然好，但是很多语言目前稳定版本，socket 都不支持 sendmsg，recvmsg，更不要说 setsockopt 设置 IP_PKTINFO 了。在 java 中没有，在 python 2.7 版本也没有，只有 python 3.3 版本支持。所以这种方法还不具有普适性，但是如果用的是 C 或者 Go 语言，实现起来倒是很方便的。

## UDP监听特定IP+PORT
不再是监听在 `0.0.0.0` 地址（也就是所有的 ip 地址）+ Port，而是Udp server上监听特定的IP+Port，那么也可以保证server 在udp 回包的时候SIP的选择也是正确的。

比如之前接口多IP时，监听的是：
```bash
nc -ul 56789
```

那么使用 nc 启动一个 udp 服务器，监听在 特定IP上，则为：
```bash
nc -ul 172.16.13.13 56789
```
这种情况下，服务端和客户端也能正常通信。
# 其他
## sendto && recvfrom 
- **函数原型**
```text
#include <sys/socket.h>

ssize_t recvfrom(int sockfd, void *buf, size_t nbytes,
					int flags, struct sockaddr *from, socklen_t *addrlen);
					
ssize_t sendto(int sockfd, const void *buf, size_t nsize, 
					int flags, const struct sockaddr *to, const socklen_t *addrlen);
					
若成功，均返回读或者写的字节数；失败则返回-1 
```

- **范例**
UDP Server：
```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#define UDP_TEST_PORT       50001

int main(int argC, char* arg[])
{
    struct sockaddr_in addr;
    int sockfd, len = 0;    
    int addr_len = sizeof(struct sockaddr_in);
    char buffer[256];   

    /* 建立socket，注意必须是SOCK_DGRAM */
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror ("socket");
        exit(1);
    }

    /* 填写sockaddr_in 结构 */
    bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(UDP_TEST_PORT);
    addr.sin_addr.s_addr = htonl(INADDR_ANY) ;// 接收任意IP发来的数据

    /* 绑定socket */
    if (bind(sockfd, (struct sockaddr *)&addr, sizeof(addr))<0) {
        perror("connect");
        exit(1);
    }

    while(1) {
        bzero(buffer, sizeof(buffer));
        len = recvfrom(sockfd, buffer, sizeof(buffer), 0, 
                       (struct sockaddr *)&addr ,&addr_len);
        /* 显示client端的网络地址和收到的字符串消息 */
        printf("Received a string from client %s, string is: %s\n", 
                inet_ntoa(addr.sin_addr), buffer);
        /* 将收到的字符串消息返回给client端 */
        sendto(sockfd,buffer, len, 0, (struct sockaddr *)&addr, addr_len);
    }

    return 0;
}
  
// End of udp_server.c
```

UDP Client：
```c
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <stdlib.h>
#include <string.h>
#include <stdio.h>

#define UDP_TEST_PORT       50001
#define UDP_SERVER_IP       "127.0.0.1"

int main(int argC, char* arg[])
{
    struct sockaddr_in addr;
    int sockfd, len = 0;    
    int addr_len = sizeof(struct sockaddr_in);      
    char buffer[256];

    /* 建立socket，注意必须是SOCK_DGRAM */
    if ((sockfd = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("socket");
        exit(1);
    }

    /* 填写sockaddr_in*/
    bzero(&addr, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(UDP_TEST_PORT);
    addr.sin_addr.s_addr = inet_addr(UDP_SERVER_IP);

    while(1) {
        bzero(buffer, sizeof(buffer));

        printf("Please enter a string to send to server: \n");

        /* 从标准输入设备取得字符串*/
        len = read(STDIN_FILENO, buffer, sizeof(buffer));

        /* 将字符串传送给server端*/
        sendto(sockfd, buffer, len, 0, (struct sockaddr *)&addr, addr_len);

        /* 接收server端返回的字符串*/
        len = recvfrom(sockfd, buffer, sizeof(buffer), 0, 
                       (struct sockaddr *)&addr, &addr_len);
        printf("Receive from server: %s\n", buffer);
    }

    return 0;
}

// End of udp_client.c
```
# 范例
## PKTINFO范例
以下实例，以dpvs中的 udp_serv.c 作为参考。

- 相关数据结构
```c

struct in_pktinfo {
  int   ipi_ifindex;
  struct in_addr  ipi_spec_dst;
  struct in_addr  ipi_addr;
};

struct in6_pktinfo {
  struct in6_addr ipi6_addr;
  int   ipi6_ifindex;
};

struct iovec {
    void  *iov_base;              /* for send buffer or receive buff */
    size_t iov_len;               /* Number of bytes to transfer */
};
 
struct msghdr {
    void         *msg_name;       /* optional address: for srcaddr of recvmsg or dstaddr of sendmsg */
    socklen_t     msg_namelen;    /* size of address */
    struct iovec *msg_iov;        /* scatter/gather array */
    size_t        msg_iovlen;     /* elements in msg_iov */
    void         *msg_control;    /* ancillary data, see below */
    size_t        msg_controllen; /* ancillary data buffer len */
    int           msg_flags;      /* flags on received message */
};
 
struct cmsghdr {
    socklen_t     cmsg_len;     /* data byte count, including hdr */
    int           cmsg_level;   /* originating protocol */
    int           cmsg_type;    /* protocol-specific type */
    /* followed by unsigned char cmsg_data[]; */
};


```

- udp server 范例
```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <sys/epoll.h>
#include <linux/ipv6.h>
#include <netinet/in.h>
/* for __u8, __be16, __be32, __u64 only */
#include "common.h"

/* for union inet_addr only */
#include "uoa_extra.h"
#include "uoa.h"

#define MAX_SUPP_AF         2
#define MAX_EPOLL_EVENTS    2
#define SA                  struct sockaddr

static __u16 SERV_PORT = 6000;

#if !__UAPI_DEF_IN6_PKTINFO
struct in6_pktinfo {
  struct in6_addr ipi6_addr;
  int   ipi6_ifindex;
};
#endif

void handle_reply(int efd, int fd)
{
    struct sockaddr_storage peer;
    struct sockaddr_in *sin = NULL;
    struct sockaddr_in6 *sin6 = NULL;
    char buff[4096], from[64], to[64];
    char msg_control[1024], control_buf[1024];;
    struct uoa_param_map map;
    socklen_t len, mlen;
    int n;
    uint8_t af = 0;
    struct iovec iovec[1];
    union {
        struct sockaddr_in6 peer_addr6;
        struct sockaddr_in peer_addr4;
    } peer_addr;

    memset(&peer_addr, 0, sizeof(peer_addr));
    iovec[0].iov_base = buff;
    iovec[0].iov_len = sizeof(buff);
    struct msghdr mh = {
        .msg_name = &peer_addr, // for the peer addr info
        .msg_namelen = sizeof(peer_addr),
        .msg_control = msg_control, // for the control info(eg. pkt info)
        .msg_controllen = sizeof(msg_control),
        .msg_iov = iovec,   //for the received data
        .msg_iovlen = sizeof(iovec) / sizeof(*iovec),
        .msg_flags  = 0,
    };
    struct cmsghdr *cmsg;
    struct in_pktinfo *pi;
    struct in6_pktinfo *p6i;
    int ifindex;
    struct in_addr dst_addr;
    struct in6_addr dst_addr6;
    int cmsg_space;
    int retvalue = 0;
    retvalue = recvmsg(fd, &mh, 0);
    if (retvalue < 0) {
        printf("recvmsg error.\n");
        return;
    } else if (retvalue < sizeof(buff)) {
        buff[retvalue] = 0;
    }
    for (cmsg = CMSG_FIRSTHDR(&mh); cmsg != NULL; cmsg = CMSG_NXTHDR(&mh, cmsg)) {
        if ((cmsg->cmsg_level == IPPROTO_IP) && (cmsg->cmsg_type == IP_PKTINFO)) {
            pi = (struct in_pktinfo *)CMSG_DATA(cmsg);
            af = AF_INET;
            dst_addr = pi->ipi_spec_dst;
            ifindex = pi->ipi_ifindex;
        } else if ((cmsg->cmsg_level == IPPROTO_IPV6) && (cmsg->cmsg_type == IPV6_PKTINFO)) {
            p6i = (struct in6_pktinfo *)CMSG_DATA(cmsg);
            af = AF_INET6;
            dst_addr6 = p6i->ipi6_addr;
            ifindex = p6i->ipi6_ifindex;
        }
    }
    if (AF_INET == af) {
        sin = (struct sockaddr_in *)&peer_addr.peer_addr4;
        inet_ntop(AF_INET, &sin->sin_addr.s_addr, from, sizeof(from));
        inet_ntop(AF_INET, &dst_addr, to, sizeof(to));
        printf("if:%d Receive %d bytes from %s:%d to %s:%d -- info:%s\n",
                ifindex, retvalue, from, ntohs(sin->sin_port), to, SERV_PORT, buff);
        /*
         * get real client address from uoa.
         *
         * note: src/dst is for original pkt, so peer is
         * "orginal" source, instead of local. wildcard
         * lookup for daddr (or local IP) is supported.
         * */
        memset(&map, 0, sizeof(map));
        map.af    = af;
        map.sport = sin->sin_port;
        map.dport = htons(SERV_PORT);
        memmove(&map.saddr, &sin->sin_addr.s_addr, sizeof(struct in_addr));
        mlen = sizeof(map);
        if (getsockopt(fd, IPPROTO_IP, UOA_SO_GET_LOOKUP, &map, &mlen) == 0) {
            inet_ntop(map.real_af, &map.real_saddr.in, from, sizeof(from));
            printf("  real client %s:%d\n", from, ntohs(map.real_sport));
        }

        // send msg
        memset(&mh, 0, sizeof(mh));
        iovec[0].iov_len = buff;
        iovec[0].iov_len = retvalue;
        mh.msg_iov = iovec;
        mh.msg_iovlen = sizeof(iovec) / sizeof(*iovec);
        mh.msg_name = &peer_addr.peer_addr4;
        mh.msg_namelen = sizeof(peer_addr.peer_addr4);
        // add srcip control msg
        mh.msg_control = control_buf;
        mh.msg_flags = 0;
        mh.msg_controllen = CMSG_SPACE(sizeof(struct in_pktinfo));
        cmsg = CMSG_FIRSTHDR(&mh);
        cmsg->cmsg_level = IPPROTO_IP;
        cmsg->cmsg_type = IP_PKTINFO;
        cmsg->cmsg_len = CMSG_LEN(sizeof(struct in_pktinfo));
        pi = CMSG_DATA(cmsg);
        memset(pi, 0, sizeof(*pi));
        pi->ipi_spec_dst = dst_addr;
        pi->ipi_ifindex = ifindex;
        mh.msg_controllen = cmsg->cmsg_len;
        retvalue = sendmsg(fd, &mh, 0);
    } else if (AF_INET6 == af) {
        sin6 = (struct sockaddr_in6 *)&peer_addr.peer_addr6;
        inet_ntop(AF_INET6, &sin6->sin6_addr, from, sizeof(from));
        inet_ntop(AF_INET6, &dst_addr6, to, sizeof(to));
        printf("if:%d Receive %d bytes from %s:%d to %s:%d -- info:%s\n",
                ifindex, retvalue, from, ntohs(sin6->sin6_port), to, SERV_PORT, buff);
        // get real client ipv6 address from uoa.
        memset(&map, 0, sizeof(map));
        map.af    = af;
        map.sport = sin6->sin6_port;
        map.dport = htons(SERV_PORT);
        memmove(&map.saddr, &sin6->sin6_addr, sizeof(struct in6_addr));
        mlen = sizeof(map);
        if (getsockopt(fd, IPPROTO_IP, UOA_SO_GET_LOOKUP, &map, &mlen) == 0) {
            inet_ntop(map.real_af, &map.real_saddr.in6, from, sizeof(from));
            printf("  real client %s:%d\n", from, ntohs(map.real_sport));
        }

        // send msg
        memset(&mh, 0, sizeof(mh));
        iovec[0].iov_len = buff;
        iovec[0].iov_len = retvalue;
        mh.msg_iov = iovec;
        mh.msg_iovlen = sizeof(iovec) / sizeof(*iovec);
        mh.msg_name = &peer_addr.peer_addr6;
        mh.msg_namelen = sizeof(peer_addr.peer_addr6);
        // add srcip control msg
        mh.msg_control = control_buf;
        mh.msg_flags = 0;
        mh.msg_controllen = CMSG_SPACE(sizeof(struct in6_pktinfo));
        cmsg = CMSG_FIRSTHDR(&mh);
        cmsg->cmsg_level = IPPROTO_IPV6;
        cmsg->cmsg_type = IPV6_PKTINFO;
        cmsg->cmsg_len = CMSG_LEN(sizeof(struct in6_pktinfo));
        p6i = CMSG_DATA(cmsg);
        memset(p6i, 0, sizeof(*p6i));
        p6i->ipi6_addr = dst_addr6;
        p6i->ipi6_ifindex = ifindex;
        mh.msg_controllen = cmsg->cmsg_len;
        retvalue = sendmsg(fd, &mh, 0);
    }
    fflush(stdout);
}

int main(int argc, char *argv[])
{
    int i, sockfd[MAX_SUPP_AF];
    int epfd, nfds;
    int enable = 1;
    struct epoll_event events[MAX_EPOLL_EVENTS];
    struct sockaddr_in local;
    struct sockaddr_in6 local6;

    if (argc > 1)
        SERV_PORT = atoi(argv[1]);
    printf("start udp echo server on 0.0.0.0:%u\n", SERV_PORT);

    if ((sockfd[0] = socket(AF_INET, SOCK_DGRAM, 0)) < 0) {
        perror("Fail to create INET socket!\n");
        exit(1);
    }
    setsockopt(sockfd[0], IPPROTO_IP, IP_PKTINFO, &enable, sizeof(enable));

    if ((sockfd[1] = socket(AF_INET6, SOCK_DGRAM, 0)) < 0) {
        perror("Fail to create INET6 socket!");
        exit(1);
    }
    setsockopt(sockfd[1], IPPROTO_IPV6, IPV6_RECVPKTINFO, &enable, sizeof(enable));

    if ((epfd = epoll_create1(0)) < 0) {
        perror("Fail to create epoll fd!\n");
        exit(1);
    }

    for (i = 0; i < MAX_SUPP_AF; i++) {
        setsockopt(sockfd[i], SOL_SOCKET, SO_REUSEADDR, &enable, sizeof(enable));
        setsockopt(sockfd[i], SOL_SOCKET, SO_REUSEPORT, &enable, sizeof(enable));
    }

    memset(&local, 0, sizeof(struct sockaddr_in));
    local.sin_family = AF_INET;
    local.sin_port = htons(SERV_PORT);
    local.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(sockfd[0], (struct sockaddr *)&local, sizeof(local)) != 0) {
        perror("Fail to bind INET socket!\n");
        exit(1);
    }

    memset(&local6, 0, sizeof(struct sockaddr_in6));
    local6.sin6_family = AF_INET6;
    local6.sin6_port = htons(SERV_PORT);
    local6.sin6_addr = in6addr_any;

    if (bind(sockfd[1], (struct sockaddr *)&local6, sizeof(local6)) != 0) {
        perror("Fail to bind INET6 socket!\n");
        exit(1);
    }

    for (i = 0; i < MAX_SUPP_AF; i++) {
        struct epoll_event ev;
        memset(&ev, 0, sizeof(ev));
        ev.events = EPOLLIN | EPOLLERR;
        ev.data.fd = sockfd[i];
        if (epoll_ctl(epfd, EPOLL_CTL_ADD, sockfd[i], &ev) != 0) {
            fprintf(stderr, "epoll_ctl add failed for sockfd[%d]\n", i);
            exit(1);
        }
    }

    while (1) {
        nfds = epoll_wait(epfd, events, 2, -1);
        if (nfds == -1) {
            perror("epoll_wait failed\n");
            exit(1);
        }

        for (i = 0; i < nfds; i++) {
            handle_reply(epfd, events[i].data.fd);
        }
    }

    for (i = 0; i < MAX_SUPP_AF; i++)
        close(sockfd[i]);

    exit(0);
}

```
# 参考
```c
https://cizixs.com/2017/08/21/docker-udp-issue/

# Linux 报文源地址选择那点事儿
https://switch-router.gitee.io/blog/srcselect/
【其他文章也很好】
```