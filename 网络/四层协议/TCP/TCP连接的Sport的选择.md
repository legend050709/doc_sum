```table-of-contents
```

# 内核中的tcp连接sport的选择机制
# 影响因素
## ip_local_port_range
## ip_local_reserved_ports
## connect之前进行bind绑定port0
### 使用场景
### 影响
正常而言，bind一个非0的ip地址，但是port=0. 此时无论这个设备上存在多少个IP，
那么这个设备上的所有的可用的连接个数被限制在 ip_local_port_range 内。

比如 一个设备上存在ip1, ip2, ip3. 该设备上程序执行了bind(ip1, 0)。
存在的连接数已经接近了 ip_local_port_range。
那么接下来即使 通过 ip2进行连接，也是会报错。如下所示：
```c
# telnet 10.44.79.153 22
Trying 10.44.79.153...
telnet: connect to address 10.44.79.153: Cannot assign requested address
```
![](attachments/Pasted%20image%2020231116153005.png)
### 解决方法
![](attachments/Pasted%20image%2020231116151554.png)
# 参考
```c
https://patchwork.ozlabs.org/project/netdev/patch/1433650677.29864.26.camel@edumazet-glaptop2.roam.corp.google.com/
```