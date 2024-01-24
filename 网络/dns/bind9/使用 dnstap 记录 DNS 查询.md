```table-of-contents
```
# 性能
`dnstap`对性能的影响比较小，关闭`querylog`后 `dnstap` 关闭/开启的性能对比大概是`13W/S` VS `10W/S`。相比传统的`querylog`的性能影响实在好太多了。但是因为是二进制的文件，查看需要用`dnstap-read`还是非常不方便的。
# 参考
```c
# 使用 dnstap 记录 DNS 查询
https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/managing_networking_infrastructure_services/proc_recording-dns-queries-using-dnstap_assembly_setting-up-and-configuring-a-bind-dns-server
```