```table-of-contents
```
# SYN Flood 攻击
`TCP`连接建立时，客户端通过发送`SYN`报文发起向处于监听状态的服务器发起连接，服务器为该连接**分配一定的资源，并发送`SYN+ACK`报文**。
![](attachments/Pasted%20image%2020240401104513.png)
对服务器来说，此时该连接的状态称为`半连接`(`Half-Open`)，而当其之后收到客户端回复的`ACK`报文后，连接才算建立完成。在这个过程中，如果服务器一直没有收到`ACK`报文(比如在链路中丢失了)，服务器会在超时后重传`SYN+ACK`。
![](attachments/Pasted%20image%2020231127162500.png)
如果经过多次超时重传后，还没有收到, 那么服务器会回收资源并关闭`半连接`，仿佛之前最初的`SYN`报文从来没到过一样！
![](attachments/Pasted%20image%2020231127162512.png)

这看上一切正常，但是如果有坏人**故意**大量不断发送伪造的`SYN`报文，那么服务器就会分配大量注定无用的资源，并且服务器能保存的半连接的数量是有限的！
所以当服务器受到大量攻击报文时，它就**不能再接收正常的连接**了。换句话说，它的服务不再可用了！这就是`SYN Flood`攻击的原理，它是一种典型的`DDoS`攻击。

# Syn-cookie
## 思路
`Syn-Flood`攻击成立的**关键在于服务器资源是有限的，而服务器收到请求会分配资源**。
通常来说，没有syn-cookie时，服务器用这些资源**保存此次请求的关键信息，包括请求的来源和目(五元组)，以及`TCP`选项，如最大报文段长度`MSS`、时间戳`timestamp`、选择应答使能`Sack`、窗口缩放因子`Wscale`等等**。当后续的`ACK`报文到达，三次握手完成，新的连接创建，这些信息可以会被复制到连接结构中，用来指导后续的报文收发。

那么现在的问题就是服务器如何在**不分配**资源的情况下：
1. **验证之后可能到达的`ACK`的有效性**，保证这是一次完整的握手
2. **如何从三次握手的ack中提取`SYN`报文中携带的`TCP`选项信息**；


## SYN cookies 算法
### 小结
`SYN Cookies`[算法](https://en.wikipedia.org/wiki/SYN_cookies)可以：
**解决上面的第`1`个问题以及第`2`个问题的一部分。**

`SYN Cookie`技术可以让服务器在收到客户端的`SYN`报文时，**不分配资源保存客户端信息(五元组信息，client的seq，client的syn的option信息)**，而是将这些信息**保存在`SYN+ACK`的初始seq序号和时间戳中。对正常的连接，这些信息会随着`ACK`报文被带回来**。

即：SYN Cookies本质是服务端根据连接信息「比如：hash(五元组+时间戳的高位)和MSS」，按照一定格式编码出一个初始seq序号，在后续收到客户端ack时，根据这个序号能反推出连接信息（比如：mss）。


### 生成syn-ack的seq序列号
我们知道，`TCP`连接建立时，双方的起始报文序号是可以任意的。`SYN cookies`利用这一点，按照以下规则构造初始序列号：
- 设`t`为一个缓慢增长的时间戳(典型实现是每`64s`递增一次)
- 设`m`为客户端发送的`SYN`报文中的`MSS`选项值
- 设`s`是连接的元组信息(源IP,目的IP,源端口，目的端口)和`t`经过密码学运算后的`Hash`值，即`s = hash(sip,dip,sport,dport,t)`，`s`的结果取低 24 位。

则初始序列号`n`为：
- 高 **5** 位为`t mod 32`
- 接下来**3**位为`m`的编码值
- 低 **24** 位为`s`

```bash
+----------+--------+-------------------+
|  6 bits  | 2 bits |     24 bits       |
| t mod 32 |  MSS   | hash(ip, port, t) |
+----------+--------+-------------------+
```
即：回复的 `syn-ack` 的 `seq num`中保存有 `syn`包中的  `mss`，五元组的hash值。
那么收到 `ack`包的 `ack num` 中就可以解析出 `syn`包中的  `mss` 等。

#### 时间戳参与hash的作用
注：时间戳，可以**防止重放攻击**。即使重放，但是时间不对，也不可以通过 `syn-cookie`的校验。 

时间戳，有2个作用：
1》时间戳在此中可以认为是随机值；
2》防重放：时间戳一般会取前N位，即只要是在一定时间范围内得到三次握手的ACK，进行hash的时间戳都是相同的，就可以将三次握手的ACK验证通过。
hash计算和验证的时候，从系统中取时间戳，

### 验证三次握手ack的ack num
当客户端收到此`SYN+ACK`报文后，根据`TCP`标准，它会回复`ACK`报文，且报文中`ack = n + 1`，那么在服务器收到它时，**将`ack - 1`就可以拿回当初发送的`SYN+ACK`报文中的seq序号了(即： cookie)**！服务器巧妙地通过这种方式间接保存了一部分`SYN`报文的信息。

接下来，服务器需要对`ack - 1`这个序号进行检查：
- 将高 **5** 位表示的`t`与当前之间比较，看其到达地时间是否能接受。
- 根据`t`和连接元组重新计算`s`，看是否和低 **24** 一致，若不一致，说明这个报文是被伪造的。
- 解码序号中隐藏的`mss`信息。

到此，连接就可以顺利建立了。

### Linux的SYN Cookies中cookie的生成以及验证

#### cookies的生成
Linux的SYN Cookies编码方式和RFC 4987中所描述的不同。在Linux中，编码32位Cookie的方式如下：

```bash
Hash(五元组) + (时间戳 << 24) + 客户端序号 + ((Hash(五元组，时间戳) + MSS序号) & 0x00FFFFFF)

如上：
	基于五元组信息的hash + SYN包的seq + SYN中MSS + MSS序号 + 时间戳左移<<N;

时间戳左移<<N: 主要在指定的时间范围内收到ACK，检查的`时间戳左移<<N` 和 生成cookie时的 `时间戳左移<<N`的值应该是相同的。

```

#### cookie的验证以及获取mss

在协议栈收到客户端ACK包时，会解码Cookie,判断是否是一个正确合法的连接。其中，==MSS序号的解码方式==为：

```bash:
Cookie - Hash(五元组) - (时间戳 << 24) - 客户端序号 - （Hash(五元组，时间戳) & 0x00FFFFFF）

Cookie = 三次握手的ACK包的ack_num -1 = Syn-ACK包的 seq_num
客户端序号 = Syn包的seq_num = 三次握手的ACK包的seq_num -1
```

其实就是做减法(还有验证时间戳的逻辑，为了简化就不写了)。这样得到的MSS序号实际是个MSS Entry Index。

Linux内核中有个MSS表，如下：

```c
static __u16 const msstab[] = {
    536,
    1300,
    1440,    /* 1440, 1452: PPPoE */
    1460,
};
```
可以看到里面有4个Entry。解码后，协议栈会判断MSS Entry Index的合法性。如果得到的MSS Entry Index 不在0、1、2、3当中，就认为是非法值；否则认为是合法值，根据Entry查表拿到MSS值。另外，在现在Linux内核中，MSS Entry最常见的值是3。

#### 潜在问题
接下来考虑一个问题：假如解码Cookie之前发生丢包，会出现什么后果？

例如：客户端发送的首个包有3个字节，并且该包被丢失，服务端处理的cookie实际是第二个包。根据上面计算MSS的流程，这种情况只会影响其中的`客户端序号`「客户端发送2个包，第一个包的seq=N， 第二个包的seq=N+3」，导致计算出的MSS Entry Index比正常值小3。由前面所说，MSS Entry Index正常值大概率就是3,因此这里计算出的MSS Entry很可能就是`3-3=0`；

`0`，依然是个合法的MSS Entry Index,也就是说此时协议栈并不知道他接收的实际上是第二个包，协议栈以为这就是个客户端的首个ACK包！当然如果首包长度是4,那么服务端就能根据MSS Entry发现问题了。
> 注：即==建立连接之后，Client连续发送多个包，首个包的长度<=3的时候，首包丢失，则依然可以cookie校验通过，会存在问题==。

##### 什么时候第一个包可能会丢失
第一个数据包长度<=3, 有可能第一个数据包在三次握手的第三个ACK中，即第三个ACK中携带数据作为第一个数据包。
在全连接队列满的时候 ??


## SYN Cookies 缺点
既然`SYN Cookies`可以减小资源分配环节，那为什么没有被纳入`TCP`标准呢？
原因是`SYN Cookies`也是有代价的：

### 缺点一：`MSS`的编码只有**3**位
`MSS`的编码只有**3**位，因此最多只能使用 **8** 种`MSS`值

### 缺点二：丢失了其他的tcp option，比如`Wscale`，`SACK`选项
服务器必须拒绝客户端`SYN`报文中的其他只在`SYN`和`SYN+ACK`中协商的选项，原因是服务器没有地方可以保存这些选项，比如`Wscale`，`SACK`选项等。

### 缺点三：消耗CPU资源
3. 增加了密码学运算，消耗了CPU。「以时间换空间」

### 缺点四：首包(<=3B)丢失后后续ack依然可以验证通过
建立连接之后，Client连续发送多个包，首个包的长度<=3的时候，首包丢失，则依然可以cookie校验通过。

# Linux中的Syn-cookie
`Linux`上的`SYN Cookies`实现与`wiki`中描述的算法在序号生成上有一些区别，其`SYN+ACK`的序号通过下面的公式进行计算：
> 内核编译需要打开 **CONFIG_SYN_COOKIES**

```c
seq = hash(saddr, daddr, sport, dport, 0, 0) + req.th.seq + t << 24 + (hash(saddr, daddr, sport, dport, t, 1) + mss_ind) & 0x00FFFFFF

mss_ind: mss index;
```
其中：
`req.th.seq`表示客户端的`SYN`报文中的序号；
`mss_ind`是客户端通告的`MSS`值得编码，它的取值在比较新的内核中有 **4** 种(老的内核有 **8** 种), 分别对应以下 **4** 种值：
```c
static __u16 const msstab[] = {
	536,
	1300,
	1440,	/* 1440, 1452: PPPoE */
	1460,
};
```
## 内核收到syn的流程
```
tcp_v4_rcv
  |
  |- __inet_lookup_skb
  |- tcp_v4_do_rcv
      |
      |- tcp_rcv_state_process
          |
          |- tcp_conn_request
             |
             |- cookie_init_sequence
                |
                |- cookie_v4_init_sequence
                   |
                   |- __cookie_v4_init_sequence
                      |
                      |-- secure_tcp_syn_cookie
```
## 内核收到三次握手ACK的流程
```
tcp_v4_rcv
 |
 |- __inet_lookup_skb
 |- tcp_v4_do_rcv
   |
   |- tcp_v4_cookie_check  //  SYN_RCV
      |
      |- cookie_v4_check
   |- tcp_child_process   
      |
      |- tcp_rcv_state_process
         |
         |- tcp_ack
```


## SYN Cookie与TCP timestamps：syn-ack中的timestamp携带sack以及Wscale选项
承接前面所述SYN Cookie的缺点：
```bash
Fortunately Linux has a work around. If TCP Timestamps are enabled, the kernel can reuse another slot of 32 bits in the Timestamp field. It contains:
```

```bash
+-----------+-------+-------+--------+
|  26 bits  | 1 bit | 1 bit | 4 bits |
| Timestamp |  ECN  | SACK  | WScale |
+-----------+-------+-------+--------+
```
如果服务器和客户端都打开了时间戳选项（Linux默认打开），那么服务器可以将客户端在SYN报文中携带了TCP选项的部分使能情况暂时保存在时间戳中。当前使用了低 6 位，分别保存Wscale、SACK和ECN。
![](attachments/Pasted%20image%2020231127163702.png)

客户端会在ACK的TSecr字段，把这些值带回来。

虽然`net.ipv4.tcp_timestamps`默认是打开的，它在SYN Cookie启用的时候，可以带来一些好处，但它也会**给每个报文增加12byte的长度**，non-trivial amount of bandwidth。当然，tcp_timestamps的作用一开始并不是为了SYN Cookie，它还有别的重要功能。

## 小结
![](attachments/Pasted%20image%2020240401105745.png)

因此：Linux文档中说明，SYN Cookie机制只是用来应对攻击，如果没有攻击，只是服务器负担过重，不建议使用这个功能。因为这个功能**不是TCP标准**，通过Cookie建立的TCP连接，不支持TCP扩展功能。**但是，tcp_syncookies默认开启，设置为1，在SYN队列被塞满后开始工作。**

# QA
## 是否每个ack包都进行cookie的校验
Q：在收到ACK报文的时候会计算cookie是否合法，那么是不是任何一个ACK报文都计算cookie值呢？
> 应该是在收到三次握手的最后一个ACK报文时才进行计算。即：==收到`ack`包，但是找不到会话的情况下，才进行`syn cookie`的检查==。


## cookie校验的问题
正常情况下，是对三次握手的ACK包进行`cookie`的校验。
但是实际上，`cookie`校验的时候，是无法区分这个ACK是否三次握手的ack，还是别人构造的一个ACK，也不知道是不是四次挥手(或者三次挥手)的最后一个ACK。

只能依赖于，cookie校验之前，是否可以查询到会话。

## dpvs中的 `reuse conn的cookie校验通过问题

# 其他
## hping3发起syn flood攻击
```c
hping3 -c 1000 -d 120 -S -p 80 --flood --rand-source 192.168.100.1

-c 指定连接数 -p 目标端口  
-d 指定数据部分的大小 -S 攻击类型是Syn flood  
–flood 以泛洪的方式攻击 –rand-source 随机产生伪造源地址
```

![](attachments/Pasted%20image%2020231127163836.png)
![](attachments/Pasted%20image%2020231127163859.png)

## SYN-Cookie设置
Linux中的`/proc/sys/net/ipv4/tcp_syncookies`是内核中的`SYN Cookies`开关。其值如下所示：
- 值为 0 :始终不生效
- 值为 1 :一般情况下不生效，但当 listen sock 的 accept 队列满时(应用程序没有及时使用 accpet()) 生效。
- 值为 2 :始终生效

# 范例
本实验是在`4.4.0`内核运行的，服务端监听`50001`端口，`backlog`参数为`3`。同时，模拟不同的客户端注入`SYN`报文。
- **不开启 SYN Cookies**
```
echo 0 > /proc/sys/net/ipv4/tcp_syncookies
```
可以看到，在收到`3`个`SYN`报文后，服务器不再响应新的连接请求了，这也就是`SYN-Flood`的攻击方式。
![](attachments/Pasted%20image%2020231127192251.png)

- **有条件使用 SYN Cookies**
```
echo 1 > /proc/sys/net/ipv4/tcp_syncookies
```
![](attachments/Pasted%20image%2020231127192314.png)
由于服务器的`backlog`参数为`3`，因此图中的从第`4`个`SYN+ACK`(`#8`报文)开始使用`SYN Cookies`。从时间戳可以看出，`#8`报文比 `#6`号报文还要小。

```c
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netinet/ip.h>
#include <netinet/tcp.h>
#include <net/if.h>
#include <sys/ioctl.h>
#include <linux/if_tun.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <pthread.h>

#define PCKT_LEN 1024
#define BACKLOG 3
#define TUN_ADDR "192.168.2.1"
#define SPOOF_NET "192.168.3.0"
#define SPOOF_PREFIX "192.168.3."

#define COUNT 8

const char* spoof_ip_list[COUNT] = {"192.168.3.1",
         "192.168.3.2",
         "192.168.3.3",
         "192.168.3.4",
         "192.168.3.5",
         "192.168.3.6",
         "192.168.3.7",
         "192.168.3.8"}; 

const uint16_t spoof_mss[COUNT] = {536, 1300, 1440, 1460, 536, 1300, 1440, 1460};

const uint32_t spoof_tsp[COUNT] = {1000, 2000, 3000, 4000, 5000,6000, 7000, 8000};

const uint8_t spoof_wscale[COUNT] = {1, 2, 3, 4, 1, 2, 3, 4};

#define TUN_PORT 50001

struct psdhdr{
 uint32_t saddr;
 uint32_t daddr;
 char zero;
 char protocol;
 uint16_t tcplen;
};

struct mss_opt{
 uint8_t kind; // = 2
 uint8_t length; // = 4
 uint16_t mss;  
}__attribute__((packed));

struct tstamp_opt{
 uint8_t kind; // = 8 
 uint8_t length; // = 10
 uint32_t tsval;  
 uint32_t tsecr;
 uint8_t nop[2];
}__attribute__((packed));

struct wscale_opt{
 uint8_t kind; // = 3 
 uint8_t length; // = 3
 uint8_t scale;  
 uint8_t nop;
}__attribute__((packed));

uint16_t calc_cksm(void *pkt, int len)
{
    uint16_t *buf = (uint16_t*)pkt;
    uint32_t cksm = 0;
    while(len > 1)
    {
        cksm += *buf++;
        cksm = (cksm >> 16) + (cksm & 0xffff);
        len -= 2;
    }
    if(len)
    {
        cksm += *((uint8_t*)buf);
        cksm = (cksm >> 16) + (cksm & 0xffff);
    }
    return (uint16_t)((~cksm) & 0xffff);
} 


unsigned short tcp_checksum (struct iphdr *ip, struct tcphdr* th, char* opt, int optlen)
{
	uint16_t sum = 0;
	char buf[PCKT_LEN];
	int chksumlen = 0;
	struct psdhdr psdhdr;

	memset(buf, 0, PCKT_LEN);

	psdhdr.saddr = ip->saddr;
	psdhdr.daddr = ip->daddr;
	psdhdr.zero = 0;
	psdhdr.protocol = ip->protocol;
	psdhdr.tcplen = htons(sizeof(struct tcphdr) + optlen);

	memcpy(&buf[0], &psdhdr, sizeof(struct psdhdr));

	chksumlen += sizeof(struct psdhdr);

	memcpy(&buf[chksumlen], th, sizeof(struct tcphdr));

	chksumlen += sizeof(struct tcphdr);

	if (optlen > 0)
	{
		memcpy(&buf[chksumlen], opt, optlen);
		chksumlen += optlen;
	}

	sum = calc_cksm(buf, chksumlen);

	return sum; 
}


int tun_create(int flags)
{
    int fd, err;
    struct ifreq ifr;

    if ((fd = open("/dev/net/tun", O_RDWR)) < 0){
        return fd;
    }

    memset(&ifr, 0, sizeof(ifr));
    ifr.ifr_flags = flags;

    if ((err = ioctl(fd, TUNSETIFF, (void*)&ifr)) < 0 )
    {
        close(fd);
        return err;
    }

    if (strcmp(ifr.ifr_name, "tun0")) {
        close(fd);
        return -1;
    }

    return fd;
} 

int tun_setup(char* tundev)
{
    struct ifreq ifr;
    int sockfd;
    int err;
    
    memset(&ifr, 0, sizeof(ifr));
    snprintf(ifr.ifr_name, (sizeof(ifr.ifr_name) - 1), "%s", tundev);
    
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);
    if (sockfd < 0)
    {
        return err;
    }
        
    if((err = ioctl(sockfd, SIOCGIFFLAGS, (void *)&ifr)) < 0 ) 
    {
        return err;
    }

    ifr.ifr_flags |= IFF_UP;
    if((err = ioctl(sockfd, SIOCSIFFLAGS, (void *)&ifr)) < 0 ) 
    {
        return err;
    }

    close(sockfd);

    return 0;
} 

/* Configure a local IPv4 address and netmask for the device */
int tun_set_address(const char* dev,
                     const char* ip,
                      int prefix_len)
{
    char command[128];

    memset(command, 0, sizeof(command));
    
    sprintf(command, "ip addr add %s/%d dev %s > /dev/null 2>&1", ip, prefix_len, dev);

    int result = system(command);
    
    return result;
} 

int tun_set_route()
{
    char command[128];

    memset(command, 0, sizeof(command));

    sprintf(command,
           "ip route add %s/24 via %s > /dev/null 2>&1", // ip -4 route add 
            SPOOF_NET, TUN_ADDR);

    int result = system(command);
    
    return result;
} 

void* server_thread(void* args)
{
	int listenfd;
	struct sockaddr_in servaddr;

	listenfd = socket(PF_INET, SOCK_STREAM, 0);

	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family = AF_INET;
	servaddr.sin_addr.s_addr = inet_addr(TUN_ADDR);
	servaddr.sin_port = htons(TUN_PORT);

	bind(listenfd, (struct sockaddr *)&servaddr, sizeof(servaddr));

	listen(listenfd, BACKLOG);
	while(1)
	{
		sleep(1);
	}

 	return NULL;
}

int server_setup()
{
	pthread_t thread;
	if (pthread_create(&thread, NULL, server_thread, NULL) != 0) {
		perror("pthread error");
		return -1;
	}	
}


void syn_send(int tun_fd, int i)
{
	char buffer[PCKT_LEN];
	struct iphdr *ip = (struct iphdr *) buffer;
	struct tcphdr *tcp = (struct tcphdr *)(buffer + sizeof(struct iphdr));
	uint16_t tot_len = sizeof(struct iphdr) + sizeof(struct tcphdr);
	uint16_t opt_len = 0;
	char* opt = (char*)(buffer + tot_len); // TCP option 
	memset(buffer, 0, PCKT_LEN);    

	if (spoof_mss[i] != 0)
	{
		struct mss_opt mss_opt;

		memset(&mss_opt, 1, sizeof(mss_opt));

		mss_opt.kind = 2;
		mss_opt.length = 4;
		mss_opt.mss = htons(spoof_mss[i]);

		memcpy(&opt[opt_len], &mss_opt, sizeof(mss_opt));  

		// if we have mss option
		tot_len += sizeof(mss_opt);
		opt_len += sizeof(mss_opt);
	}

	if (spoof_tsp[i] != 0)
	{
		struct tstamp_opt ts_opt;

		memset(&ts_opt, 1, sizeof(ts_opt));

		ts_opt.kind = 8;
		ts_opt.length = 10;
		ts_opt.tsval = htonl(spoof_tsp[i]);
		ts_opt.tsecr = 0;

		memcpy(&opt[opt_len], &ts_opt, sizeof(ts_opt));
		tot_len += sizeof(ts_opt);
		opt_len += sizeof(ts_opt);
	}

	if (spoof_wscale[i] != 0)
	{
		struct wscale_opt wscale_opt;

		memset(&wscale_opt, 1, sizeof(wscale_opt));

		wscale_opt.kind = 3;
		wscale_opt.length = 3;
		wscale_opt.scale = spoof_wscale[i];
		
		memcpy(&opt[opt_len], &wscale_opt, sizeof(wscale_opt));
		tot_len += sizeof(wscale_opt);
		opt_len += sizeof(wscale_opt);
	}

	ip->ihl = 5;
	ip->version = 4;
	ip->tos = 16;
	ip->tot_len = htons(tot_len);
	ip->id = htons(60000 + i);
	ip->frag_off = 0;
	ip->ttl = 64;
	ip->protocol = 6; // TCP 
	ip->saddr = inet_addr(spoof_ip_list[i]);
	ip->daddr = inet_addr(TUN_ADDR);
	ip->check = calc_cksm((unsigned short *)buffer,sizeof(struct iphdr));

	tcp->th_sport = htons(60000 + i);
	tcp->th_dport = htons(TUN_PORT);
	tcp->th_seq = htonl(1);
	tcp->th_ack = 0;
	tcp->th_off = (sizeof(struct tcphdr) + opt_len + sizeof(uint32_t) - 1) / sizeof(uint32_t);
	tcp->th_flags = TH_SYN;
	tcp->th_win = htons(4096);
	tcp->th_urp = 0;
	tcp->th_sum = 0;
	tcp->th_sum = tcp_checksum(ip, tcp, opt, opt_len);  

	if(write(tun_fd, buffer, tot_len) < 0)
	{
		perror("write() error");
		exit(-1);
	}
	else
	{
		printf("send packet %d\n", i);
	}

	return;
}

int main(int argc, char *argv[])
{
    int tun_fd, err;

    
    struct sockaddr_in sin, din;
    int one = 1;
    const int *val = &one;

    
    tun_fd = tun_create(IFF_TUN | IFF_NO_PI);
    if (tun_fd < 0)
    {
        perror("tun_create");
        return 0;
    }
    
    if (tun_setup("tun0") < 0)
    {
        perror("tun_setup");
        return 0;
    } 
    
    if (tun_setup("tun0") < 0)
    {
        perror("tun_setup");
        return 0;
    }

    if (tun_set_address("tun0", TUN_ADDR, 24) < 0)
    {
        perror("set address");
        return 0;
    }

    if (tun_set_route() < 0)
    {
        perror("set address");
        return 0;
    }

 	server_setup();

    sleep(5);

	for(int i = 0; i < COUNT; i++)
    {
        syn_send(tun_fd, i);
		
        usleep(10000);
    }

 	sleep(5);
	
    close(tun_fd);

    return 0; 
}
```

# 总结

## syn-cookie的理解
SYN cookie是服务器专门选择的初始序列号(ISN: initial seq num)，用于对握手的链接信息进行编码，使其能够忘记部分建立的连接，直到客户端用ACK进行回复以完成连接建立。

在SYN cookie条件下，当服务器接收到客户端的第一个ACK以完成连接的建立时，关键是服务器没有半建立连接的记录。它必须仅使用客户端的ACK数据包中的信息重新创建连接状态，即端点的地址和端口号、客户端的初始序列号、其自身的初始序列编号和最大段大小。


## DPVS中的syn-cookie的问题
### DPVS中的syn-cookie校验只和ack包有关系
由于`syn-cookie` 就是为了去除状态的记录。
正常情况下，收到 `syn`包，回复`syn-ack`包，此时没有任何记录信息或者状态或者会话，等到收到三次握手的 `ack`包时，**仅仅基于ack包中的信息**(`seq_num、ack_num、五元组信息等等`)检查是否符合`cookie`检查。正常情况下，如果是`client`的内核回复的，而不是攻击者构建的`ack`包，正常都是可以检测通过的。

如果是攻击者自己构建的`ack`包，甚至攻击者都没有发送过`syn`包，`dpvs`也没回复`syn-ack`，攻击者仅仅发送一个`ack`包, 理论上也是至少有`1/4G`（`ack_num为32bit`）的概率可以通过`cookie`检查，进而构建`conn`会话。

即：**收到 `ack`包时，仅仅基于这个`ACK`包进行`cookie` 校验，不依赖之前的状态或者会话**。其实是不知道这个`ack`是不是三次握手的`ack`，也不知道之前是否收到了`syn`，以及是否回复了`syn-ack`。

### conn-reuse场景下无法识别ack是特定的四次挥手的最后ack和新的三次握手的ack
**（1）正常情况下**
如果是 四次挥手的ack包，在收到此包之前的conn应该是已经处于`CLOSE_WAIT`或者`FIN_WAIT`或者`LAST_ACK`的状态，期望的状态是直接将收到的ACK 转发给 后端的RS，关闭连接就可以了。


如果是新的三次握手的ack，该报文又是复用了之前的五元组，但是`DPVS`中的老的 `Conn`还没有删除，收到 ack，


### 用户的请求时延变高问题
`LB`中的 `syn-proxy`是使用 `syn-cookie` 实现的;
`VS` 开启了 `syn-proxy`后，正常情况下，用户的`HTTP`请求的耗时会增加一个`RTT`。
![](attachments/image%20(4).png)



# 参考
```c
# 深入浅出TCP中的SYN-Cookies
https://switch-router.gitee.io/blog/TCP-SYN-Cookies/
https://segmentfault.com/a/1190000019292140

# TCP SYN Cookie [讲述了syn cookie的缺点]
https://cs.pynote.net/net/tcp/202205052/
```