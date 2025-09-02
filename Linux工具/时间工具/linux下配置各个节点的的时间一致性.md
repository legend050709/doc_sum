```table-of-contents
```

# date 命令调整系统时间

```bash
date #查看当前系统时间和日期
date -s 02/21/2019 #设置日期,例如将系统日期设定成 2019 年 02 月 21 日
date -s 19:21:45 #设置时分秒，例如将系统时间设定成 19 点 21 分 45 秒
hwclock -w #将当前时间和日期写入 BIOS，即设置为硬件时间，避免重启后失效

```


# ntp 调整系统时间
NTP（Network Time Protocol，网络时间协议），用来在分布式时间服务器和客户端之间进行时间同步。NTP基于UDP报文进行传输，使用的UDP端口号为123。它提供高精准度的时间校正（LAN 上与标准间差小于1毫秒，WAN 上几十毫秒），且可通过加密确认的方式来防止恶毒的协议攻击。

使用NTP的目的是对网络内所有具有时钟的设备进行时钟同步，使网络内所有设备的时钟保持一致，从而使设备能够提供基于统一时间的多种应用。对于运行NTP的本地系统，既可以接收来自其他时钟源的同步，又可以作为时钟源同步其他的时钟，并且可以和其他设备互相同步。

## ntp 工作原理
NTP的基本工作原理如图所示。Device A和Device B通过网络相连，它们都有自己独立的系统时钟，需要通过NTP实现各自系统时钟的自动同步。为便于理解，作如下假设：  

在Device A和Device B的系统时钟同步之前，Device A的时钟设定为10:00:00am，Device B的时钟设定为11:00:00am。NTP报文在Device A和Device B之间单向传输所需要的时间为1秒。

![](attachments/Pasted%20image%2020250813192736.png)


- Device A发送一个NTP报文给Device B，该报文带有它离开Device A时的时间戳，该时间戳为10:00:00am（T1）。
- 当此NTP报文到达Device B时，Device B加上自己的时间戳，该时间戳为11:00:01am（T2）。
- 当此NTP报文离开Device B时，Device B再加上自己的时间戳，该时间戳为11:00:02am（T3）。
- 当Device A接收到该响应报文时，Device A的本地时间为10:00:03am（T4）。
- 至此，Device A已经拥有足够的信息来计算两个重要的参数：  
    NTP报文的往返时延Delay=（T4-T1）-（T3-T2）=2秒。  
    Device A相对Device B的时间差offset=（（T2-T1）+（T3-T4））/2=1小时。

## ntp 报文

![](attachments/Pasted%20image%2020250813192821.png)

其中192.10.10.189为NTP的server端，192.10.10.32为client端。

![](attachments/Pasted%20image%2020250813192834.png)

![](attachments/Pasted%20image%2020250813192846.png)


![](attachments/Pasted%20image%2020250813192908.png)

客户端和服务端都有一个时间轴，分别代表着各自系统的时间，当客户端想要同步服务端的时间时，客户端会构造一个NTP协议包发送到NTP服务端，客户端会记下此时发送的时间t0，经过一段网络延时传输后，服务器在t1时刻收到数据包，经过一段时间处理后在t2时刻向客户端返回数据包，再经过一段网络延时传输后客户端在t3时刻收到NTP服务器数据包。t0和t3是客户端时间系统的时间、t1和t2是NTP服务端时间系统的时间，它们是有区别的。

t0、t1、t2分别对应着server->cient NTP报文中的三个参数：  
t0：origin timestamp  
t1: receive timestamp  
t2: transmit timestamp  
t3为client收到回复报文时本地的时间。

**延时和时间偏差计算**
假设：客户端与服务端的时间系统的偏差定义为θ、网络的往/返延迟(单程延时)定义为δ。  
推导过程：  
1）根据交互原理，可以列出方程组：  
t0+θ+δ=t1  
t2-θ+δ=t3  
2）求解方程组，得到以下结果：  
θ=(t1-t0+t2-t3)/2  
δ=(t1-t0+t3-t2)/2  
记忆时可以采用极限法，分别假设延时和偏差为0.
    
**client时间校准**：  
对于时间要求不那么精准设备，client端可把server端的返回时间t2固化为本地时间。但是作为一个标准的通信协议，必须计算上网络的传输延时，需要把t2+δ 固化为本地时间。

## ntp使用
### 检测 ntp 是否已安装
```bash
rpm -qa | grep ntp #检查命令

若只有 ntpdate 而未见 ntp，则需删除原有 ntpdate：
rpm -e ntpdate-4.2.6p5-22.el7.x86_64 #删除 ntpdate

```

### 用ntpdate自动更新系统时间

- 采用微软的校时服务器调整系统时间
```bash
yum install -y ntp #安装 ntp 服务器
ntpdate time.windows.com #采用微软的校时服务器调整系统时间
```

- 设置自动同步，同步频率：每十分钟同步一次
```bash
crontab -e #编辑crontab
*/10 * * * * /usr/sbin/ntpdate time.windows.com >> /tmp/crontab.log

```



# 参考
```bash
https://blog.csdn.net/legend050709/article/details/130132277
```