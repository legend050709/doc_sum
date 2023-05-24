# 常用用法
# curl查看时延等信息
## httpstat工具
- 背景
是否经历过通过命令行**curl**某个接口的时候响应很慢，但是你并不能百分之百肯定是那一个阶段的耗时多，是否跟研发掰扯过我的服务访问慢，是网络有问题之类的场景。
在任何需要分析网站速度在每个阶段耗时的场景下，通过抓包分析报文的方式太繁杂，有这么一款工具，可视化将每个阶段耗时统计出来。
- 介绍
httpstat通过封装curl命令，将整个连接过程每个阶段耗时可视化统计出来. 就如README所述："httpstat visualizes `curl(1)` statistics in a way of beauty and clarity。"
- 安装
```c
方法一：
wget -c https://raw.githubusercontent.com/reorx/httpstat/master/httpstat.py

方法二：
pip install httpstat
```
- 使用
`httpstat` 可以根据你安装它的方式来使用，如果你直接下载了它，进入下载目录使用下面的语句运行它：
```c
python httpstat.py url cURL_options
```

如果你使用 `pip` 来安装它，你可以作为命令来执行它，如下表：
```c
httpstat url cURL_options
```
![](attachments/Pasted%20image%2020230619103630.png)



curl 参数：
```c
httpstat <URL> -X POST -d 'xxx' -v

1. `-X` 命令标记指定一个客户与 HTTP 服务器连接的请求方法。
2. -d参数默认将POST字段的内容以application/x-www-form-urlencoded类型传递给服务端。
3. 如果是application/json格式，则需要使用`-H "Content-Type: application/json"`指定类型。
4. `--data-urlencode` 这个选项将会把数据按 URL 编码的方式编码后再提交。
5. `-v` 开启详细模式。
6. 

```
![](attachments/Pasted%20image%2020230619105842.png)


环境变量：
```c
1》如果只是单次生效
直接在httpstat前面加变量声明即可，shell会将此变量解析，只在这条命令中单次生效，如：

HTTPSTAT_SHOW_BODY=true httpstat https://cloud.tencent.com

2》如果需要在当前终端生效
则需要用到export来申明变量，如：
export HTTPSTAT_SHOW_BODY=true
httpstat https://cloud.tencent.com

如需取消，通过unset命令来重置:
HTTPSTAT_SHOW_BODY

3》需要永久生效，则将`export`的变量赋值写入到`.bashrc`或`.zshrc`;
根据shell解释器配置文件走，如：
export HTTPSTAT_SHOW_IP=false
export HTTPSTAT_SHOW_SPEED=true
export HTTPSTAT_SAVE_BODY=false

```

- 范例
![](attachments/Pasted%20image%2020230619104513.png)

## curl自带参数
`curl` 命令提供了 `-w` 参数：
![](attachments/Pasted%20image%2020230619102355.png)

先往文本文件 `curl-format.txt` 写入下面的内容：
```c
~ cat curl-format.txt
    time_namelookup:  %{time_namelookup}\n
       time_connect:  %{time_connect}\n
    time_appconnect:  %{time_appconnect}\n
      time_redirect:  %{time_redirect}\n
   time_pretransfer:  %{time_pretransfer}\n
 time_starttransfer:  %{time_starttransfer}\n
                    ----------\n
         time_total:  %{time_total}\n

```
```c
这些变量都是什么意思呢？我解释一下：

- `time_namelookup`：DNS 域名解析的时候，就是把 `https://zhihu.com` 转换成 ip 地址的过程
- `time_connect`：TCP 连接建立的时间，就是三次握手的时间
- `time_appconnect`：SSL/SSH 等上层协议建立连接的时间，比如 connect/handshake 的时间
- `time_redirect`：从开始到最后一个请求事务的时间
- `time_pretransfer`：从请求开始到响应开始传输的时间
- `time_starttransfer`：从请求开始到第一个字节将要传输的时间
- `time_total`：这次请求花费的全部时间
```
范例：
```bash
curl -w "@curl-format.txt" -o /dev/null -s -L "http://cizixs.com"
    time_namelookup:  0.012
       time_connect:  0.227
    time_appconnect:  0.000
      time_redirect:  0.000
   time_pretransfer:  0.227
 time_starttransfer:  0.443
                    ----------
         time_total:  0.867

从这个输出，我们可以算出各个步骤的时间：
- DNS 查询：12ms
- TCP 连接时间：pretransfter(227) - namelookup(12) = 215ms
- 服务器处理时间：starttransfter(443) - pretransfer(227) = 216ms
- 内容传输时间：total(867) - starttransfer(443) = 424ms


来个比较复杂的，访问某度首页，带有中间有重定向和 SSL 协议：
~ curl -w "@curl-format.txt" -o /dev/null -s -L "https://baidu.com"
    time_namelookup:  0.012
       time_connect:  0.018
    time_appconnect:  0.328
      time_redirect:  0.356
   time_pretransfer:  0.018
 time_starttransfer:  0.027
                    ----------
         time_total:  0.384
可以看到 `time_appconnect` 和 `time_redirect` 都不是 0 了，其中 SSL 协议处理时间为 `328-18=310ms`。而且 `pretransfer` 和 `starttransfer` 的时间都缩短了，这是重定向之后请求的时间。
```


# 设置curl的超时时间
## tcp 连接超时时间
“--connect-timeout”参数来指定tcp三次握手的连接超时时间
![](attachments/Pasted%20image%2020230619112247.png)
## 请求超时时间
“--max-time”参数来指定整个操作的最大超时时间。
![](attachments/Pasted%20image%2020230619112324.png)

## retry相关
![](attachments/Pasted%20image%2020230619113317.png)

## timeout 命令设置curl请求时长
```
timeout <option> <duration> <command>
```

## 使用场景
在写bash脚本时，如果脚本中使用了curl，如果curl没设置超时时间，发生异常时「比如断网」，有可能导致curl阻塞，影响后续命令的执行。

# 其他
```c
httpie 工具（是一个python写的类curl的命令行工具。使用python实现，其目标是让 CLI 和 web 服务之间的交互尽可能的人性化）：
	https://xin053.github.io/2016/08/15/httpie%E4%BA%BA%E6%80%A7%E5%8C%96curl%E5%B7%A5%E5%85%B7%E4%BD%BF%E7%94%A8%E8%AF%A6%E8%A7%A3/

```
# 参考
```c
httpstat:
https://linux.cn/article-8039-1.html
https://developer.aliyun.com/article/195250
https://blog.linux-code.com/articles/thread-1987.html
https://blog.smart-tools.cn/2023/01/13/%E4%BD%A0%E4%B8%8D%E7%9F%A5%E9%81%93%E7%9A%84%E7%A5%9E%E5%99%A8httpstat/
```
