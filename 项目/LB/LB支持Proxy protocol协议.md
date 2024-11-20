
```table-of-contents
```
# 背景
## toa存在的问题
通过TOA/UOA方式获取Client的IP/Port存在的问题，如下所示：
### TCP Option空间有限，可能无法插入额外信息
如IPv6场景下，同时插入CIP/CPort以及 VIP/VPort
### 攻击者在TCP OPtion中伪造假的CIP/CPort的风险
参考：[# L4LB网络中间件DPVS在FNAT TOA模式存在IP伪造漏洞](https://www.ctfiot.com/149666.html)
## uoa存在的问题
UDP场景下，后端缺失UOA模块会导致流量不通
UDP场景下，更改IP协议号为248的报文可能被部分交换机/路由器丢弃
内核模块升级可能导致所有表项丢失，获取CIP/CPort失败
内核模块中表项的刷新/超时时间和KGW会话的刷新/超时时间不一致问题
## uoa/toa共同的问题
后端需要安装内核模块，风险高：
1》不同内核版本需要不同版本的内核模块
2》内核模块异常可能导致整机宕机

# PP 介绍
PP(proxy protocol)是[haproxy的方案](https://github.com/haproxy/haproxy/blob/master/doc/proxy-protocol.txt)，通过为tcp添加一个很小的头信息，来方便的传递客户端信息（协议栈、源IP、目的IP、源端口、目的端口等)，在网络情况复杂又需要获取用户真实IP时非常有用。其本质是在三次握手结束后由代理在连接中插入了一个携带了原始连接四元组信息的数据包。

PP协议是在TCP连接的开头插入了一段PP协议头。后端服务器处理新的socket的时候，先读取这段数据，解析其中的IP，就可以获取到客户端的真实源IP。

![](attachments/image%20(10).png)

## 注意事项

● 后端服务器修改代码才能支持
● PP协议存在2个版本，版本1和版本2
● 对于服务器的同一个监听端口，不存在兼容带Proxy Protocol包的连接和不带Proxy Protocol包的连接。如果服务器接收到的第一个数据包不符合Proxy Protocol的格式，那么服务器会直接终止连接。
● 后端服务器如果直连公网，需要检查socket的源IP，如果是DPVS的LIP，才读取PP协议头，否则有客户端伪造PP协议的风险（直连公网的服务器就不开启PP功能? 因为可直接获取CIP）
● 如果客户端伪造了PP协议，DPVS再添加一个PP协议头，请求可能失败，需要看后端应用程序的处理逻辑。

# 方案设计 

## Proxy Protocol格式

PP v1仅支持human-readable报头格式（ASCIII码）以及TCP协议。

v2需同时支持human-readable和二进制格式，即需要兼容v1格式，同时支持TCP、UDP协议。

下面通过PP V2来说明。
```c
union pp2_hdr {
    struct {
        char line[108];
    } v1; // proxy protocol version1
    struct {
        uint8_t sig[12]; // 固定的signature;\x0D\x0A\x0D\x0A\x00\x0D\x0A\x51\x55\x49\x54\x0A
            uint8_t cmd:4,              // 0:LOCAL, 1:PROXY
                    ver:4;              // 2:v2
            uint8_t proto:4,            // 0:UNSPEC, 1:STREAM, 2:DGRAM
                    af:4;               // 0:AF_UNSPEC, 1:AF_INET, 2:AF_INET6, 3:AF_UNIX
        uint16_t len;
        union {
            struct { /* for TCP/UDP over IPv4, len = 12 */
                uint32_t src_addr;
                uint32_t dst_addr;
                uint16_t src_port;
                uint16_t dst_port;
            } ip4;
            struct { /* for TCP/UDP over IPv6, len = 36 */
                uint8_t src_addr[16];
                uint8_t dst_addr[16];
                uint16_t src_port;
                uint16_t dst_port;
            } ip6;
            struct { /* for AF_UNIX sockets, len = 216 */
                uint8_t src_addr[108];
                uint8_t dst_addr[108];
            } unx;
        } addr;
    } v2; // proxy protocol version2
};

（1）PP V1说明
对于PP V1, 格式如下所示（"\r\n"为结束符）：
// PROXY AF L3_SADDR L3_DADDR L4_SADDR L4_DADDR\r\n
存在3种情况，如下：
PROXY TCP4 202.112.144.236 10.210.12.10 5678 80\r\n
PROXY TCP6 2001:da8:205::100 2400:89c0:2110:1::21 6324 80\r\n
PROXY UKNOWN\r\n
注：
第二个字段仅支持“TCP4”和“TCP6”，不支持UDP4和UDP6.
实际填充到TCP载荷中的PP数据，并不一定是108B，而是实际生成的PPV1的数据的长度；"\r\n"为PP数据的结束符。
```

## TCP协议支持PP
### PP插入时机
TCP的第一个数据包（数据包不能是不含有data的Pure ACK）。

### PP插入位置
紧跟着TCP头之后的载荷中。

对于IPv4，则插入的PP数据为28B（16B的头+12B的四元组）
对于IPv6，则插入的PP数据是52B（16B的头+36B的四元组）
注：插入后需要更改TCP Seq的差值，IP头中的 total len。

### 其他
#### 第一个数据包过大，插入PP之后，超过MTU的情况
发送单独的包含PP信息的TCP数据。

#### 第一个数据包有重传怎么办？
Client重传第一个数据包，DPVS对原始包以及后续的重传数据包插入PP数据。

#### 插入PP之前，原始数据包存在PP数据的情况
可基于VS的配置，决定是否替换掉原始包中的PP数据。
（1）替换
 替换掉原有的TCP载荷中的第一个PP数据。
（2）不替换
保持原有的TCP载荷中的PP数据。


## UDP协议支持PP


# 测试
## nginx支持
`nginx` 从`1.13.11`版本后，已支持`Proxy Protocol v2`，为配置支持`Proxy Protocol`，修改配置文件`/etc/nginx/nginx.conf`。

```c
http {
    #...
    server {
        listen 80 proxy_protocol;
        listen 443 ssl proxy_protocol;
        #...
    }
}
stream {
    #...
    server {
        listen 12345 proxy_protocol;
        #...
    }
}
```

nginx日志的配置为：

```c
http {
    #...
    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
        '$status $body_bytes_sent "$http_referer" '
        '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;
}
```

## curl 测试

curl从7.60.0版本后，支持在连接开始阶段发送`Proxy Protocol v1 header`，命令如下：
```bash
./curl --haproxy-protocol -v http://106.12.153.80/index.html
```

curl目前不支持发送Proxy Protocol v2 header，只能编写socket程序在TCP连接建立后立即发送`Proxy Protocol v2 header`。

## 自定义程序

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <stddef.h>
#include <stdint.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>
#include <netinet/tcp.h>

union pp2_hdr {
    struct {
        char line[108];
    } v1;
    struct {
        uint8_t sig[12];
        uint8_t ver_cmd;
        uint8_t fam;
        uint16_t len;
        union {
            struct { /* for TCP/UDP over IPv4, len = 12 */
                uint32_t src_addr;
                uint32_t dst_addr;
                uint16_t src_port;
                uint16_t dst_port;
            } ip4;
            struct { /* for TCP/UDP over IPv6, len = 36 */
                uint8_t src_addr[16];
                uint8_t dst_addr[16];
                uint16_t src_port;
                uint16_t dst_port;
            } ip6;
            struct { /* for AF_UNIX sockets, len = 216 */
                uint8_t src_addr[108];
                uint8_t dst_addr[108];
            } unx;
        } addr;
    } v2;
};

struct pp2_tlv {
    uint8_t type;
    uint8_t length_hi;
    uint8_t length_lo;
    uint8_t value[0];
};

static int build_pp2(uint8_t *buf)
{
    union pp2_hdr *hdr;
    uint8_t sig[12] = {0x0D, 0x0A, 0x0D, 0x0A, 0x00, 0x0D, 0x0A, 0x51, 0x55, 0x49, 0x54, 0x0A};

    memset(buf, 0, sizeof(buf));
    hdr = (union pp2_hdr *)buf;
    memcpy(hdr->v2.sig, sig, 12);
    hdr->v2.ver_cmd = 0x21;
    hdr->v2.fam = 0x11;
    hdr->v2.len = htons(sizeof(hdr->v2.addr.ip4));
    hdr->v2.addr.ip4.src_addr = htonl(0x0A000001); /* "10.0.0.1" */
    hdr->v2.addr.ip4.dst_addr = htonl(0xC0A80001); /* "192.168.0.1" */
    hdr->v2.addr.ip4.src_port = htons(1000);
    hdr->v2.addr.ip4.dst_port = htons(2000);
    return 16 + sizeof(hdr->v2.addr.ip4);
}

static int build_http(uint8_t *buf)
{
    uint8_t http_req[] = {
        0x47, 0x45, 0x54, 0x20, 0x2f, 0x69, 0x6e, 0x64, 0x65, 0x78, 0x2e,
        0x68, 0x74, 0x6d, 0x6c, 0x20,
        0x48, 0x54, 0x54, 0x50, 0x2f, 0x31, 0x2e, 0x31, 0x0d, 0x0a, 0x48,
        0x6f, 0x73, 0x74, 0x3a, 0x20,
        0x31, 0x30, 0x36, 0x2e, 0x31, 0x32, 0x2e, 0x31, 0x35, 0x33, 0x2e,
        0x38, 0x30, 0x0d, 0x0a, 0x55,
        0x73, 0x65, 0x72, 0x2d, 0x41, 0x67, 0x65, 0x6e, 0x74, 0x3a, 0x20,
        0x63, 0x75, 0x72, 0x6c, 0x2f,
        0x37, 0x2e, 0x36, 0x34, 0x2e, 0x30, 0x0d, 0x0a, 0x41, 0x63, 0x63,
        0x65, 0x70, 0x74, 0x3a, 0x20,
        0x2a, 0x2f, 0x2a, 0x0d, 0x0a, 0x0d, 0x0a
    }; /* HTTP GET */
    memcpy(buf, http_req, sizeof(http_req));
    return sizeof(http_req);
}

int main(int argc, char *argv[])
{
    int ret = -1, flag;
    int sockfd = 0, tx_size = 0, n = 0;
    uint8_t buf[1024];
    struct sockaddr_in serv_addr;

    if (argc != 2) {
        printf("\n Usage: %s <ip of server>\n", argv[0]);
        return -1;
    }
    memset(buf, 0, sizeof(buf));
    if ((sockfd = socket(AF_INET, SOCK_STREAM, 0)) < 0) {
        perror("socket");
        return -1;
    }
    flag = 1;
    if (setsockopt(sockfd, IPPROTO_TCP, TCP_NODELAY, (char *)&flag, sizeof(flag)) < 0) {
        perror("setsockopt");
        goto out;
    }
    memset(&serv_addr, 0, sizeof(serv_addr));
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(80);
    if (inet_pton(AF_INET, argv[1], &serv_addr.sin_addr) <= 0) {
        perror("inet_pton");
        goto out;
    }
    if (connect(sockfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr)) < 0) {
        perror("connect");
        goto out;
    }
    tx_size = build_pp2(buf);
    if ((n = send(sockfd, buf, tx_size, 0)) < 0) {
        perror("send");
        goto out;
    } else if (n != tx_size) {
        printf("Part of PP2 header sent!\n");
        goto out;
    }
    tx_size = build_http(buf);
    if ((n = send(sockfd, buf, tx_size, 0)) < 0) {
        perror("send");
        goto out;
    } else if (n != tx_size) {
        printf("Part of http sent!\n");
        goto out;
    }
    if ((n = read(sockfd, buf, sizeof(buf) - 1, 0)) > 0) {
        buf[n] = 0;
        printf("size: %d\n", n);
        printf("%s", buf);
    } else if (n < 0) {
        perror("read");
        goto out;
    }
    ret = 0;

out:
    close(sockfd);
    return ret;
}
```

# pp 和 toa/uoa对比

![](attachments/image%20(11).png)

# 参考
```bash
## Client Address Conservation in Fullnat
https://github.com/iqiyi/dpvs/blob/master/doc/client-address-conservation-in-fullnat.md
  
# L4LB网络中间件DPVS在FNAT TOA模式存在IP伪造漏洞
https://www.ctfiot.com/149666.html
```