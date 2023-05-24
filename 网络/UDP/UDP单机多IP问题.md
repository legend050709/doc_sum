# 背景
# 分析
# 原理
# 解决
# 范例
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