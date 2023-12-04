```table-of-contents
```

# socket的API接口
```c
int getsockopt(int sockfd, int level, int optname,
              void *optval, socklen_t *optlen);
              
int setsockopt(int sockfd, int level, int optname,
              const void *optval, socklen_t optlen);

```
## 参数说明
- **level** ：
通过level这个参数，使用统一个接口函数能够设置不同协议的属性。
当设置为SOL_SOCKET时表示的是socket通用的属性，那么不需要涉及具体的协议，可以直接搞定。
如果涉及到具体的协议，如TCP/UDP则需要指定level为对应的值。
一般来说，这么几种类型level：**SOL_SOCKET, SOL_TCP, SOL_UDP，SOL_RAW，IPPROTO_IP， IPPROTO_IPV6**。
```c
int kernel_setsockopt(struct socket *sock, int level, int optname,
            char *optval, unsigned int optlen)
{
    mm_segment_t oldfs = get_fs();
    char __user *uoptval;
    int err;

    uoptval = (char __user __force *) optval;

    set_fs(KERNEL_DS);
    if (level == SOL_SOCKET)
        err = sock_setsockopt(sock, level, optname, uoptval, optlen);
    else
        err = sock->ops->setsockopt(sock, level, optname, uoptval,
                        optlen);
    set_fs(oldfs);
    return err;
}
```
- optname：需设置的选项
> 有部分选项需在listen/connect调用前设置才有效，这部分选项如下：SO_DEBUG、SO_DONTROUTE、SO_KEEPALIVE、SO_LINGER、SO_OOBINLINE、SO_RCVBUF、SO_RCVLOWAT、SO_SNDBUF、SO_SNDLOWAT、TCP_MAXSEG、TCP_NODELAY。

- optval：指针，指向存放选项值的缓冲区
- optlen：optval缓冲区长度


## 不同协议对应的setsockopt函数
sock->ops->setsockopt 在inet_init函数中被初始化。
![](attachments/Pasted%20image%2020231115141943.png)
- **tcp protocol**
来看看tcp_proto全局变量：
![](attachments/Pasted%20image%2020231115142019.png)
在socket被创建时（即用户态使用socket系统调用时），内核就把就判断用户创建的socket类型，是tcp还是udp还是raw，然后把相应的指针函数，赋值成相应协议的函数。

**当我们用户态level不是SOL_SOCKET，如果是tcp的socket，那么sock->ops->setsockopt 就是tcp_setsockopt。**
```c
int tcp_setsockopt(struct sock *sk, int level, int optname, char __user *optval,
         unsigned int optlen)
{
    struct inet_connection_sock *icsk = inet_csk(sk);

    if (level != SOL_TCP)
        return icsk->icsk_af_ops->setsockopt(sk, level, optname,
                         optval, optlen);
    return do_tcp_setsockopt(sk, level, optname, optval, optlen);
}
```
我们看到内核再次对level作出“筛选”，如果level 是 SOL_TCP，那么ok，直接调用tcp 的 `setsockopt`函数。  
如果level!=SOL_TCP，那么level 就是`IPPROTO_IP` or `IPPROTO_IPV6`
，就会调用`icsk->icsk_af_ops->setsockopt`。  
那么，`icsk->icsk_af_ops->setsockopt`又是什么函数呢？  
其实`icsk->icsk_af_ops->setsockopt`是在`inet_create` or `inet6_create`
函数中被初始化的，也就是当你新建一个socket的时候，就被初始化了：

下面以`inet_create` 为例：
```c
static int inet_create(struct net *net, struct socket *sock, int protocol)
{
    ...................
    if (sk->sk_prot->init) {
    err = sk->sk_prot->init(sk);
    if (err)
        sk_common_release(sk);
    }
    out:
    return err;
    out_rcu_unlock:
    rcu_read_unlock();
    goto out;

}
```
 `sk->sk_prot->init(sk)`` 这个函数，init指针其实就是之前我们在`tcp_proto` 全局变量中，看到的 init字段，被初始化为：`tcp_v4_init_sock()`。
 
## 小结
所以用户态的setsockopt函数。最终根据你给出的level以及相应的协议类型，会调用下面的其中一个函数：  
```c
sock_setsockopt  
udp_setsockopt  
tcp_setsockopt  
ip_setsockopt
ipv6_setsockopt
```


# socket level 分层
## socket层
## tcp/udp层
## ip层
### IPv4层
### Ipv6 层
## raw socket的opt
# 范例
## IP_BIND_ADDRESS_NO_PORT
![](attachments/Pasted%20image%2020231115135400.png)
如上所示，ipv4 socket 以及 ipv6 scoket 在进行setsockopt 设置BIND_ADDRESS_NO_PORT 的时候，level 都是 SOL_IP.
参考：[linux IP_BIND_ADDRESS_NO_PORT patch](https://patchwork.ozlabs.org/project/netdev/patch/1433650677.29864.26.camel@edumazet-glaptop2.roam.corp.google.com/)
## IP_DONTFRAGMENT
```c
SOCKET sock = socket(PF_INET, SOCK_DGRAM, 0);
int dontfragment = 1;
if (setsockopt(sock,IPPROTO_IP,IP_DONTFRAGMENT,&dontfragment,sizeof(dontfragment)) == -1) {
    printf("set failed \r\n");
}
```
# 参考
```c
# 深入理解Linux socket
https://blog.csdn.net/lianhunqianr1/article/details/119621750?ops_request_misc=%257B%2522request%255Fid%2522%253A%2522170003006116800215015528%2522%252C%2522scm%2522%253A%252220140713.130102334.pc%255Fblog.%2522%257D&request_id=170003006116800215015528&biz_id=0&utm_medium=distribute.pc_search_result.none-task-blog-2~blog~first_rank_ecpm_v1~rank_v31_ecpm-1-119621750-null-null.nonecase&utm_term=socket&spm=1018.2226.3001.4450

[setsockopt 内核实现]
(http://blog.chinaunix.net/uid-24857907-id-4217438.html)

# setsockopt的常用选项
https://www.imooc.com/article/290578
```