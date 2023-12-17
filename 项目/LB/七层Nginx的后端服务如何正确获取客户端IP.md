```table-of-contents
```
# 背景
在Web应用开发中，经常会需要获取客户端IP地址。一个典型的例子就是投票系统，为了防止刷票，需要限制每个IP地址只能投票一次。  
如何获取客户端IP在Java中，获取客户端IP最直接的方式就是使用request.getRemoteAddr()。这种方式能获取到连接服务器的客户端IP，在中间没有代理的情况下，的确是最简单有效的方式。但是目前互联网Web应用很少会将应用服务器直接对外提供服务，一般都会有一层Nginx做反向代理和负载均衡，有的甚至可能有多层代理。在有反向代理的情况下，直接使用request.getRemoteAddr()获取到的IP地址是Nginx所在服务器的IP地址，而不是客户端的IP。  
  
HTTP协议是基于TCP协议的，由于request.getRemoteAddr()获取到的是TCP层直接连接的客户端的IP，对于Web应用服务器来说直接连接它的客户端实际上是Nginx，也就是TCP层是拿不到真实客户端的IP。  
  
为了解决上面的问题，很多HTTP代理会在HTTP协议头中添加X-Forwarded-For头，用来追踪请求的来源。

# 基础知识
## `X-Forwarded-For`介绍
`X-Forwarded-For`是 HTTP头的一个字段，最开始是由 `Squid`这个缓存代理软件引入，在客户端访问服务器的过程中如果需要经过HTTP代理或者负载均衡服务器，可以被用来获取最初发起请求的客户端的IP地址，如今它已经成为事实上的标准，被各大 HTTP 代理、负载均衡等转发服务广泛使用。 [RFC 7239](http://tools.ietf.org/html/rfc7239)（Forwarded HTTP Extension）是这个头信息的标准化版本。

## `X-Forwarded-For` 的格式
`X-Forwarded-For` 的格式如下：
```
X-Forwarded-For: client1, proxy1, proxy2
```
`X-Forwarded-For` 包含多个IP地址，每个值通过逗号+空格分开，最左边（client1）是最原始客户端的IP地址，中间如果有多层代理，每一层代理会将连接它的客户端IP追加在`X-Forwarded-For`右边。

> 注：`X-Forwarded-For`只规定了这个字段的格式，并不代表这是代理服务器的默认行为，是否增加还要是具体配置。

## Nginx添加`X-Forwarded-For`

要让 Nginx支持`X-Forwarded-For`头，需要配置：  
```shell
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

- **(1) X-Forwarded-For**
> the “X-Forwarded-For” client request header field with the `$remote_addr` variable appended to it, separated by a comma. If the “X-Forwarded-For” field is not present in the client request header, the `$proxy_add_x_forwarded_for` variable is equal to the `$remote_addr` variable.

>分两种情况：
> - 如果请求中不带X-Forwarded-For头，那么取`$remote_addr`的值；
> - 如果请求中带X-Forwarded-For头，那么在X-Forwarded-For后追加`$remote_addr`，即: `X-Forwarded-For,$remote_addr`。
`$proxy_add_x_forwarded_for` 会将和`Nginx`直接连接的设备的IP追加在请求原有`X-Forwarded-For`值的右边。

- **(2)$remote_addr**  
是nginx与客户端进行TCP连接过程中，获得的客户端真实地址. 
> Remote Address 无法伪造，因为建立 TCP 连接需要三次握手，如果伪造了源 IP，无法建立 TCP 连接，更不会有后面的 HTTP 请求

- **(3)X-Real-IP**  
是一个自定义头。X-Real-Ip 通常被 HTTP 代理用来表示与它产生 TCP 连接的设备 IP，这个设备可能是其他代理，也可能是真正的请求端。
> 需要注意的是，X-Real-Ip 目前并不属于任何标准，代理和 Web 应用之间可以约定用任何自定义头来传递这个信息。

## 后端server获取client-ip
下面就是一种常用的获取客户端真实IP的方法，首先从HTTP头中获取`X-Forwarded-For`，如果`X-Forwarded-For`头存在就按逗号分隔取最左边第一个IP地址，不存在直接通过`request.getRemoteAddr()`获取IP地址：
```c
public String getClientIp(HttpServletRequest request) {  
    String xff = request.getHeader("X-Forwarded-For");  
    if (xff == null) {  
        return request.getRemoteAddr();  
    } else {  
        return xff.contains(",") ? xff.split(",")[0] : xff;  
    }  
}
```

# 伪造`X-Forwarded-For`
一般的客户端（例如浏览器）发送HTTP请求是没有`X-Forwarded-For`头的，当请求到达第一个代理服务器时，代理服务器会加上`X-Forwarded-For`请求头，并将值设为客户端的IP地址（也就是最左边第一个值），后面如果还有多个代理，会依次将IP追加到`X-Forwarded-For`头最右边，最终请求到达Web应用服务器，应用通过获取`X-Forwarded-For`头取左边第一个IP即为客户端真实IP。

但是如果客户端在发起请求时，请求头上带上一个伪造的`X-Forwarded-For`，由于后续每层代理只会追加而不会覆盖，那么最终到达应用服务器时，获取的左边第一个IP地址将会是客户端伪造的IP。也就是上面的Java代码中`getClientIp()`方法获取的IP地址很有可能是伪造的IP地址，如果一个投票系统用这种方式做的IP限制，那么很容易会被刷票。

> 使用`curl -H 'X-Forwarded-For: 8.8.8.8' http://www.dianduidian.com`一条命令就能实现伪造。

## 解决方法
### 方法一
TCP必须经过3次握手，客户端的IP是无法伪造的。所以**最外层的代理**一定要取`$remote_addr`的值，即：第一层代理（proxy1）用`remote_addr`来覆盖`X-Forwarded-For`。

第一层代理的对应配置：
```shell
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $remote_addr;
```
> 分析：
> 这里使用`$remote_addr`替代上面的`$proxy_add_x_forwarded_for`。`$proxy_add_x_forwarded_for`会在原有`X-Forwarded-For`上追加IP，这就相当于给了伪造`X-Forwarded-For`的机会。而`$remote_addr`是获取的是直接TCP连接的客户端IP，这个是无法伪造的，即使客户端伪造也会被覆盖掉，而不是追加。


第二三四层代理的对应配置：
```shell
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### 方法二
另外一种方法是我在Tomcat源码中发现的：`org.apache.catalina.valves.RemoteIpValve `

实现思路：
> 遍历`X-Forwarded-For`头中的IP地址，和上面方法不同的是，不是直接取左边第一个IP，而是从右向左遍历。遍历时可以根据正则表达式剔除掉内网IP和已知的代理服务器本身的IP（例如192.168开头的），那么拿到的第一个非剔除IP就会是一个可信任的客户端IP。
这种方法的巧妙之处在于，即时伪造`X-Forwarded-For`，那么请求到达应用服务器时，伪造的IP也会在`X-Forwarded-For`值的左边，从右向左遍历就可以避免取到这些伪造的IP地址。这种方式本文就不提供具体实现代码了，有兴趣可以查看`Tomcat`源码。

# 参考
```c
# 谈谈Nginx做反代时后端服务如何正确获取客户端IP?
https://blog.dianduidian.com/post/nginx-x-forwarded-for/

利用X-Forwarded-For伪造客户端IP漏洞成因及防范
https://bbs.sangfor.com.cn/forum.php?mod=viewthread&tid=56611
```