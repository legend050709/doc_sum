```table-of-contents
```
# 前言
要查看Linux系统中的TCP连接，可以使用以下几个命令：

1. netstat命令：使用netstat命令可以显示出当前系统中的网络连接情况。可以通过加上参数-t或-a来过滤只显示TCP连接，如：  
“`  
netstat -t #只显示TCP连接  
netstat -ta #显示所有TCP连接，包括监听和已建立连接  
“`

2. ss命令：ss命令是在netstat的基础上进行改进的，更加高效。可以使用以下命令来查看TCP连接：  
“`  
ss -t #只显示TCP连接  
ss -ta #显示所有TCP连接，包括监听和已建立连接  
“`

3. lsof命令：lsof命令可以列出系统中打开的文件和网络连接。可以通过加上参数-i来过滤只显示TCP连接，如：  
“`  
lsof -i tcp #只显示TCP连接  
“`

4. /proc文件系统：Linux系统中的/proc文件系统提供了很多关于系统状态的信息，包括当前的网络连接。可以通过访问`/proc/net/tcp`和`/proc/net/tcp6`文件来查看TCP连接的详细信息，如：  
“`  
cat /proc/net/tcp #显示IPv4的TCP连接信息  
cat /proc/net/tcp6 #显示IPv6的TCP连接信息  
“`

以上是在Linux系统中查看TCP连接的几种常用方法，根据实际情况选择合适的命令来查看。

# ss命令
# /proc/net/tcp文件
# 参考
```bash
# 「八」Linux文件/proc/net/tcp分析
https://guanjunjian.github.io/2017/11/09/study-8-proc-net-tcp-analysis/
```