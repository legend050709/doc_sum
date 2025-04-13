```table-of-contents
```
# 内核 socket lookup 逻辑
内核中的 socket lookup 逻辑，也就是当 TCP 层收到一个包时， **==如何判断这个包属于哪个 socket==**。
逻辑其实非常简单： 两阶段， 先精确匹配，再模糊匹配：
![](attachments/Pasted%20image%2020231120142513.png)
![](attachments/Pasted%20image%2020231120142521.png)
![](attachments/Pasted%20image%2020231121200645.png)
1. 首先是 `(src_ip,src_port,dst_ip,dst_port)` 4-tuple 精确匹配，看能不能找到 **connected 状态的 socket**；如果找不到，
2. 再尝试 `(dst_ip,dst_port)` 2-tuple，寻找有没有 **listening 状态的 socket**；如果还是没找到，
3. 再尝试 `(INADDR_ANY)` 1-tuple，寻找有没有 **listening 状态的 socket**。

# TCP 连接建立的细节
## 前置知识
TCP 用四元组来区分不同的连接。
对内核来说，它还会将 sock 套接字按照 TCP 状态放在两个独立的 hash 表中。 而当内核 TCP 收到一个报文，它总会首先在搜索是否有匹配的 ESTABLISHED 状态的 sock，如果没有，再去搜索 LISTEN 状态的。
```c
struct inet_hashinfo tcp_hashinfo    
    +-------------------+               struct inet_ehash_bucket
    |                   |         +-------->+----------+
    +-------------------+         |         |   [0]    |----->+---------+--->+---------+---->
    |     ehash         |---------+         +----------+      | sock #1 |    | sock #2 |
    +-------------------+                   |   [1]    |      +---------+    +---------+
    |     ......        |                   +----------+
    +-------------------+                   |   [2]    |----->+---------+
    |                   |                   +----------+      | sock #3 | 
    +-------------------+                                     +---------+
    |                   +
    +-------------------+               struct inet_listen_hashbucket
    |  listening_hash   |------------------>+----------+        
    +-------------------|                   |   [0]    |----->+---------+--->+---------+---->
    |    ......         |                   +----------+      | sock #4 |    | sock #5 |
    +-------------------+                   |   [1]    |      +---------+    +---------+
                                            +----------+
                                            |   [2]    |
                                            +----------+
```
![](attachments/Pasted%20image%2020231121202438.png)

## TCP 连接的建立步骤
- **不开启 SYN-Cookie的建立步骤**
一个普通的 TCP 连接的建立步骤 (不使用 SYN-Cookie)：
1. 收到一个 SYN 报文，它搜索 ESTABLISHED 表, 没有找到匹配的 sock, 然后搜索 LISTEN 表，找到了一个匹配的 listen sock (状态为 LISTEN)
2. 创建一个 request sock ，插入 ESTABLISHED 表, 回复 SYNACK 报文，此时 request sock 的状态为 (NEW_SYN_RECV)
3. 收到第三次握手的 ACK 报文，从 ESTABLISHED 表找到了 request sock，创建新的 child sock, 加入 ESTABLISHED 表 (状态为 ESTABLISHED)，删除 request sock


- **开启 SYN-Cookie的 TCP 连接的建立步骤**
1. 收到一个 SYN 报文，它搜索 ESTABLISHED 表, 没有找到匹配的 sock, 然后搜索 LISTEN 表，找到了一个匹配的 listen sock (状态为 LISTEN)
2. 回复一个特别的 SYNACK (序列号经过精心计算)，本地不创建任何资源(request sock)
3. 收到第三次握手的 ACK 报文，搜索 ESTABLISHED 表，找不到，然后搜索 LISTEN 表，找到 listen sock 进行 SYN-Coookie 检查，检查通过后，创建 child sock，加入 ESTABLISHED 表 (状态为 ESTABLISHED)。


# listen套接字查找的演变
## 4.17版本以前的listen套接字查找
> The current listener hashtable is hashed by port only. When a process is listening at many IP addresses with the same port (e.g.[IP1]:443, [IP2]:443… [IPN]:443), the inet[6]_lookup_listener() performance is degraded to a link list. It is prone to syn attack.

4.17版本之前，TCP的listener socket是按`port`进行hash，然后插入到对应的冲突链表中的。这就使得如果很多个listen套接字都侦听同一个port，就会使得链表拉得比较长, 这种情况在3.9版本引入`REUSEPORT`之后更加严重。
![](attachments/Pasted%20image%2020231121200831.png)
举个栗子，主机上启动了6个listener,它们都侦听21端口，因此被放到同一条链表上(其中`sk_B`使用了`REUSEPORT`)。如果此时收到一个目标位`1.1.1.4:21`的SYN连接请求,内核在查找listenr的时候，始终会从头开始遍历到尾，直到找到匹配的`sk_D`。

## 4.17版本：在两个hashtable中查找listen套接字
4.17版本增加了一个新的hashtable(`lhash2`)来组织listen套接字，这个`lhash2`是按`port+addr`作为key进行hash的，而原来按`port`进行hash的hashtable保持不变。换句话说，同一个listen套接字会同时放到两个hashtable中(例外情况是，如果它绑定的本地地址是0.0.0.0,则只会放到原来的hashtable中)

`lhash2`增加了addr作为key，也就增加hash的随机性。还是以上面的例子为例，此时，原来的`sk_A~C`可能就被hash到其他冲突链了,当然与此同时，也有可能有原来在其他冲突链上的`sk_E`被hash到`lhash2[0]`这条冲突链。
![](attachments/Pasted%20image%2020231121201013.png)
因此在listen套接字的查找时，内核会根据SYN报文中的`port+addr`，同时计算出满足条件的套接字应该在两个hashtable中所属的链表，然后比较这两个链表的长度，如果在1st链表长度不长或者小于2nd链表的长度，则还是以原来的方式，在1st链表中进行查找，否则就在2nd链表中进行查找。

## 5.0版本：只在2nd hashtable中查找listen套接字
内核在5.0版本又将查找方式改为了只在2nd hashtable中进行查找。这样修改的原因是按原来的查找方式，如果选择了在1st hashtable中进行查找，可能发生在通配地址(0.0.0.0)和特定地址(比如1.1.1.1)都侦听同一个`Port`时，反而匹配上通配地址的listener的问题。这其实不是4.17版本的锅，而是在3.9版本引入`SO_PORTREUSE`就已经存在了！

来看看怎么回事：
![](attachments/Pasted%20image%2020231121201133.png)
设置了`SO_REUSEPORT`的`sk_A`和`sk_B`同时侦听21端口，如果`sk_A`是后启动，那么它将添加到链表头，这样当收到一个`1.1.1.2:21`的报文时，内核会发现`sk_A`就已经匹配了，它就不会再去尝试匹配更精确的`sk_B`！这显然不好，要知道在`SO_REUSEPORT`进入内核之前，内核会遍历整个链表，对每个套接字进行匹配程度打分(`compute_score`)。

5.0版本修改为只在2nd hashtable中进行查找，并且修改了`compute_score`的实现方式，如果侦听地址与报文的目的地址不相同，则直接算匹配失败。而在之前，通配地址是可以直接通过这项检查的。

# 参考
```c
# [译] Socket listen 多地址需求与 SK_LOOKUP BPF 的诞生
https://arthurchiao.art/blog/birth-of-sk-lookup-bpf-zh/#12-%E9%9C%80%E6%B1%82%E5%A6%82%E4%BD%95%E8%AE%A9%E4%B8%80%E4%B8%AA%E6%9C%8D%E5%8A%A1%E7%9B%91%E5%90%AC%E8%87%B3%E5%B0%91%E5%87%A0%E7%99%BE%E4%B8%AA-ip-%E5%9C%B0%E5%9D%80


# TCP listen套接字的查找的变化
https://switch-router.gitee.io/blog/tcp-listener/

# 内核一个 IPv6 socket 的插入顺序修改引入的 bug
https://switch-router.gitee.io/blog/ipv6-sock/
```