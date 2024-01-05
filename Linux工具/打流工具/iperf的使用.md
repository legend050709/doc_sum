```table-of-contents
```
# 介绍
# 参数
```c
-u: udp
-c: client
-p: server port to listen on/connect to
-t, --time      #        time in seconds to listen for new connections as well as to receive traffic (default not set)
-P, --parallel  #        number of parallel client threads to run
-b, --bandwidth #[kmgKMG | pps]  bandwidth to send at in bits/sec or packets per second
-s: server
-B: --bind <ip>[%<dev>]  bind to multicast address and optional device;
-i: --interval  #        seconds between periodic bandwidth reports;
-l, --len n[kmKM]
              set read/write buffer size (TCP) or length (UDP) to n 
-M: --mss n
              set TCP maximum segment size (MTU - 40 bytes)
              可以设置tcp数据包的大小；
```
# 范例
```c
udp client:
iperf -u -c 192.22.2.8 -p 3599 -b 500M -t 60 
iperf -u -c 192.22.2.8 -p 3599 -b 500M -t 60 -P 30
iperf -u -c 192.20.0.29 -p 3599 -b 500M -t 60 

udp server:
iperf -u -s -p 3599 -B 192.21.6.16 -i 1
iperf -u -s -p 3599 -B 192.21.7.16 -i 1

tcp client:
iperf -c 192.22.2.4 -p 80 -b 500M -t 60 
iperf -c 192.22.2.4 -p 80 -b 500M -t 60 -P 30

tcp server:
iperf -s -p 80 -B 192.21.6.11 -i 1
```
# 参考
```c

```