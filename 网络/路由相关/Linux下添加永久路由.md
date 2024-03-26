```table-of-contents
```
# 背景
通常我们使用`ip route add xxx`添加临时路由。  
当系统重启，临时路由将丢失，重新配置路由带来了不必要的麻烦。可通过固化临时路由为永久路由的方法解决该问题。
# 介绍
`static-routes`文件为路由固化文件。`/etc/sysconfig`目录下，系统一般不会自动生成`static-routes`文件，需要手工创建。

# `static-routes`文件中路由固化的格式
```bash
① 添加默认路由
any net 0.0.0.0 netmask 0.0.0.0 gw 10.92.2.1
或者
any net 0.0.0.0/0 gw 10.92.2.1

② 添加网络路由
any net 1.1.1.0 netmask 255.255.255.0 gw 1.1.1.10
或者
any net 1.1.1.0/24 gw 1.1.1.10
```
# 参考
```bash
# [Linux添加永久路由的方法](https://www.cnblogs.com/t-bar/p/11170120.html)

```