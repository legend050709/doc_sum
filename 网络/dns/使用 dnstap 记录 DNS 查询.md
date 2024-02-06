```table-of-contents
```
# 介绍
## 前提条件
`bind-9.11.26-2` 软件包或更新的版本已安装。

# 配置
## 编辑 `options` 块启用 `dnstap`
```bash
options
{
	# ...
	dnstap { all; }; # Configure filter
	dnstap-output file "/var/named/data/dnstap.bin";
	# ...
};
# end of options
```

### DNS过滤器
指定您要记录的 DNS 流量类型，请将 `dnstap` 过滤器添加到 `/etc/named.conf` 文件中的 `dnstap` 块中。
可以使用以下过滤器：
- `auth` - 权威区域响应或回答。
- `client` - 内部客户端查询或回答。
- `forwarder` - 转发的查询或来自它的响应。
- `resolver` - 迭代的解析查询或响应。
- `update` - 动态区域更新请求。
- `all` - 以上选项中的任何一个。
- `query` 或 `response` - 如果您没有指定 `query` 或 `response` 关键字，则 `dnstap` 两个都记录。


`dnstap` 过滤器包含多个由 `;` 分隔的定义，`dnstap {}` 块的语法如下：`dnstap{(all | auth | client | forwarder | resolver | update)[(query | response)]; …​ };`

### 日志配置定期回滚
在以下示例中，`cron` 调度程序每天运行一次用户编辑的脚本的内容。
值为 `3` 的 `roll` 选项指定 `dnstap` 最多可以创建三个备份日志文件。值 `3` 覆盖 `dnstap-output` 变量的 `version` 参数，并将备份日志文件数限制为三个。

二进制日志文件被移到另一个目录并被重命名，并且永远不会达到 `.2` 后缀，即使三个备份文件已存在。
如果根据大小限制自动回滚二进制日志足够了，则您可以跳过这一步。

```bash
Example:

sudoedit /etc/cron.daily/dnstap

#!/bin/sh
rndc dnstap -roll 3
mv /var/named/data/dnstap.bin.1 /var/log/named/dnstap/dnstap-$(date -I).bin

# use dnstap-read to analyze saved logs
sudo chmod a+x /etc/cron.daily/dnstap
```
### `dnstap-read`读取
在以下示例中，`dnstap-read` 工具以 `YAML` 文件格式打印输出
```bash
Example:

dnstap-read -y [file-name]
```
# 性能
`dnstap`对性能的影响比较小，关闭`querylog`后 `dnstap` 关闭/开启的性能对比大概是`13W/S` VS `10W/S`。相比传统的`querylog`的性能影响实在好太多了。但是因为是二进制的文件，查看需要用`dnstap-read`还是非常不方便的。
# 参考
```c
# 使用 dnstap 记录 DNS 查询
https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/8/html/managing_networking_infrastructure_services/proc_recording-dns-queries-using-dnstap_assembly_setting-up-and-configuring-a-bind-dns-server
```