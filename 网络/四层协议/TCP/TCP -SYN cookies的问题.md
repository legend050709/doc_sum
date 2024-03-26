```table-of-contents
```
# Linux TCP -SYN cookies无法检测特定的首包丢弃
## 背景
TCP我们都知道链接可靠性传输保证发送端所有的 字节都以相同的顺序被接收端接收，没有数据丢失，也不会乱序。但是Linux实现TCP syn cookie有一个缺陷。
具体表现是：客户端连续并发发送两个数据包，在特定场景下「在三次握手的ack以及第一个小包(3B以内)丢失或者晚到」，服务端TCP堆栈将第二个数据包中的数据传送给应用程序，而不知道这不是流中的第一个数据包。此时，客户端和服务端都认为正常。

## 问题总结
当以下几种条件都成立，就会发生这种问题：
- 服务端：SYN cookie已启用
- 客户端：先传输数据包。client作为三次握手最后的ACK 以及 第一个数据包很小（小于3bytes）没有被传递到服务端（中间丢失或晚到服务端）。
- 客户端：在对第一个数据包进行任何重传之前，客户端发送的第二个数据包被服务端接收，并回复了正确的ACK。
> 注：由于对第二个数据包回复了正确的ACK，此时client不会重传第一个数据包。

![](attachments/Pasted%20image%2020240324153704.png)

**结果就是客户端发了"dog"和"cat"，但是服务端只收到了“cat”，两边都不知道有什么丢失了，两端都认为传输正常**。

## 分析
在SYN cookie条件下，当服务器接收到客户端的第一个ACK以完成连接的建立时，关键是服务器没有半建立连接的记录。它必须仅使用客户端的ACK数据包中的信息重新创建连接状态，即端点的地址和端口号、客户端的初始序列号、其自身的初始序列编号和最大段大小。

服务器可以推断它自己的初始序列号是什么（SYN-ACK包的Seq序列号）。它是客户端的ACK数据包的确认号减去一，这是TCP所要求的。服务器特别选择了这一点，因此它也在其中编码了（近似的）MSS和低分辨率时间戳（以防止重放攻击）

服务器还可以推断客户端的初始序列号(Syn包的Seq序列号)是什么——它是ACK数据包的序列号，减去1。

如果客户端的ACK数据包从未送达怎么办？此外，如果包含3字节消息“dog”的客户端的第一个数据包也丢失了，该怎么办？服务器不会注意到任何问题。在它从客户端接收到一些东西之前，它甚至没有任何连接记录。

通常情况下，客户端会注意到服务器在合理的时间内没有确认“dog”，并重新发送它。但假设早在发生这种情况之前，客户端应用程序就进行了第二次send（）调用，以发送“cat”。客户端以新的数据包发送它。

Linux内核中的相关函数是`secure_tcp_syn_cookie（）`：
```c
static __u32 secure_tcp_syn_cookie(__be32 saddr, __be32 daddr, __be16 sport,
                                   __be16 dport, __u32 sseq, __u32 data)
{
        /*
         * Compute the secure sequence number.
         * The output should be:
         *   HASH(sec1,saddr,sport,daddr,dport,sec1) + sseq + (count * 2^24)
         *      + (HASH(sec2,saddr,sport,daddr,dport,count,sec2) % 2^24).
         * Where sseq is their sequence number and count increases every
         * minute by 1.
         * As an extra hack, we add a small "data" value that encodes the
         * MSS into the second hash value.
         */
        u32 count = tcp_cookie_time();
        return (cookie_hash(saddr, daddr, sport, dport, 0, 0) +
                sseq + (count << COOKIEBITS) +
                ((cookie_hash(saddr, daddr, sport, dport, count, 1) + data)
                 & COOKIEMASK));
}
```

在Linux上，SYN cookie是通过将时间戳的最后八位放在cookie的前八位中（count＜COOKIEBITS），然后添加一个哈希值（连接的常量），另一个哈希值更低的24位（取决于时间戳），还添加客户端的初始序列号（sseq），最后添加数据「它是一个介于0和3之间的数字，告诉我们将使用msstab中的哪个值的`index`下标作为最大段大小（MSS）」而形成的。**正是这种将数据与客户端的初始序列号一起压缩到cookie中的尝试导致了我们的问题。**


当服务器从客户端获得第一个ACK时，它必须检查确认数字减1是否看起来像SYN cookie。它使用的函数是`check_tcp_syn_cookie（）`：
```c
/*
 * This retrieves the small "data" value from the syncookie.
 * If the syncookie is bad, the data returned will be out of
 * range.  This must be checked by the caller.
 *
 * The count value used to generate the cookie must be less than
 * MAX_SYNCOOKIE_AGE minutes in the past.
 * The return value (__u32)-1 if this test fails.
 */
static __u32 check_tcp_syn_cookie(__u32 cookie, __be32 saddr, __be32 daddr,
                                  __be16 sport, __be16 dport, __u32 sseq)
{
        u32 diff, count = tcp_cookie_time();

        /* Strip away the layers from the cookie */
        cookie -= cookie_hash(saddr, daddr, sport, dport, 0, 0) + sseq;

        /* Cookie is now reduced to (count * 2^24) ^ (hash % 2^24) */
        diff = (count - (cookie >> COOKIEBITS)) & ((__u32) -1 >> COOKIEBITS);
        if (diff >= MAX_SYNCOOKIE_AGE)
                return (__u32)-1;

        return (cookie -
                cookie_hash(saddr, daddr, sport, dport, count - diff, 1))
                & COOKIEMASK;   /* Leaving the data behind */
}
```

该函数采用我们所谓的SYN cookie，减去第一个哈希值和它假设的客户端的初始序列号（sseq，通过从该数据包的序列号中减去1得出），将时间戳保留在前8位，其余保留在后24位。它检查时间戳是否在当前时间戳的小增量内。如果是这样，它只考虑较低的24位，并减去第二个散列，以留下MSS索引值（数据），该值在0和3之间。该值通常为3，表示msstab表（1460）中的最高MSS。

由调用者检查返回的值是否是msstab的有效索引，如果不是，则不认为它是有效的SYN cookie，我们重置连接。当然，如果这个ACK数据包的序列号与实际的初始序列号相差甚远，我们不会返回0到3之间的数字，这将导致服务器发送RST数据包来拒绝连接。

## 原因总结
如下所示：
![](attachments/Pasted%20image%2020240324201532.png)
客户端发送一个SYN，服务器发送一个带有SYN cookie的SYN+ACK。来自三方握手的ACK丢失，第一个数据包（“dog”）也丢失。客户端的第二个数据包（“cat”）是自SYN发送到服务器以来的第一个数据包。它的序列号为`clients_initial_sequence_number+4`（ACK的序列号中的1比SYN多1，已经发送的三个字节中的3）。

在这种情况下，sseq比我们预期的多出三个。当`check_tcp_syn_cookie（）`从cookie中减去它时，cookie最终比它应该少三个。它没有返回3，而是返回0。这仍然是msstab的有效索引，因此使用低于所需的MSS接受连接，流中的前三个字节丢失，并且没有系统调用指示错误。

总之，数据包的序列号与我们预期的序列号相比存在差异，应该提醒我们这不是第一个数据包，但被误认为是比通常的MSS值小。

# DPVS中的syn-cookie的问题
## DPVS中的syn-cookie校验只和ack包有关系
由于`syn-cookie` 就是为了去除状态的记录。
正常情况下，收到 `syn`包，回复`syn-ack`包，此时没有任何记录信息或者状态或者会话，等到收到三次握手的 `ack`包时，**仅仅基于ack包中的信息**(`seq_num、ack_num、五元组信息等等`)检查是否符合`cookie`检查。正常情况下，如果是`client`的内核回复的，而不是攻击者构建的`ack`包，正常都是可以检测通过的。

如果是攻击者自己构建的`ack`包，甚至攻击者都没有发送过`syn`包，`dpvs`也没回复`syn-ack`，攻击者仅仅发送一个`ack`包, 理论上也是至少有`1/4G`（`ack_num为32bit`）的概率可以通过`cookie`检查，进而构建`conn`会话。

即：**收到 `ack`包时，仅仅基于这个`ACK`包进行`cookie` 校验，不依赖之前的状态或者会话**。其实是不知道这个`ack`是不是三次握手的`ack`，也不知道之前是否收到了`syn`，以及是否回复了`syn-ack`。

## conn-reuse场景下无法识别ack是特定的四次挥手的最后ack和新的三次握手的ack
**（1）期望状态**
如果是 四次挥手的ack包，在收到此包之前的conn应该是已经处于`CLOSE_WAIT`或者`FIN_WAIT`或者`LAST_ACK`的状态，期望的状态是直接将收到的ACK 转发给 后端的RS，关闭连接就可以了。

如果是新的三次握手的ack，新的三次握手的报文又是复用了之前关闭的五元组，但是`DPVS`中的老的 `Conn`还没有删除，收到 新的三次握手的`ack`，此时可以查找到 `conn`，接下里就不用新建`conn`了，直接更改`conn`的状态，构建`syn`报文，发送给后端的`RS`。

**(2) 问题总结**
![](attachments/image%20(5).png)
开启SynPorxy的情况下，**Syn-cookie校验，无法区分特定流量模型的四次挥手的ACK报文和TCP连接复用下的新的三次握手的ACK报文**。
可能将特定流量模型的四次挥手的ACK报文认为是连接复用下的新的三次握手的ACK报文，进而重新给后端RS发送Syn，建立了连接。

注：三次握手之后，即可四次挥手的探测，就属于上诉中的特定流量模型。包括连接建立到关闭期间，收发的字节数一样，也是这种特定流量模型。

## 用户的请求时延变高问题
`LB`中的 `syn-proxy`是使用 `syn-cookie` 实现的;
`VS` 开启了 `syn-proxy`后，正常情况下，用户的`HTTP`请求的耗时会增加一个`RTT`。
![](attachments/image%20(4).png)
# 参考
```bash
# Linux TCP -SYN cookies的一个问题分析
https://zhuanlan.zhihu.com/p/640934692
```