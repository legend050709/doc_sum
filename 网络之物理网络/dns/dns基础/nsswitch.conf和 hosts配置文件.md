```table-of-contents
```
# 本地主机映射文件hosts
## 介绍
**/etc/hosts 文件，保存主机名与IP地址的映射记录。**
hosts —— the static table lookup for host name（主机名查询静态表）。 
hosts文件是Linux系统上一个负责ip地址与域名快速解析的文件，以ascii格式保存在/etc/目录下。hosts文件包含了ip地址与主机名之间的映射，还包括主机的别名。

**hosts文件位置：**
Linux系统：`/etc/hosts`
Windows系统：`c/windows/system32/drivers/etc/hosts`

## 查看 /etc/hosts 文件
```bash
[root@localhost ~]# cat /etc/hosts
  127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
  ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
  119.75.218.70   www.baidu.com
```
## hosts 文件和DNS服务器的比较
- 默认情况下，系统首先从hosts文件查找解析记录
- hosts 文件的优先级高于DNS服务器，这是由 `/etc/nsswitch.conf` 文件规定的
- hosts 文件只对当前的主机有效  
- hosts 文件可减少DNS查询过程，从而加快访问速度

# /etc/nsswitch.conf 文件
## 介绍

## 查看
```bash
[root@localhost ~]# cat /etc/nsswitch.conf
 ------------------------------------
  38 #hosts:     db files nisplus nis dns
  39 hosts:      files dns myhostname
 

```
 files代表`/etc/hosts`文件，dns代表DNS服务器。
 **files**：
 files 写在dns之前，表示`/etc/hosts`文件的优先级高于DNS服务器。首先在本地 `/etc/hosts` 文件中查找,可以手动指定主机名与 IP 地址之间的映射关系。如果主机名在该文件中找到匹配项，系统将直接使用该 IP 地址，不进行 DNS 查询。
 
**dns**：
然后执行DNS域名解析查找`/etc/resolv.conf`,如果在`/etc/hosts` 文件中找不到匹配项，系统将使用 DNS 解析器进行域名解析。解析器会检查`/etc/resolv.conf` 文件以获取 DNS 服务器的配置信息。

**myhostname**：
最后使用查找`本地配置`的`系统主机名`,表示系统将使用本地主机名来解析主机名。本地主机名可以通过 /etc/hostname 文件或通过网络配置获得。使用 myhostname 关键字时，系统将尝试将主机名解析为本地主机名的 IP 地址。


## 模拟普通应用程序 DNS 解析过程
`glibc-common` 软件包中的 `getent` 命令，会按照`/etc/nsswitch.conf`所指定的主机名称解析顺序执行名称解析。这种解析过程也是大多数应用程序解析的过程。

使用方式
```bash
# getent hosts classroom.example.com 
172.25.254.254 classroom.example.com classroom
```

# 小结
## 域名的解析过程
Client（客户机）--> 查询/etc/hosts文件 --> Client DNS Service Local Cache （ 查询DNS服务本机缓存，只有Windows系统有）--> DNS Server(主机向本地域名服务器请求，递归查询)-->DNS Server Cache （本地域名服务器查询缓存信息）--> DNS iteration（本地域名服务器进行迭代查询）——>根域名服务器）-->一级域名服务器-->二级域名服务器-->三级域名服务器...


**用户在网页地址栏输入www.sina.com.cn]后，Linux系统中的DNS域名解析过程：**
1. 客户机先去查找本机的`/etc/hosts` 文件，看文件中是否存在该域名和IP地址的映射记录。如果有就调用，没有就进行下一步。
2. 客户机请求本地域名服务器（LDNS）来解析这个域名，主机要求本地域名服务器直接返回最终结果。在返回结果之前，客户机将完全处于等待状态，不再二次请求。统一由本地域名服务器向各级域名服务器转发请求。
3. 本地域名服务器收到客户机的请求后，先查询自己的缓存信息，如果有这个域名的映射记录则返回结果，没有则进行下一步。
4. 本地域名服务器请求根域名服务器解析这个域名，根域名告诉本地域名服务器去找对应的一级域名服务器。
5. 本地域名服务器请求一级域名服务器解析这个域名，一级域名服务器告诉它去找对应的二级域名服务器。
6. 本地域名服务器请求二级域名服务器解析这个域名，二级域名服务器告诉它去找对应的子域名服务器。
7. 本地域名服务器请求子域名服务器解析这个域名，子域名服务器返回对应的IP地址。
8. 本地域名服务器将IP地址记录到缓存中，并返回给客户机。客户机根据收到的IP地址访问该网站。


# 参考
```c
# Linux网络服务之DNS域名解析服务（上）

https://developer.aliyun.com/article/1079494?spm=a2c6h.27925324.detail.220.6a575c16cnVbrl
```