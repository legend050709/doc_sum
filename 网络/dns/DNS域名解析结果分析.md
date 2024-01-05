```table-of-contents
```
# 关于DNS-Rcode
DNS-Rcode作为DNS应答报文中有效的字段，主要用来说明DNS应答状态，这可是小编排查域名解析失败的重要指标。通常常见的Rcode值如下：
- Rcode值为0，对应的DNS应答状态为NOERROR，意思是成功的响应，即这个域名解析是成功
- Rcode值为2，对应的DNS应答状态为SERVFAIL，意思是服务器失败，也就是这个域名的权威服务器拒绝响应或者响应REFUSE，递归服务器返回Rcode值为2给CLIENT
- Rcode值为3，对应的DNS应答状态为NXDOMAIN，意思是不存在的记录，也就是这个具体的域名在权威服务器中并不存在
- Rcode值为5，对应的DNS应答状态为REFUSE，意思是拒绝，也就是这个请求源IP不在服务的范围内
![](attachments/Pasted%20image%2020240104105307.png)

# DNS请求结果具体分析
## NXDOMAIN
域名记录不存在，即Rcode值为3（NXDOMAIN）的情况。
这种情况下域名权威服务器及托管的主域名(zone)均正常，但是权威并不存在这条具体的域名记录，于是权威返回了`NXDOMAIN`。
值的注意的是这个`NXDOMAIN`的报文中会包含一个`AUTHORITY SECTION`，内容为该主域名的SOA记录（对应zone中的SOA记录），这个应答结果会在递归服务器中被缓存，缓存时间周期为域名的SOA记录的TTL：
![](attachments/Pasted%20image%2020240104141717.png)

## SERVFAIL
权威解析失败，即Rcode值为2（SERVFAIL）的情况。
递归服务器会给请求源这个结果的原因是向权威解释请求异常，包括且不限于权威不响应/或者权威返回refuse/或者权威返回`servfail`。
这个`SERVFAIL`的应答结果当然是一个空结果，不过BIND会强制给这个结果增加一个1S的TTL，所以`SERVFAIL`的应答会在递归服务器中被缓存，缓存时间周期为1S。

### 权威不响应
包括递归服务器至权威服务器中间的网络异常在内，递归服务器在发出递归请求并完成重试超时后，给请求源一个SERVFAIL的应答，并缓存1S ：
![](attachments/Pasted%20image%2020240104141952.png)

### 权威向递归服务器应答REFUSE
当权威服务器不存在主域名（即zone文件）及对应的SOA记录时，权威会向递归服务器返回REFUSE，即不在我服务的范围内拒绝，递归服务器在收到这个REFUSE应答后，给请求源一个SERVFAIL的应答，并缓存1S。
> 注：此中和`NXDOMAIN`区别：此中是权威服务器不包含对应的Zone；而`NXDOMAIN`不存在具体的请求记录，但是存在Zone（一个Zone中包含了多个A/NS记录等）。

![](attachments/Pasted%20image%2020240104142254.png)

### 权威向递归服务器应答SERVFAIL
当权威服务器存在主域名但是由于zonefile被破坏导致权威服务器上域名的NS记录异常时，权威会向递归服务器返回SERVFAIL，即解析失败，递归服务器在收到这个SERVFAIL应答后，给请求源一个SERVFAIL的应答，并缓存1S。
![](attachments/Pasted%20image%2020240104142335.png)

### 权威向递归服务器应答其他的错误Rcode
由于不常见本文就不展开了，递归服务器在收到其他错误应答后，给请求源一个SERVFAIL的应答，并缓存1S。

## REFUSE
拒绝服务，即Rcode值为5（REFUSE）的情况。
除了记录不存在（NXDOMAIN）和解析失败（SERVFAIL）以外，如果请求源不在递归服务器的服务范围内，这种情况下递归服务器会直接给请求源一个REFUSE的应答，本地直接应答无缓存。
![](attachments/Pasted%20image%2020240104142519.png)

## 响应成功，但是没有解析结果
这是一种比较特殊的情况，这种情况是Rcode值为0（NOERROR）的情况。
这种情况下域名权威服务器及托管的主域名均正常，权威本身也存在这条具体的域名记录，但是没有对应的记录类型（不包含CNAME，CNAME是特殊情况，可以响应任意类型的请求）。
这时权威返回了`NOERROR`，值的注意的是这个`NOERROR`的报文中没有`ANSWER SECTION`。但是会包含一个`AUTHORITY SECTION`，内容为改主域名的SOA记录(即对应Zone文件中的SOA记录)，这个应答结果会在递归服务器中被缓存，缓存时间周期为域名的SOA记录的TTL。
![](attachments/Pasted%20image%2020240104143002.png)

## 递归服务器本身不响应
如果递归服务器不响应，那么请求段收不到任何应答，这个时候请求端终端如果有超时机制则会跑出一个dns请求 timeout的结果。
![](attachments/Pasted%20image%2020240104143043.png)
## 小结
![](attachments/Pasted%20image%2020240104141446.png)

# 解析故障排查技巧

# 参考
```c
# 阿里DNS：域名解析失败的那些事
https://zhuanlan.zhihu.com/p/40659713

# 跟我学-域名解析故障排查技巧
https://zhuanlan.zhihu.com/p/101378917
```