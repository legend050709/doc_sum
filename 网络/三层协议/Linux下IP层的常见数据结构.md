# 关系
![](attachments/Pasted%20image%2020230605152131.png)

其实`sockaddr` 和 `sockaddr_in` 之间的转化很容易理解，因为他们开头一样，内存大小也一样，但是`sockaddr`和`sockaddr_in6`之间的转换就有点让人搞不懂了，
其实你有可能被结构所占的内存迷惑了，这几个结构在作为参数时基本上都是以指针的形式传入的，我们拿函数`bind()`为例。
这个函数一共接收三个参数，第一个为监听的文件描述符，第二个参数是sockaddr*类型，第三个参数是传入指针原结构的内存大小，所以有了后两个信息，无所谓原结构怎么变化，因为他们的头都是一样的，也就是uint16 sa_family，那么我们也能根据这个头做处理，原本我没有看过bind()函数的源代码，但是可以猜一下:
```c
int bind(int socket_fd, sockaddr* p_addr, int add_size)
{
    if (p_addr->sa_family == AF_INET)
    {
        sockaddr_in* p_addr_in = (sockaddr_in*)p_addr;
        //...
    }
    else if (p_addr->sa_family == AF_INET6)
    {
        sockaddr_in6* p_addr_in = (sockaddr_in6*)p_addr;
        //...
    }
    else
    {
        //...
    }
}
```
- 小结
>- 通过等价替换的方式我们可以更好的了解sockaddr、sockaddr_in、sockaddr_in6之间的异同。
>- 网路接口函数针对于IPv4和IPv6虽然有不同的结构(struct sockaddr_in 和 struct sockaddr_in6)，但是接口基本相同(在使用地址的时候需要强转为struct sockaddr*结 构)，主要是为了用户（开发者）使用方便吧。


# 数据结构
## ipv4
```c
   typedef unsigned short int sa_family_t;

   struct sockaddr_in {
	   sa_family_t    sin_family; /* address family: AF_INET */
	   in_port_t      sin_port;   /* port in network byte order */
	   struct in_addr sin_addr;   /* internet address */

       /* Pad to size of `struct sockaddr'.  */
       unsigned char sin_zero[sizeof (struct sockaddr) -
                              sizeof (sa_family_t) -
                              sizeof (in_port_t) -
                             sizeof (struct in_addr)];
   };
   
    typedef uint32_t in_addr_t;
    struct in_addr  {
        in_addr_t s_addr;                    /* IPv4 address */
    };
```
## ipv6
```c
   struct sockaddr_in6 {
	   sa_family_t     sin6_family;   /* AF_INET6 */
	   in_port_t       sin6_port;     /* port number */
	   uint32_t        sin6_flowinfo; /* IPv6 flow information */
	   struct in6_addr sin6_addr;     /* IPv6 address */
	   uint32_t        sin6_scope_id; /* Scope ID (new in 2.4) */
   };

	 struct in6_addr {
		 union {
			 uint8_t u6_addr8[16];
			 uint16_t u6_addr16[8];
			 uint32_t u6_addr32[4];
		 } in6_u;
	 
		 #define s6_addr                 in6_u.u6_addr8
		 #define s6_addr16               in6_u.u6_addr16
		 #define s6_addr32               in6_u.u6_addr32
	 };

```
## unix socket
```c
   struct sockaddr_un {
	   sa_family_t sun_family;               /* AF_UNIX */
	   char        sun_path[108];            /* Pathname */
   };
```
## 通用
通用结构体1: struct sockaddr, 16个字节
```c

struct sockaddr {
  sa_family_t sa_family;  /* address family, AF_xxx */
  char    sa_data[14];  /* 14 bytes of protocol address */
};

```

 通用结构体2: struct sockaddr_storage,128个字节
```c
/* Structure large enough to hold any socket address (with the historical exception of AF_UNIX). 128 bytes reserved. */

#if ULONG_MAX > 0xffffffff
    # define __ss_aligntype __uint64_t
#else
    # define __ss_aligntype __uint32_t
#endif
#define _SS_SIZE        128
#define _SS_PADSIZE     (_SS_SIZE - (2 * sizeof (__ss_aligntype)))
struct sockaddr_storage
{
    sa_family_t     ss_family;      /* Address family */
    __ss_aligntype  __ss_align;  /* Force desired alignment.  */
    char            __ss_padding[_SS_PADSIZE];
};

```

## 其他
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
```

# 参考
```c
https://blog.csdn.net/weixin_44874963/article/details/89395292
```