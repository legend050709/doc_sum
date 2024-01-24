```table-of-contents
```

# 问题
一个域名，存在多个A/AAAA记录。DNS返回的顺序的问题，Client是默认选择第一个记录，还是如何选择的问题。

其他问题：一个设备上存在多个IP地址，那么在往外发包的时候，该如何选择地址的问题。

# DNS的负载均衡
作为互联网的基本设施，DNS 通过将域名转换为一组 IP 地址，在不同的连接尝试中，客户端将接收该域名服务的不同 IP，从而将整体负载分配到不同服务器之间。

在一些对响应延迟极度敏感的场景下，服务端负载不均会显著增加 P99/P999 延迟，例如：Redis 服务接入。假如后端服务能力一致，使用 DNS 作为服务发现的情况下，怎样才能让负载均衡到不同的服务器。通常意义上，我们倾向于认为 **DNS 解析**返回的结果是 Round-robin 的，然而实际上并非如此。

如下所示，2次`dig`返回的结果是随机的。
```bash
# dig -t A www.baidu.com +short
www.a.shifen.com.
110.242.68.4
110.242.68.3

# dig -t A www.baidu.com +short
www.a.shifen.com.
110.242.68.3
110.242.68.4
```
## DNS轮询(round-robin DNS)
DNS 轮询是一种 DNS 负载均衡技术，旨在分散流量并确保服务的高可用性。
它通过在 DNS 响应中列出一个域名的多个 IP 地址，对于每次DNS响应，列表中的IP地址的顺序都会变化。传统上，IP客户端最初尝试使用从DNS查询返回的第一个地址进行连接，这样在不同的连接尝试中，第一个IP在不断变化，那么请求就会发给不同的服务器，从而分发流量到多个服务器。

### DNS 轮询的作用
- 负载分配
负载分配是分发流量到多个服务器上。
DNS 轮询是一种简单而有效的负载分配的方法，它可以将流量分发到不同的服务器。

- 高可用
轮询时DNS服务器会将所有的IP返回给浏览器，浏览器自身的机制会使其在连接出错时继续连接下一个IP，直到所有IP都无法连接或者连接成功为止。

### DNS轮询的缺点
- 负载分配不均匀
(1) 不支持权重
DNS负载均衡采用的是简单的轮询算法，不能区分服务器的差异，不能反映服务器的当前运行状态，不能做到为性能较好的服务器多分配请求。
（2）DNS缓存的影响
本地DNS服务器会**缓存**已解析的域名到IP地址的映射，这会导致使用该DNS服务器的用户在一段时间内访问的是同一台Web服务器，导致Web服务器间的负载不均匀。
此外，用户本地计算机也会缓存已解析的域名到IP地址的映射。当多个用户计算机都缓存了某个域名到IP地址的映射时，而这些用户又继续访问该域名下的网页，这时也会导致不同Web服务器间的负载分配不均匀。

- 可靠性低
假设一个域名DNS轮询多台服务器，如果其中的一台服务器发生故障，那么所有的访问该服务器的请求将不会有所回应。
即使从DNS中去掉该服务器的IP，但在Internet上，各地区电信、网通等宽带接入商将众多的DNS存放在缓存中，以节省访问时间，DNS记录全部生效需要几个小时，甚至更久。所以，尽管DNS轮询在一定程度上解决了负载均衡问题，但是却存在可靠性不高的缺点。

### bind9中的轮询配置
![](attachments/Pasted%20image%2020240117174217.png)
在 named.conf 中可以设置 bind 的 round-robin 的给出结果的顺序：  
```bash
options {  
	rrset-order { order random; };  
};

```

`rrset-order` 可以设置在 `option`中，也可以设置在view中。

`rrset-order` 支持三个参数：fixed, random, cyclic 。  
- fix： 会将多个A记录按配置文件的顺序固定给出  
- random： 会随机给出  
- cyclic： 会循环给出.

### bind9中 `sortlist`的设置
`sortlist` 可以设置在 `option`中，也可以设置在view中。
![](attachments/Pasted%20image%2020240117173720.png)


**`sortlist` 中只有一个元素**
那么它会被用来核对查询者的IP地址。如果查询者的地址匹配，则名称服务器会对发送给查询者的应答进行排序，以使任何匹配该元素的地址都被放在最前面。**只有一个条目，则匹配查询者的IP，也对响应的多记录进行排序**
![](attachments/Pasted%20image%2020240117172800.png)
这个排序列表中只有一条包含一个元素的条目。位于该网络的查询者查询，该列表会将位于网络192.249.249/24的地址排在应答的最前面。

**`sortlist` 中包含2个元素**
第一个元素被用来匹配查询者的IP地址。如果查询者的地址匹配，名称服务器会对发送给查询者的应答进行排序，以使任何匹配第二个元素的地址都被放在最前面。
第二个元素实际上可以是一个包含许多元素的地址匹配列表，在这种情况下，应答中的第一个地址就是第一个匹配该列表的地址。
![](attachments/Pasted%20image%2020240117173012.png)
这个排序列表会作用在位于192.249.249/24的查询者上，并且发送给它们的应答会将它们自己的网络地址排在最前面，接着才是位于192.253.253/24的地址。

排序列表中的元素，可以是简单的子网甚至是单独的主机。
![](attachments/Pasted%20image%2020240117173046.png)

### 小结
DNS轮循本身可能不是负载均衡的最佳选择，因为只是在每次查询名称服务器时交替IP地址记录的顺序。由于DNS **Cache缓存**的存在，DNS轮循只会进行**负载分配，而不是负载均衡**。即：DNS 的轮询机制并不能做为一个 `load balancing`的解决方案，只能作为一个 `load distribution` 方案。

## 其他DNS负载均衡算法
Round robin (RRS)： 将工作平均的分配到服务器 (用于实际服务主机性能一致)
Least-connections (LCS)： 向较少连接的服务器分配较多的工作(IPVS表存储了所有的活动的连接。用于实际服务主机性能一致。)
Weighted round robin (WRRS)： 向较大容量的服务器分配较多的工作。可以根据负载信息动态的向上或向下调整。 (用于实际服务主机性能不一致时)；
Weighted least-connections (WLC)： 考虑它们的容量向较少连接的服务器分配较多的工作。容量通过用户指定的砝码来说明，可以根据装载信息动态的向上或向下调整。(用于实际服务主机性能不一致时)；
## 多个记录的返回顺序
如果一个域名有多条 A 记录，当发送 DNS 请求时：
1. DNS 服务是否会返回全部记录？
2. DNS 服务会以什么顺序返回记录？

由于 RFC 缺少相关的规定，在传输协议的范围内，不同的名称服务器有不同的路由策略。两者共同决定了返回的记录和顺序。

### 返回记录的个数限制
因此基于 UDP 的 DNS ，有效载荷限制为小于 512 字节，保证了如果 DNS 数据包在传输中被分段，可以重新组装，降低数据包被随机丢弃的可能性。超过 512 字节的响应将被截断，解析器必须通过 TCP 重新发出请求。
如果解析器支持 EDNS0，也可以通过 UDP 响应最多 4096 字节，且不会被截断。

### DNS的记录返回顺序
常见的一种DNS路由策略(routing policy)设置是：轮询 DNS。
即 DNS响应中，一个域名的多个A记录的顺序是 Round-robin 的。
当查询有多条记录时，名称服务器执行循环 DNS。在一个请求和下一个请求时，发送响应的顺序会有所不同。大多数客户端将连接到第条记录，因此可以实现负载平衡。

分别使用 `8.8.8.8` 和 CoreDNS 分别作为名称服务器。前者直接解析返回，后者配置 `loadbalance round_robin` shuffle 返回。
```bash
loadbalance [round_robin | weighted WEIGHTFILE] { reload DURATION }
```

查看 `serverfault.com` 返回记录的顺序，可以看到响应首位的结果差异:
```bash
$> dig +short serverfault.com
104.18.23.101
104.18.22.101

# 8.8.8.8
$> for i in $(seq 1 10); do  dig +short serverfault.com | head -n 1; done | sort | uniq -c
     10 104.18.23.101

# CoreDNS: round-robin
 $> for i in $(seq 1 10); do  dig +short serverfault.com | head -n 1; done | sort | uniq -c
      4 104.18.22.101
      6 104.18.23.101
```

除了 CoreDNS 的 `round-robin`，AWS route 53 之类的 DNS 服务提供了更多路由策略，常见：
```bash
- Geolocation routing policy
- IP-based routing policy
- Weighted routing policy
```
**值得注意的是，由于 CoreDNS 等下游递归解析器，在启用缓存时，并不感知上游的路由策略，因此会导致上游策略失效，甚至导致缺陷。**
> 假设，上游域名服务随机返回部分 IP，该部分 IP 会持续缓存直至缓存失效。在缓存失效前所有请求都会集中到该部分 IP，导致较为严重的访问倾斜。

## client的dns请求 Resolver库

在 Linux 上并不存在一个 `syscall` 用于域名解析，实际上**大多数**程序是通过一个 C 标准库调用 `getaddrinfo`或者 `gethostbyname` 完成的。
> `dig` 、`nslookup` 等，是查询 DNS 域名服务的工具，因此没有调用 `resolver` 库

通过 strace 命令可以看到执行的部分细节：
```bash
$> strace -e trace=openat -f ping -c1 serverfault.com
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libcap.so.2", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libidn2.so.0", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libunistring.so.2", O_RDONLY|O_CLOEXEC) = 3
openat(AT_FDCWD, "/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 5
openat(AT_FDCWD, "/etc/host.conf", O_RDONLY|O_CLOEXEC) = 5
openat(AT_FDCWD, "/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 5
openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
openat(AT_FDCWD, "/etc/gai.conf", O_RDONLY|O_CLOEXEC) = 5
PING serverfault.com (104.18.22.101) 56(84) bytes of data.
openat(AT_FDCWD, "/etc/hosts", O_RDONLY|O_CLOEXEC) = 5
64 bytes from 104.18.22.101 (104.18.22.101): icmp_seq=1 ttl=62 time=68.6 ms
```
![](attachments/Pasted%20image%2020240110173606.png)

可以看到依次读取了 `/etc/nsswitch.conf`，`/etc/host.conf`，`/etc/resolv.conf` `/etc/gai.conf` 四个配置文件， DNS 解析的策略也跟他们相关。

### nsswitch.conf
Name Service Switch (NSS) 配置文件，管理了各种信息来源的类别和顺序。每一行可以当做是一个数据库，冒号前面的是信息类型，冒号后面是数据来源或服务。举例：
```bash
...
hosts:          files dns
networks:       files
...
```

域名解析时，`gethostbyname` 会读取 hosts 一行，并从 files 和 dns 两个来源依次获取数据：
- `/lib/libnss_files.so.X`：实现了 “files” 数据源，读取本地文件：`/etc/hosts`
- `/lib/libnss_dns.so.X`：实现 “dns” 数据源，访问远端 DNS 服务。

相比于固定搜索顺序的硬编码， NSS 提供了一种更灵活的方法可以动态更新搜索顺序，插件化的增减来源。

### host.conf
`host.conf` 包含了为解析库声明的配置信息. 每行含一个配置关键字，其后跟着合适的配置信息.。举例：
```bash
# The "order" line is only used by old versions of the C library.
order hosts,bind
multi on
```
- order：管理解析顺序。表示先使用 `/etc/hosts` 文件，再使用 name server 解析。bind(Berkeley Internet Name Domain)，一种开源 DNS 协议实现。
- multi on：允许主机名对应多个 IP 地址，如果机器有多张网卡，就设置为 on。

### resolv.conf
`resolv.conf` 是解析器的核心配置，举例：
```bash
$> cat /etc/resolv.conf
options rotate     
options timeout:2  
options attempts:3  
options single-request-reopen
nameserver 8.8.4.4
nameserver 8.8.8.8
```

其配置项既要满足解析的基本要求：

1. 首先，在发起查询前要填补 local domain 得到 **FQDN** (Fully Qualified Domain Name 全限定域名): `search`、`ndots:n`
2. 其次，有多个 nameserver 时，需要定义查询选择的 nameserver 策略: `nameserver`、`rotate`

> **配置 `rotate` 时**  
> 以 **Round Robin** 的形式挑选 `nameserver`，而非每次都选择第一个，起到**负载均衡**的的作用。一次性请求的工具不生效，因为只有一次请求。  
>   
> **不配置 `rotate` 时**  
> 首先使用第一个 nameserver  
> 如果请求成功，永远不会继续尝试后续的 nameserver  
> 如果请求失败且尚未超时，则继续使用后续 nameserver，直至成功

1. 再次，既然是远程调用，更要控制好请求超时时间，以及出错时的重试次数: `timeout`、`attempts`
2. 最后，支持对返回的多个结果排序: `sortlist`

也要兼容历史变迁的沧桑：

1. 首先，要兼容 IPv4 和 IPv6
2. 其次，数据包过大时，可以 TCP 解析: `use-vc`
3. 最后，兼容种种历史缺陷: `single-request-reopen`、`single-request`


### gai.conf
调用 `getaddrinfo` 可能会返回多个结果。根据 rfc3484 / rfc6724 的要求（相关排序机制也可以通过 `/etc/gai.conf` 配置控制），**DNS 解析返回的结果应当是固定顺序的，而非 round-robin**，那么当 DNS server 返回 round-robin的结果时，就会因为解析器的排序而不生效，导致新旧版本 library 之间行为不一。

最新的规范的前提都是 使用IPv6 进行DNS请求，然而 IPv6 到目前位置支持的并不理想，并且考虑基于兼容性的考虑：**当返回结果中仅有 IPv4 时，不适用最长匹配相关的规则，也就不会调整结果的相对顺序（稳定排序）**。

#### 范例
```c
# cat getaddr2.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <netdb.h>

int main(int argc, char *argv[]) {
    char *hostname = NULL;
    char *service = NULL;
    struct addrinfo hints, *result;
    memset(&hints, 0, sizeof(hints));

    if (argc <= 1){
        printf("argc <1 \n");
        exit(EXIT_FAILURE);
    }

    hostname = (char *)argv[1];
    hints.ai_family = AF_UNSPEC; // 允许 IPv4 或 IPv6
    hints.ai_socktype = SOCK_STREAM; // TCP协议
    int status = getaddrinfo(hostname, service, &hints, &result);
    if (status != 0) {
        fprintf(stderr, "getaddrinfo error: %s\n", gai_strerror(status));
        return 1;
    }
    for (struct addrinfo *p = result; p != NULL; p = p->ai_next) {
        char ip[INET6_ADDRSTRLEN];
        void *addr;
        if (p->ai_family == AF_INET) { // IPv4
            struct sockaddr_in *ipv4 = (struct sockaddr_in *)p->ai_addr;
            addr = &(ipv4->sin_addr);
        } else { // IPv6
            struct sockaddr_in6 *ipv6 = (struct sockaddr_in6 *)p->ai_addr;
            addr = &(ipv6->sin6_addr);
        }

        inet_ntop(p->ai_family, addr, ip, sizeof(ip));
        printf("IP address: %s\n", ip);
    }
    freeaddrinfo(result);
    return 0;
}

结果：
(1) 执行1
# ./getadd2 www.baidu.com
IP address: 110.242.68.3
IP address: 110.242.68.4
IP address: 2408:871a:2100:2:0:ff:b09f:237
IP address: 2408:871a:2100:3:0:ff:b025:348d

(2) 执行2
# ./getadd2 www.baidu.com
IP address: 110.242.68.4
IP address: 110.242.68.3
IP address: 2408:871a:2100:3:0:ff:b025:348d
IP address: 2408:871a:2100:2:0:ff:b09f:237

(3) 执行3
# ./getadd2 www.baidu.com
IP address: 110.242.68.4
IP address: 110.242.68.3
IP address: 2408:871a:2100:2:0:ff:b09f:237
IP address: 2408:871a:2100:3:0:ff:b025:348d
```

## golang的DNS请求
```go
func Dial(network, address string) (Conn, error)
```
Golang 创建连接时，使用 Dial 连接到 `named network` 的地址。

已知 `network` 类型有：
- TCP：”tcp”、”tcp4” (IPv4-only)、”tcp6” (IPv6-only)
- UDP：”udp”、”udp4” (IPv4-only)、”udp6” (IPv6-only)
- IP：”ip”、”ip4” (IPv4-only)、”ip6” (IPv6-only)
- Unix domain socket：”unix”, “unixgram” and “unixpacket”.

Golang 默认使用双栈（IPv4&IPv6）DNS 解析，当 IPV6 不能访问时，支持 IPv6 的程序需要延迟几秒钟才能正常切换到 IPv4，为了不影响用户体验可以指定 `network` 为 `tcp4`，直接禁用 IPv6。

## DNS 负载均衡小结
综述，一次 DNS 解析，如果指定 network 为 TCP，在启用 IPv6 时：
1. Golang Resolver 会并发发出 IPv4 和 IPv6 DNS 查询请求。查询的域名服务节点是 /etc/resolv.conf 指定的递归解析器。
2. 递归解析器如果从缓存中发现结果，则直接使用，否则递归查询上游的域名服务，并将结果缓存。得到结果之后，再根据路由策略返回。每一级域名服务均如是。
3. Golang net.Dial 选择 IP 列表中的第一个 IP 建立连接。

DNS 本身作为服务发现，通过轮询 DNS 提供了最基本的负载分配功能，而不能保证完美的负载均衡。对负载有极致需求的业务，建议自行负载均衡，策略参考：

1. **动态（定时）更新** DNS 对应的 IP 列表
2. 根据**负载均衡策略**从 IP 列表中选择合适的 IP
3. **根据 IP 从连接池中获取连接**，发起请求


# 多个IPv4记录
## 现象
一个域名可以解析出几个IP地址，例如在访问 `www.163.com`时，抓包得到的DNS响应包中有2个IP地址：221.229.167.47和58.220.39.91，如下图所示。
![](attachments/Pasted%20image%2020240110154344.png)
虽然DNS解析得到了多个IP，但是大多数软件只会使用第一个IP地址。

## `gethostbyname`介绍
TCP/IP网络通信是基于IP地址的，当要访问的服务器地址是域名时，就需要先把域名解析成IP地址。在TCP/IP API中有一个叫`gethostbyname`的函数，负责把域名解析成IP地址。 函数的原型定义如下，参数`name`就是要解析的域名。`man gethostbyname` 如下所示：
```bash
#include <netdb.h>
extern int h_errno;
struct hostent *gethostbyname(const char *name);

/* GNU extensions */
struct hostent *gethostbyname2(const char *name, int af);

struct hostent {
   char  *h_name;            /* official name of host */
   char **h_aliases;         /* alias list */
   int    h_addrtype;        /* host address type */
   int    h_length;          /* length of address */
   char **h_addr_list;       /* list of addresses */
}
#define h_addr h_addr_list[0] /* for backward compatibility */
```

结构体中的`h_addr_list`是一个数组，用于存放解析出的多个IP地址，但很少有程序员会去考虑多个IP地址的问题，通常直接使用宏`h_addr`来获取IP地址，也就是第一个IP地址。

一些大型网站或CDN服务商为了实现负载均衡，他们的DNS服务器会动态改变多个IP地址的顺序，使得每个IP地址都有机会成为解析结果中的第一个IP地址。
## DNS负载均衡的解决方案
- 应用程序
应用程序得到一个域名的多个A记录时，随机选择某个A记录，而不是每次选择第一个A记录。

- DNS 解析器设置路由策略
DNS解析器，返回一个域名的多个记录的时候，设置路由策略，比如随机变换A记录的次序。

## `gethostbyname`的使用范例

```c
#include <sys/types.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <stdio.h>
#include <errno.h>

int main(int argc, char *argv[])
{
    struct hostent *hp=NULL;
    struct in_addr ip_addr;
    char str[64];

    if (argc <= 1){
        printf("argc <1 \n");
        exit(EXIT_FAILURE);
    }
    hp = gethostbyname(argv[1]);
    if (!hp){
        printf("%s was not resolved %d\n", argv[1], errno);
        exit(EXIT_FAILURE);
    }
    char **pptr;
    for (pptr = hp->h_addr_list; *pptr!=NULL; pptr++) {
        printf("HostName: %s was resolved to :%s \n",
                argv[1], inet_ntop(hp->h_addrtype, *pptr, str, sizeof(str)));
    }
    exit(EXIT_SUCCESS);
}
```

strace 结果如下所示：
```bash
1）demo编译：
gcc gethostbyname.c -o gethostbyname

2） 配置文件：
/etc/nsswitch.conf 中
    hosts: files dns

/etc/resolv.conf 中
    nameserver  223.5.5.5

service nscd status
    inactive

3) strace 查看：
strace ./gethostbyname www.baidu.com > gethostbyname.default.log 2>&1
```
```bash
  1 execve("./gethostbyname", ["./gethostbyname", "www.baidu.com"], [/* 22 vars */]) = 0
...
//载入resolv.conf host.conf nsswitch.conf 文件
 26 stat("/etc/resolv.conf", {st_mode=S_IFREG|0644, st_size=241, ...}) = 0
 27 open("/etc/host.conf", O_RDONLY|O_CLOEXEC) = 3
 28 fstat(3, {st_mode=S_IFREG|0644, st_size=92, ...}) = 0
 29 read(3, "# The \"order\" line is only used "..., 4096) = 92
 30 read(3, "", 4096)                       = 0
 31 close(3)                                = 0
 32 open("/etc/resolv.conf", O_RDONLY|O_CLOEXEC) = 3
 33 fstat(3, {st_mode=S_IFREG|0644, st_size=241, ...}) = 0
 34 read(3, "# This file is managed by man:sy"..., 4096) = 241
 35 read(3, "", 4096)                       = 0
 36 close(3)                                = 0
...
//ncsd  
uname({sysname="Linux", nodename="vm-134", ...}) = 0
 38 socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
 39 connect(3, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
 40 close(3)                                = 0
 41 socket(AF_UNIX, SOCK_STREAM|SOCK_CLOEXEC|SOCK_NONBLOCK, 0) = 3
 42 connect(3, {sa_family=AF_UNIX, sun_path="/var/run/nscd/socket"}, 110) = -1 ENOENT (No such file or directory)
 43 close(3)                                = 0
...

 44 open("/etc/nsswitch.conf", O_RDONLY|O_CLOEXEC) = 3
 45 fstat(3, {st_mode=S_IFREG|0644, st_size=497, ...}) = 0
 46 read(3, "# /etc/nsswitch.conf\n#\n# Example"..., 4096) = 497
 47 read(3, "", 4096)                       = 0
 48 close(3)                                

// 链接动态库根据nsswitch 查找相关函数 这里由于配置了files先查找libnss_files.so.2动态库中指定函数
 54 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libnss_files.so.2", O_RDONLY|O_CLOEXEC) = 3
 55 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\360!\0\0\0\0\0\0"..., 832) = 832
 56 fstat(3, {st_mode=S_IFREG|0644, st_size=47608, ...}) = 0
...
 64 open("/etc/hosts", O_RDONLY|O_CLOEXEC)  = 3
 65 fstat(3, {st_mode=S_IFREG|0644, st_size=348, ...}) = 0
 66 read(3, "127.0.0.1\tlocalhost\n127.0.1.1\twe"..., 4096) = 348
 67 read(3, "", 4096)                       = 0
 68 close(3)    

// 没有结果，通过dns方式查找相关函数, 最终走的是res_search等相关方法，
74 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libnss_dns.so.2", O_RDONLY|O_CLOEXEC) = 3
 75 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\20\20\0\0\0\0\0\0"..., 832) = 832
 76 fstat(3, {st_mode=S_IFREG|0644, st_size=27016, ...}) =  0
...
 82 openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libresolv.so.2", O_RDONLY|O_CLOEXEC) = 3
 83 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0p8\0\0\0\0\0\0"..., 832) = 832
 84 fstat(3, {st_mode=S_IFREG|0644, st_size=97136, ...}) = 0

// 可以看到基于UDP协议进行了域名查询
 93 socket(AF_INET, SOCK_DGRAM|SOCK_CLOEXEC|SOCK_NONBLOCK, IPPROTO_IP) = 3
 94 connect(3, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("223.5.5.5")}, 16) = 0
 95 poll([{fd=3, events=POLLOUT}], 1, 0)    = 1 ([{fd=3, revents=POLLOUT}])
 96 sendto(3, "B\217\1\0\0\1\0\0\0\0\0\0\3www\5baidu\3com\0\0\1\0\1", 31, MSG_NOSIGNAL, NULL, 0) = 31
 97 poll([{fd=3, events=POLLIN}], 1, 5000)  = 1 ([{fd=3, revents=POLLIN}])
 98 ioctl(3, FIONREAD, [90])                = 0
 99 recvfrom(3, "B\217\201\200\0\1\0\3\0\0\0\0\3www\5baidu\3com\0\0\1\0\1\300"..., 1024, 0, {sa_family=AF_INET, sin_port=htons(53), sin_addr=inet_addr("223.5.5.    5")}, [28->16]) = 90
100 close(3)                                = 0
```

## `gethostbyname`的问题

### 不支持ipv6的问题
`gethostbyname()` 只适用 IPv4，涉及 IPv6 就崩溃了。它必须被更好的东西取代。
`getaddrinfo `支持 IPv6 和更多功能。

### 阻塞问题
**问题**：
由于DNS的递归查询，常常会发生`gethostbyname`函数在查询一个域名时严重超时。而该函数又不能像`connect`和`read`等函数那样通过`setsockopt`或者`select`函数那样设置超时时间，因此常常成为程序的瓶颈。

在多线程下面，`gethostbyname`会一个更严重的问题，就是如果有一个线程的`gethostbyname`发生阻塞，其它线程都会在`gethostbyname`处发生阻塞。


**解决方法**：
（1）方法一：
首先使用`gethostbyname_r`函数保证线程安全;
再通过修改DNS的配置文件`/etc/resolv.conf`来设置超时时间解决阻塞问题。

（2）方法二：
- 使用alarm设定信号，如果超时就用`sigsetjmp`和`siglongjmp`跳过`gethostbyname`函数。
- 独立开启一个线程来调用`gethostbyname`函数，该线程除了调用此函数外，不做任何事情。

范例如下所示：
```c
#include <setjmp.h>
#include <time.h>
 
static sigjmp_buf jmpbuf;
static void alarm_func()
{
     siglongjmp(jmpbuf, 1);
}
 
static struct hostent *gngethostbyname(char *HostName, int timeout)
{
     struct hostent *lpHostEnt;
 
     signal(SIGALRM, alarm_func);
     if(sigsetjmp(jmpbuf, 1) != 0)
     {
           alarm(0); /* 取消闹钟 */
           signal(SIGALRM, SIG_IGN);
           return NULL;
     }
     alarm(timeout); /* 设置超时时间 */
     lpHostEnt = gethostbyname(HostName);
     signal(SIGALRM, SIG_IGN);
 
     return lpHostEnt;
}
```

# 多个IPv6 以及 IPv4和IPv6混合问题
## 背景
![](attachments/Pasted%20image%2020240111141153.png)

在 IPv6+IPv4 双栈下，DNS查询会同时发送 AAAA 和 A 解析，无论访问域名有没有 AAAA 解析都会浪费一定时间去查询。如果访问的域名同时拥有 A 和 AAAA 解析，那么 Linux 系统会优先使用 AAAA 解析，也就是 IPv6 地址，同时网络出口的优先级都会比 IPv4 高。

## `getaddrinfo`介绍

IPv6中引入了`getaddrinfo()`的新API，它是协议无关的，既可用于IPv4也可用于IPv6。
![](attachments/Pasted%20image%2020240110162854.png)
```c
int getaddrinfo(const char *node, const char *service,
               const struct addrinfo *hints,
               struct addrinfo **res);

struct addrinfo {
   int              ai_flags;
   int              ai_family;
   int              ai_socktype;
   int              ai_protocol;
   socklen_t        ai_addrlen;
   struct sockaddr *ai_addr;
   char            *ai_canonname;
   struct addrinfo *ai_next;
};
```
## `getaddrinfo` 没有轮询
![](attachments/Pasted%20image%2020240110171006.png)
调用 `getaddrinfo` 可能会返回多个结果。根据 rfc3484 / rfc6724 的要求，需要**根据来源 IP 与结果 IP 进行最长匹配排序，以便相同子网里的 IP 在列表中排在首位，以得到成功率最高的结果**。换句话说，按照最新规范，**DNS 解析返回的结果应当是固定顺序的，而非 round-robin**。

## `getaddrinfo`范例
见上`getaddrinfo` 的范例.
```结果：
# gcc getaddr2.c -o getadd2

# ./getadd2 www.baidu.com
IP address: 110.242.68.4
IP address: 110.242.68.3
IP address: 2408:871a:2100:2:0:ff:b09f:237
IP address: 2408:871a:2100:3:0:ff:b025:348d
```
抓包如下所示：
![](attachments/Pasted%20image%2020240110164056.png)
如上所示，使用相同的五元组发送了2个请求包，而不是一个包中2个请求。

# RFC3484
参考：[RFC 3484介绍](https://rfc2cn.com/rfc3484.html)
![](attachments/Pasted%20image%2020240111142801.png)

优先级以及Label 介绍
![](attachments/Pasted%20image%2020240111144704.png)

## 理解
具体来讲，在尚未确定源IPv6地址的时候，给出一台机器上不同网卡上配置的所有IPv6地址，作为一个集合，遵循 **RFC3484给出的8干条Rule**，最终将会确定一个唯一的一个最优IP地址作为通信的源IP地址。

这8条Rule **依次顺序匹配**，每一条Rule简单来讲都是一个比较排序规则，它将一些候选的IPv6地址排在另一些候选IPv6地址之前或者之后，排在后面的地址将再没有机会被选中而被淘汰。
只有在一条Rule没有确定唯一的优胜者时，后面的Rule才会起作用。很显然，这是一个逐步细化的不同维度的排序过程，如果前一条Rule选择了多个优胜者，将会进行下一条Rule规定的竞选。

这8条Rule分别是：
Rule 1: 优选与目标地址相同的地址
Rule 2: 优选其scope与目标地址scope相比更接近且scope更大的地址
Rule 3: 优选"preferred" 地址
Rule 4: 优选home地址
Rule 5: 优选和路由出口在同一块网卡上配置的地址
Rule 6: 优选Label匹配的地址
Rule 7: 优选非临时/临时地址(可配置)
Rule 8: 优选和目标地址最长前缀匹配的地址

### Rule 2
#### scope
IPv6在设计时就对地址 按照其作用域范围(scope) 进行了 分类，比如：

fe80::/10： 链路本地（link local）地址，不能出三层设备，只能用于本二层链路
fec0::/10： 站点本地(site local)地址，已废弃
fc00::/7： ULA(uniqic local addr)地址，不能在公网路由，但可用于私有网络，自己把握
2002::/16： IPv6/v4兼容地址
2001::/32： IPv6/v4兼容地址
全球全局地址：global uniqic address

这些作用域，比如说链路本地，站点本地，全局等等，在标准上就限制了这些地址能被路由的范围。
如果一台公网路由器的协议栈是严格按照标准实现的，那么当它发现类似fc00::/7这种地址的时候，就应该将其丢弃；另外，路由器也不能转发fe80::/10这种链路本地地址，这相比IPv4 **只要有路由就能通，一切靠路由**要严格太多了，这种严格大大减轻了配置管理员的负担！

所以说，scope其实就是一个 **硬约束**。
两个地址如果要通信，其socpe越接近，其 **在物理上就越接近**，与此相对，还有一个 **软约束**， 即最长前缀匹配度，两个地址的最长前缀越匹配，表明二者 **在拓扑上越接近**。
#### iPv4 scope
IPv4没有这个scope的概念，但是在很多操作系统协议栈中，它依然被 **模拟** 了出来。
比如对于Linux内核而言，它就将127.0/8以及169.254这种DHCP失败的地址视为了本机scope地址，本地链路scope地址，意思是说它们已经被限制住了范围，此外，关于路由scope也是一样，Linux内核将链路层自己发现的路由，其scope定为link范围，而非自动发现的路由，其scope则是global。
![](attachments/Pasted%20image%2020240111153731.png)
#### 理解
Rule 2有两层意思：
1. **优选更大scope**
    这是因为越大的scope，其可达性范围越广。这保证了源和目标通信节点的连通性。
2. **优选与目标地址的scope更接近的scope**
    这是因为，如果源和目标在scope约束下能通信，那么scope越接近，它们在物理上就越接近！
### Rule3
在Linux上废弃一个地址非常容易，执行下面的语句就可以了：
```bash
ip addr change $IPv6_addr_to_deprecated dev tun preferred_lft 0
```
### Rule 6
IPv6允许为每一个IPv6地址配置一个元组映射：M:{地址前缀，优先级，Label}。
该元组映射作为系统的一张策略表存在，用于选择源IP地址或者目标IP地址。

如果Rule 1～Rule 5均没有选择出优选IP地址，也就是说多个幸存者IP无法决胜负，成了平局，那么在从剩余的幸存者候选IP地址中选择的时候，执行以下序列：
![](attachments/Pasted%20image%2020240111153954.png)
即：基于目标IP的最长前缀匹配，找到对应的标签。然后再选择同标签，且优先级大的Sip。

如下方式，查看M映射表。
```bash
# ip addrlabel show
prefix ::1/128 label 0
prefix ::/96 label 3
prefix ::ffff:0.0.0.0/96 label 4
prefix 2001::/32 label 6
prefix 2001:10::/28 label 7
prefix 3ffe::/16 label 12
prefix 2002::/16 label 2
prefix fec0::/10 label 11
prefix fc00::/7 label 5
prefix ::/0 label 1
```

### Rule 7
在IPv6中，有一种生命周期有限的地址，称作 **临时地址(含有tentative标记)**。临时地址一般用于私密通信，用完即丢。
比如：刚配置一个IPv6地址到接口上，刚配置上去，则含有tentative标记，dad检测没有问题之后，则会将该标记去掉。
![](attachments/Pasted%20image%2020240111154254.png)

## gai.conf 文件
![](attachments/Pasted%20image%2020240110181853.png)
注： gai.conf 中的优先级的值越大，就是越优先。
标签（Label）类似于分组。

## ipv6 源地址的选择
看Linux内核的源码，以选择源地址为例，整个地址选择浓缩在了 **ipv6_dev_get_saddr**函数中。
简单看了一下，该函数其实就是对机器上所有的地址进行了一个冒泡排序而已。排序的规则就是从Rule 1到Rule 8，遍历两层嵌套，外层是遍历设备，内层是遍历每一个设备的所有IPv6地址。

```c
    for_each_netdev_rcu(net, dev) {
        struct inet6_dev *idev;

        /* Candidate Source Address (section 4)
         *  - multicast and link-local destination address,
         *    the set of candidate source address MUST only
         *    include addresses assigned to interfaces
         *    belonging to the same link as the outgoing
         *    interface.
         * (- For site-local destination addresses, the
         *    set of candidate source addresses MUST only
         *    include addresses assigned to interfaces
         *    belonging to the same site as the outgoing
         *    interface.)
         */
        if (((dst_type & IPV6_ADDR_MULTICAST) ||
            // 如果是链路本地地址，不允许跨网卡！
             dst.scope <= IPV6_ADDR_SCOPE_LINKLOCAL) &&
            dst.ifindex && dev->ifindex != dst.ifindex)
            continue;
       ...
   }

```

## 范例
理解RFC3484最好的方法不是使劲读文档，也不是思考，最好的方式就是直接去读RFC3484后面的实例，这些实例非常好。

### Default Source Address Selection
```text
The source address selection rules, in conjunction with the default
policy table, produce the following behavior:

Destination: 2001::1
Candidate Source Addresses: 3ffe::1 or fe80::1
Result: 3ffe::1 (prefer appropriate scope)

Destination: 2001::1
Candidate Source Addresses: fe80::1 or fec0::1
Result: fec0::1 (prefer appropriate scope)

Destination: fec0::1
Candidate Source Addresses: fe80::1 or 2001::1
Result: 2001::1 (prefer appropriate scope)

Destination: ff05::1
Candidate Source Addresses: fe80::1 or fec0::1 or 2001::1
Result: fec0::1 (prefer appropriate scope)

Destination: 2001::1
Candidate Source Addresses: 2001::1 (deprecated) or 2002::1
Result: 2001::1 (prefer same address)

Destination: fec0::1
Candidate Source Addresses: fec0::2 (deprecated) or 2001::1
Result: fec0::2 (prefer appropriate scope)

Destination: 2001::1
Candidate Source Addresses: 2001::2 or 3ffe::2
Result: 2001::2 (longest-matching-prefix)
…
```

### Default Destination Address Selection
IPv6和IPv4的默认选择规则：
![](attachments/Pasted%20image%2020240111160040.png)

IPv4优先：
![](attachments/Pasted%20image%2020240111160336.png)

IPv6全局唯一地址优先：
![](attachments/Pasted%20image%2020240111160705.png)

### 其他应用
环境：站点A存在2个地址，一个地址作为和B高性能通信的地址，一个地址作为和其他站点通信的地址；站点B存在2个地址，一个地址作为和A高性能通信的地址，一个地址作为和其他站点通信的地址。

期望：
A和B之间通信使用高性能地址通信；A或者B和其他站点通信使用普通地址通信。
![](attachments/Pasted%20image%2020240111161730.png)

# 参考
```c
# linux C 使用getaddrinfo解析域名
https://avmedia.0voice.com/?id=49938

# 闲谈IPv6-源IP地址的选择(RFC3484读后感)
https://blog.csdn.net/dog250/article/details/87815123

# 译｜getaddrinfo with round robin DNS and happy eyeballs
https://www.cyningsun.com/09-07-2023/getaddrinfo-with-round-robin-dns-and-happy-eyeballs-cn.html

# 深入理解 DNS 解析
https://www.cyningsun.com/10-08-2023/dive-into-dns-resolution.html

#IPv4/IPv6双栈网络下配置IPv4链路优先
https://kanochan.net/archives/3249.html
```