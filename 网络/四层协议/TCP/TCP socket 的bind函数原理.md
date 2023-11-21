```table-of-contents
```
# 内核层面的bind流程
# 接口多ip指定sip的解决方法
## connect之前进行bind
### 方法
### 问题
### 解决
**IP_BIND_ADDRESS_NO_PORT**
![](attachments/Pasted%20image%2020231116151554.png)
ipv4 以及 ipv6的socket 都是使用 SOL_IP level的 IP_BIND_ADDRESS_NO_PORT 来进行setsockipt 设置。
# 参考
```c
https://patchwork.ozlabs.org/project/netdev/patch/1433650677.29864.26.camel@edumazet-glaptop2.roam.corp.google.com/
```