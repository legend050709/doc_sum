```table-of-contents
```
# 监控统计

参考：[Monitoring Recommendations for BIND 9](https://kb.isc.org/v1/docs/en/monitoring-recommendations-for-bind-9)

## 获取方式
![](attachments/Pasted%20image%2020240124120307.png)

### named.stats文件
介绍如上所示
### statistics channel 信道
![](attachments/Pasted%20image%2020240124121435.png)

#### 监听端口
默认情况下，bind 监听UDP 53端口，作为DNS的查询。
监听TCP 953，作为和 rndc的通信。
监听 TCP 80，作为 统计查询。
```bash
BIND listens for queries on Port 53. It listens for communication from RNDC on port 953. And it can be configured to listen for statistics requests on another port, typically port 80.
```
#### 统计内容
```bash
The statistics channel now also includes many new statistics, including stats for the resolver, cache, address database, dispatch manager and task manager, which collectively can be used to monitor server health.
```

#### 配置
![](attachments/Pasted%20image%2020240124144703.png)
需要编译bind9的时候就支持XML或者JSON格式的 统计查询。然后在`named.conf`中配置
统计信道的配置。

```bash
statistics-channels {
     inet 10.1.10.10 port 8080 allow { 192.168.2.10; 10.1.10.2; };
     inet 127.0.0.1 port 8080 allow { 127.0.0.1; };
};
```

#### 对比`named.stats`文件方式
![](attachments/Pasted%20image%2020240124121620.png)


## 监控指标
### 常见指标
![](attachments/Pasted%20image%2020240124142507.png)

### 递归服务器的监控指标
![](attachments/Pasted%20image%2020240124140735.png)

### 权威服务器的监控指标
![](attachments/Pasted%20image%2020240124141904.png)

# 问题定位
fewf

# 参考
```bash
https://kb.isc.org/v1/docs/en/monitoring-recommendations-for-bind-9

# youtube:  bind 9 statistics monitor and log analysis
https://www.youtube.com/watch?v=7Uu6XvY68SM

# Long-Term Statistics Monitoring and Log Analysis
https://www.isc.org/docs/BIND_9webinar2.pdf

# bind ARM 官方用户手册：BIND 9.19 Administrator Reference Manual
https://kb.isc.org/docs/aa-01031
```