```table-of-contents
```
# 内核tcp序列号的生成
看一下内核中对这个报文序列号的选择方式：
```bash
tcp_v4_rcv-->>tcp_v4_do_rcv-->>tcp_rcv_state_process-->>icsk->icsk_af_ops->conn_request-->>tcp_v4_conn_request-->>tcp_v4_init_sequence-->>secure_tcp_sequence_number
```
# tcp序列号 回绕问题
我们知道 TCP 的 sequence number 序列号是不断根据发送数据包的长度不断 递增的，如下图：
![](attachments/Pasted%20image%2020240327135349.png)

从上图可以看出， TCP 的 sequence number 字段，只用了 4 个字节来存储，也就是说 sequence number 是一个 32位的数字，最大值 2^32 = 4294967296(4G)。
这是不是意味着 TCP 最多只能传 4294967296 字节的数据？？
不是的，TCP 针对这个问题做了回环（wraparound）处理。

## 解决方法
主要有两种措施用于解决序列号回绕问题：
- 限制TCP窗口大小
- 时间戳机制
### 限制TCP窗口大小
处理回绕问题的关键在于：
在回绕发生时，**如何判断两个序列号的先后关系**。

#### 内核中判断两个序列号的先后关系
在内核中，判断先后关系的代码如下：(代码在Linux源码的`include/net/tcp.h`)

```c
/*
 * The next routines deal with comparing 32 bit unsigned ints
 * and worry about wraparound (automatic with unsigned arithmetic).
 */

static inline bool before(__u32 seq1, __u32 seq2)
{
        return (__s32)(seq1-seq2) < 0;
}
#define after(seq2, seq1)   before(seq1, seq2)

/* is s2<=s1<=s3 ? */
static inline bool between(__u32 seq1, __u32 seq2, __u32 seq3)
{
    return seq3 - seq2 >= seq1 - seq2;
}
```

可以看到，内核中判断两个序列号的先后关系实现得非常简单，相减然后强转成有符号32位整数，然后判断是否小于0即可。

通过一个例子来看这个方法的正确性，TCP序列号是一个32位无符号整数，范围是`[0, 4294967295]`，现在假设要比较两个序列号：

##### 测试一
```bash
seq1 = 4294967295
seq2 = 4294967296，超过了可表示的范围，回绕为0

分析：
调用before(seq1, seq2)时，seq1-seq2的结果是4294967295，然后强制转换成有符号整数，4294967295 转换成了-1，小于0，于是函数输出seq1在seq2前面，这和真实情况一致。
```

##### 测试二
```bash
seq1 = 4294967295
seq2 = 4294967296 + 4294967294，超过了可表示的范围，回绕为4294967294

分析：
这次`before(seq1,seq2)`返回的结果就与事实不符了。
```

##### 小结
上面的算法能使用的前提是：**回绕的范围不能太大。
准确地说，回绕的范围不能超过`2^31`，或者说，回绕之后的值，一定要小于`2^31`**。

> 注：`2^31为 s32的最大值???`

Linux 内核通过 `(_s32)(seq1-seq2) < 0` 来处理，即使绕回了，因为从无符号强制转换为符号，也会变成负数。

#### TCP 窗口大小
##### tcp windown字段
![](attachments/Pasted%20image%2020241113165551.png)

tcp 的 window 是2个字节(2的16次方)。

##### tcp win scale option

![](attachments/Pasted%20image%2020241113170422.png)

 TCP头只给接收窗口值留了16bit，接受窗口太小。
 解决方案就是在三次握手时是，把自己的Window Scale告知对方。Window Scale放在TCP头之外的Options中，向对方声明一个Shilt count，把它作为2的指数，再剩以TCP头中定义的接收窗口，就得到真正的TCP接收窗口了。

==注意==：
> shift.cnt取值范围为0~14，即最大TCP序号限定为2^16 * 2^ 14  = 2^30 < 2^31。该限制用于防止字节序列号溢出。

##### TCP窗口的最大值
那这个前提在TCP里面能满足吗？
可以！TCP窗口的最大值是`1GB = 2^30 B（2^16 * 2^14） < 2^31 B`;
所以使用上面的算法在实际情况中能够解决TCP的回绕问题，正确地区分出哪个序列号在前，哪个在后。

简单理解，==seq 和 ack 是在滑动窗口机制下 同步递增的，他们之间的差距不会超过 `2^31 - 1`==。

TCP 的滑动窗口机制，不会容忍发了`2147483649 (2^31 + 1)`个字节还没收到 ACK，TCP 发几个包后如果没收到ACK ，滑动窗口就满了，没法再发数据了，所以不可能连续发 `2147483649` 个字节 数据。

### 时间戳机制
#### 场景
理论上来说，上面的算法可以区分两个序列号，但是实际上可能出现这样的情况：
客户端发送了一个包A给服务器，序列号范围在`[seq1, seq2]`之间；
由于网络原因，这个包A过了很久都没有到达服务器
服务器流量很大，序列号回绕了一次，序列号范围又回到了`[seq1, seq2]`；
包A历经千辛万苦终于到达服务器
问题来了：这个包A是当前的有效包？还是回绕之前的包？


比如：
```text
假设三个数据包的*第一次*发送时间分别是A，B和C(A < B < C)，但A和C含有相同的序列号。
而A数据包由于某种原因，在阻塞在了网络中，因此发送方进行了重传，重传时间为A2


当接收端在接收到A2后，又接着确认到了数据包B，下一个想接收的数据是数据包C;
此时如果收到了数据包A(A从阻塞中恢复过来了，但并未真的丢失)，
由于A与C的序列号是相同的。如果没有别的保护措施就会出现数据紊乱，没有做到可靠传输
```

#### 解决: TCP 时间戳选项
这个问题可以使用时间戳来解决，通过判断报文段的时间戳，可以判断报文段是否是过时的。
TCP 时间戳选项（即放在TCP option中）用于 high performance。如果有一端没有开启时间戳选项，那么就不会使用了。
Timestamps 由 TSval 和 Tsecr 组成。
TSval：Timestamps value，发送时报文时携带本机的时间戳。
TSecr：Timestamps echo reply，发送时，将接收到的时间戳放在这里。
![](attachments/Pasted%20image%2020240328174947.png)


#### 时间戳作用
##### 两端往返时延测量（RTTM）
**（1）不启用 timestamp 选项**
在启用 timestamp 选项之前，测量 RTT 的过程如下。
![](attachments/Pasted%20image%2020240328175058.png)
TCP 在发送一个包时，会记录这个包的发送的时间 t1，用收到这个包的确认包时 t2 减去 t1 就可以得到这次的 RTT。这里有一个问题，如果发出的包出现重传，计算就变得复杂起来，如下所示。
![](attachments/Pasted%20image%2020240328175123.png)
这里的 RTT 到底是 t3 – t1 还是 t3 – t2 呢？这两种方式无论选择哪一种都不太合适，无法得知收到的确认 ACK 是对第一次包还是重传包的的确认。
==TCP RFC6298 对这种行为的处理是不对重传包进行 RTT 计算，这样计算不会带来错误，但当所有包都出现重传的情况下，将没有包可用来计算 RTT==。

**（2）启用 Timestamps 选项**
在启用 Timestamps 选项以后，因为 ACK 包里包含了 TSval 和 TSecr，这样无论是正常确认包，还是重传确认包，都可以通过这两个值计算出 RTT。


##### 防止序列号回绕（PAWS）
TCP 的序列号用 32bit 来表示，因此在 `2^32` 字节的数据传输后序列号就会溢出回绕。
TCP 的窗口经过窗口缩放可以最高到 1GB（`2^30`)，在高速网络中，序列号在很短的时间内就会被重复使用。

下面以一个实际的例子来说明，如下图所示。
![](attachments/Pasted%20image%2020240328175548.png)

假设发送了 6 个数据包，每个数据包的大小为 1GB，第 5 个包序列号发生回绕。第 2 个包因为某些原因延迟导致重传，但没有丢失到时间 t7 才到达。这个迷途数据包与后面要发送的第 6 个包序列号完全相同，如果没有一些措施进行区分，将会造成数据的紊乱。

如果有 Timestamps 的存在，内核会维护一个为==每个连接==维护一个 ts_recent 值，记录最后一次通信的的 timestamps 值，在 t7 时间点收到迷途数据包 2 时，由于数据包 2 的 timestamps 值小于 ts_recent 值，就会丢弃掉这个数据包。等 t8 时间点真正的数据包 6 到达以后，由于数据包 6 的 timestamps 值大于 ts_recent，这个包可以被正常接收。

#### 注意事项
（1） ==timestamps 值是一个单调递增的值，这个选项**不要求两台主机进行时钟同步**==。

两端 timestamps 值增加的间隔也可能步调不一致，比如一条主机以每 1ms 加一的方式递增，另外一条主机可以以每 1s 加一的方式递增。


（2）==timestamps 是一个双向的选项，如果只要有一方不开启，双方都将停用 timestamps==

![](attachments/Pasted%20image%2020240328175910.png)

可以看到客户端发起 SYN 包时带上了自己的 TSval，服务器回复的 SYN+ACK 包没有 TSval和TSecr，从此之后的包都没有带上时间戳选项了。

> 注意：**三次握手中的第二步，如果服务端回复 SYN+ACK 包中的 TSecr 不等于握手第一步客户端发送 SYN 包中的 TSval，客户端在对 SYN+ACK 回复 RST**。示例包如下所示。
![](attachments/Pasted%20image%2020240328175943.png)


（3）==序列号一样，递增 timestamps 值也是会溢出回绕的==。

timestamps 是 4B。如下所示：

![](attachments/Pasted%20image%2020241113172835.png)

# 参考
```bash
# TCP序列号回绕问题
https://blog.csdn.net/charles_neil/article/details/120603271

# 理解 TCP （图很不错）
https://www.inlighting.org/archives/understand-tcp
```