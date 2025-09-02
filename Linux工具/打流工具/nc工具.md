```table-of-contents
```
# nc传输文件
# nc 发送tcp
## IPv4
### 超时设置
需要注意的是，如果不设置超时时间，默认情况下nc命令会一直等待连接成功或出现错误，直到手动中断程序。

## ipv6
## 发送特定流
nc 可以指定 srcport，dip，dport 发送指定数据流。
ab 以及 curl 无法指定srcport，使用 nc 继续for 循环来指定srcport。
比如：
```c
nc -v -p55666 192.22.2.3 8080
```
## nc模拟长连接以及短连接
参考下面的场景范例。
## nc模拟curl
由于Curl以及ab无法指定sport，如果期望指定sport进行curl。那么可以使用nc指定sport，然后模拟进行curl。

可以先使用 curl -v xxx 查看请求的格式；然后通过` echo -e "xxx\nyyy\nzzz\n.." | nc -tv IP PORT` 来模拟get请求. 比如
```c
echo -e "GET /hiknini/itemList.html HTTP/1.1\nHost:localhost\n\n" | nc 192.168.1.107 8080
```

# nc 发送udp
## IPv4
## ipv6
# 场景
## 测试TCP 三次握手/四次挥手
```c
nc IP PORT -z -v
作用：短连接，三次握手然后立刻主动四次挥手。

telnet IP PORT 可以模拟长连接。
nc IP PORT -tv 也是长连接。

echo 1111 |  nc IP PORT -tv 则是短连接，三次握手，然后发送数据，然后主动四次挥手。
```
比如：
```c
# nc 10.44.79.156 80 -z -v
Ncat: Version 7.50 ( https://nmap.org/ncat )
Ncat: Connected to 10.44.79.156:80.
Ncat: 0 bytes sent, 0 bytes received in 0.01 seconds.
```
抓包如下所示：
![](attachments/Pasted%20image%2020231013200028.png)

### tcp偶发超时测试复现
场景
比如某个机器A TCP和其他设备进行三次握手，偶发的时延比较大或者不通。如何复现这个偶发的问题。

![](attachments/Pasted%20image%2020250707161218.png)

```c
for i in {1..1000};do echo $i;  time nc IP PORT -z -v;done 2>&1  | grep real  | sort -rn | head -n 5
```
![](attachments/Pasted%20image%2020231013200323.png)
如上所示，复现出来部分的TCP三次握手的时间较长。

# 参考