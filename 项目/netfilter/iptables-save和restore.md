```table-of-contents
```
# iptables-save
## 介绍
## 使用
### iptables规则保存及重载
```bash
#CentOS6 :
service iptables save
    iptables-save > /etc/sysconfig/iptables

service iptables restart
    iptables-restore < /etc/sysconfig/iptables

#CentOS7 :
引入了新的iptables前端管理服务工具:firewalld
    firewalld-cmd
    firewall-config
```
# iptables-restore
## 介绍
## 使用
# 参考
```bash

```