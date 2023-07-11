# 概述
# 场景
# 使用方法
# 操作

## 删除
### 删除指定范围的行
```bash
for a in `seq 25 39`; do sed -i '18,145d' 192.22.2.${a}_tcp_8080.conf; done
```
### 删除匹配指定字符串的行
```c
sed -i '/hello/d' aaaa
sed -i '/^hello/d' aaaa
```
## 添加
### 指定字符串之后行添加
- 在指定字符串的下一行插入一行
```c
sed -i '/snat_log_enable/a\        snat_log_stop 1' /etc/dpvs.conf
或者
sed -i "/snat_log_enable/a\\        snat_log_stop 1" /etc/dpvs.conf
```

- 在指定字符串的下面插入多行
```bash
比如，要在keepalived.conf中的 “cluster_tag test-fnat“ 字符串后面插入：
        vxlan {
            vni 1
            local_vtep_ip 192.22.2.100
            local_vtep_mac 0C:42:A1:90:F4:43
            vtep_ip 192.20.144.100
            vtep_port 4789
            vtep_mac 0c:42:a1:78:4f:a1
        }


# sed -i "/cluster_tag test-fnat/a\\        vxlan {\n            vni 1\n            local_vtep_ip 192.22.2.100\n            local_vtep_mac 0C:42:A1:90:F4:43\n            vtep_ip 192.20.144.100\n            vtep_port 4789\n            vtep_mac 0c:42:a1:78:4f:a1\n        }\n" keepalived.conf
```

### 注释指定行
```c
文件中，不是以#开头的行，在开头添加#
sed -i "s/^[^#].*$/#&/g" ${outFile}
```
![](attachments/Pasted%20image%2020230720123333.png)
## 替换
### 替换为常量
```c
ls *tcp_8080.conf | xargs sed -i 's/192.20.1/192.20.45/g'
```
### 替换为变量
如下所示，变量用'${xx}' 。
```c
for a in `seq 25 39`; do sed -i 's/192.22.2.'${a}'_tcp_80/192.22.2.'${a}'_tcp_8080/g' 192.22.2.${a}_tcp_8080.conf; done
```
# 参考
