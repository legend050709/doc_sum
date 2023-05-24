# 背景
在脚本中调用一些系统命令，有可能由于某些原因出现无响应情况。脚本该如何判断调用命令是否超时，并且在超时的时候终止掉调用并继续执行以避免hang住呢？

# 使用
```c
使用：
# timeout <option> <duration> <command>

参数：
-k 当达到命令结束的时间没有结束时，再经过指定时间后结束命令；
-s 或者 –signal=，在超时时发送的信号；

```

![](attachments/Pasted%20image%2020230619114313.png)

# 场景范例
先写一个简单地死循环脚本`test.sh`：
```
while [ 1 -gt 0 ];do
    echo test > /dev/null
done
```
然后使用以下命令设置10秒超时返回：

```
timeout 10s sh test.sh
```

- 如何判断是超时呢？
`timeout`对于超时强制结束的指令返回的返回码是`124`，可以通过输出`echo $?`来获取返回码：
```
$echo $?
124
```

# 参考