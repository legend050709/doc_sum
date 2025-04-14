```table-of-contents
```
# 问题

# 解决方案
## DNS64+NAT64
![](attachments/Pasted%20image%2020240111192700.png)

具体流程：
![](attachments/Pasted%20image%2020240111193231.png)
![](attachments/Pasted%20image%2020240111193317.png)
### DNS64
**介绍**
`DNS64`则主要是配合`NAT64`工作，主要是将`DNS`查询信息中的A记录（`IPv4`地址）合成到`AAAA`记录（`IPv6`地址）中，返回合成的`AAAA`记录用户给`IPv6`侧用户。

**流程**

在NAT64+DNS64场景中，主要分析IPv6用户发起的DNS解析流程。
对于IPv6用户而言，用户有可能访问IPv4-only服务器，也有可能访问IPv6服务器，对于访问IPv6服务器的DNS AAAA请求，DNS6服务器会直接返回IPv6地址；
对于访问IPv4-only服务器的DNS AAAA请求，DNS6服务器没有记录，需要首先将AAAA的请求转换为A的请求发送到DNS4上进行解析，得到A记录后再通过DNS64模块进行AAAA记录合成。

### NAT64
NAT 网关支持从 IPv6 到 IPv4 的网络地址转换，这通常称为 NAT64。

## bind中DNS64配置
`/etc/named.conf` 中的 options 模块中 配置 `dns64`。

```bash
dns64 ipv6-prefix {    
    //dns64是参数名，IPv6_prefix 参数是返回的IPv4地址应附加到的前缀，并且是必需的。比如64:ff9b::/96  
    [ clients { address_match_list } ; ]  
    // clients 是参数名， 指示为其提供服务的客户端的地址匹配列表； address_match_list 是地址段或者单个ip地址，每个dns64支持一个可选的clients访问控制表，它决定哪些客户端受这条指令的影响。如果未设 置，缺省值为any;

    [ mapped { address_match_list } ; ]
    //mapped是参数名 ，建议配置为any，每个dns64支持一个可选的mapped访问控制表，它在相关的A资源记录集中选择哪些IPv4地址会被映射。如果未设置，缺省值为any;。 

    [ exclude { address_match_list } ; ]  
    //  缺省为none，从DNS64服务中排除哪些IPv6客户端（将返回实际AAAA记录或NXDOMAIN）。 

    [ suffix ip6-address ; ]            
    // suffix 是参数名，可用于指定要包含在IPv4地址之后的映射响应中的其他位（默认为::）。例如，如果前缀长度是64位，而IPv4地址是32位，则在这种情况下可能会附加32位。

    [ recursive-only yes_or_no ; ]
     //recursive-only 是参数名，缺省为no，如果recursive-only被设置为yes，DNS64合成仅仅发生在递归请求

    [ break-dnssec yes_or_no ; ]  
    //  break-dnssec 是参数名，缺省为no， 如果break-dnssec被设置为yes，将会进行DNS64合成，即使结果在验证时会导致一个DNSSEC验 证失败。如果这个选项被设置为no（缺省值），进来的请求中带有DO位，在可用的记录中 有RRSIG，这时不会进行DNS64合成

}; 

```

范例：
![](attachments/Pasted%20image%2020240111192133.png)

# 参考
```c
# 通过NAT64实现ipv6 client 访问ipv4 Server
https://blog.csdn.net/legend050709/article/details/123377713
```