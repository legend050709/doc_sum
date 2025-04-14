```table-of-contents
```
# 背景
TCP虽然能保证传输的可靠性，但其繁琐的状态机以及复杂的拥塞控制机制，丢包重传、超时重传机制，让它难以作为隧道报文的外层封装，详见[TCP-in-TCP](https://segmentfault.com/a/1190000020427497)。

相对而言，UDP就没这个困扰了，丢包的事情交给隧道内层协议或者内层协议的应用层处理就行。因而，不少隧道协议都是将UDP作为外层报文的方案。自然而然，与网络发展联系紧密的Linux内核也开始支持这些隧道协议，较新的内核已经支持fou、l2tp、vxlan、tipc、geneve等UDP隧道协议。

最开始，各个隧道协议都是独立实现的，但随着数量的增多，在[patch](https://link.segmentfault.com/?enc=e%2BS3shoUyGVh75pthPqOyw%3D%3D.SbYu5e%2Fq3%2B3WeDnW%2BatUTqIMq3S6UBbThUfWOgdjdIOfzY3lPphJV9nTLwkKSBHaVfdv%2ByvrfW0%2FvPKz5vaP8GUDQ%2FZ3z0y5G%2B9s1ZnOOku4qncWBj%2F8BKFIm5s5dY%2FQ2J4nQfVOTlH59z2yBaXS0SJ6w56beJNP4n8F%2FrPkZ2iA8SqN1YhFR3OM08A4AGrs)之后，内核将这些UDP隧道公共的部分抽离出来，也就形成了UDP隧道框架，其涉及的API在`include/net/udp_tunnel.h`中定义

# 内核UDP隧道的原理
下图以vxlan为例，展示了内核UDP隧道的工作过程：
![](attachments/Pasted%20image%2020231128175314.png)
其中，左边是发送端，右边是接收端，绿色阴影的部分是内核协议栈。可以看出，无论是发送端还是接收端，都涉及**函数重入**：发送端两次进入`ip_local_out()`, 接收端两次进入`ip_local_deliver()`

## 隧道socket
对发送端来说，第一次进入`ip_local_out()`传入的`sk`是与原始报文关联的套接字，也就是原始协议的套接字，它可能是个TCP套接字，也可能是UDP套接字或者RAWIP套接字，隧道并不care这件事。但是第二次进入`ip_local_out()`时，它需要一个隧道的UDP套接字。UDP隧道框架提供了一个创建隧道套接字的API。
```c
static inline int udp_sock_create(struct net *net,
                  struct udp_port_cfg *cfg,
                  struct socket **sockp)
{
    if (cfg->family == AF_INET)
        return udp_sock_create4(net, cfg, sockp);

    ......
    return -EPFNOSUPPORT;
}

int udp_sock_create4(struct net *net, struct udp_port_cfg *cfg,
             struct socket **sockp)
{
    int err;
    struct socket *sock = NULL;
    struct sockaddr_in udp_addr;

    err = sock_create_kern(net, AF_INET, SOCK_DGRAM, 0, &sock);
    if (err < 0)
        goto error;

    udp_addr.sin_family = AF_INET;
    udp_addr.sin_addr = cfg->local_ip;
    udp_addr.sin_port = cfg->local_udp_port;
    err = kernel_bind(sock, (struct sockaddr *)&udp_addr,
              sizeof(udp_addr));
    if (err < 0)
        goto error;

    if (cfg->peer_udp_port) {
        udp_addr.sin_family = AF_INET;
        udp_addr.sin_addr = cfg->peer_ip;
        udp_addr.sin_port = cfg->peer_udp_port;
        err = kernel_connect(sock, (struct sockaddr *)&udp_addr,
                     sizeof(udp_addr), 0);
        if (err < 0)
            goto error;
    }

    sock->sk->sk_no_check_tx = !cfg->use_udp_checksums;

    *sockp = sock;
    return 0;

error:
    if (sock) {
        kernel_sock_shutdown(sock, SHUT_RDWR);
        sock_release(sock);
    }
    *sockp = NULL;
    return err;
}
```

`cfg`参数指定了UDP隧道本端和对端和IP地址和使用的端口号。这里创建的套接字都是**内核套接字**(区别于用户态使用socket()创建的)

接收端也是同样的道理，从真实网卡收到的一定是一个UDP报文，因此接收端也需要一个UDP套接字。这个套接字中记录的地址和端口信息与接收端正好相反。
![](attachments/Pasted%20image%2020231128175737.png)

## Encap rcv回调函数
对接收端来说，收到UDP报文后，它还需要将报文找到分流给正确的隧道协议，比如这个UDP隧道报文是交给vxlan，还是交给geneve？又或者这根本就只是一个普通的UDP报文，不是一个UDP隧道报文？
因此，内核需要将如何分流记录在UDP套接字上。
```c
struct udp_sock {
    ......
    /*
     * For encapsulation sockets.
     */
    int (*encap_rcv)(struct sock *sk, struct sk_buff *skb);
    ......
};
```
这里的`encap_rcv`回调函数便是起到隧道报文分流的作用

在UDP接收时，内核会首先查看套接字上是否设置了该回调函数，如果设置了，表示这是一个隧道套接字。调用对应的处理函数，比如vxlan隧道会将其设置为`vxlan_rcv`,genven隧道会将其设置为`geneve_udp_encap_recv`。

设置的过程是通过下面这个API完成的：
```c
void setup_udp_tunnel_sock(struct net *net, struct socket *sock,
               struct udp_tunnel_sock_cfg *cfg)
```

# 参考
```c
https://segmentfault.com/a/1190000020458221
```