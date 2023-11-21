```table-of-contents
```
# 内核 socket lookup 逻辑
内核中的 socket lookup 逻辑，也就是当 TCP 层收到一个包时， **==如何判断这个包属于哪个 socket==**。
逻辑其实非常简单： 两阶段， 先精确匹配，再模糊匹配：
![](attachments/Pasted%20image%2020231120142513.png)
![](attachments/Pasted%20image%2020231120142521.png)

1. 首先是 `(src_ip,src_port,dst_ip,dst_port)` 4-tuple 精确匹配，看能不能找到 **connected 状态的 socket**；如果找不到，
2. 再尝试 `(dst_ip,dst_port)` 2-tuple，寻找有没有 **listening 状态的 socket**；如果还是没找到，
3. 再尝试 `(INADDR_ANY)` 1-tuple，寻找有没有 **listening 状态的 socket**。

# 参考
```c
# [译] Socket listen 多地址需求与 SK_LOOKUP BPF 的诞生
https://arthurchiao.art/blog/birth-of-sk-lookup-bpf-zh/#12-%E9%9C%80%E6%B1%82%E5%A6%82%E4%BD%95%E8%AE%A9%E4%B8%80%E4%B8%AA%E6%9C%8D%E5%8A%A1%E7%9B%91%E5%90%AC%E8%87%B3%E5%B0%91%E5%87%A0%E7%99%BE%E4%B8%AA-ip-%E5%9C%B0%E5%9D%80
```