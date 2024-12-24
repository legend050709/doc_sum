```table-of-contents
```
# 使用场景
## 查看内存变化
```c
# pidstat -r -p 79824 2 4
Linux 4.18.0-2.4.3.3.kwai.x86_64 (bjrz-k536.lf) 	10/31/2023 	_x86_64_	(80 CPU)

11:48:13 AM   UID       PID  minflt/s  majflt/s     VSZ    RSS   %MEM  Command
11:48:15 AM     0     79824      0.00      0.00 274343720 146164   0.06  dpvs
11:48:17 AM     0     79824      0.00      0.00 274343720 146164   0.06  dpvs
11:48:19 AM     0     79824      0.00      0.00 274343720 146164   0.06  dpvs
11:48:21 AM     0     79824      0.00      0.00 274343720 146164   0.06  dpvs
Average:        0     79824      0.00      0.00 274343720 146164   0.06  dpvs
```

## IO使用情况
### 系统所有进程的IO使用情况
```bash
# 间隔 1 秒输出多组数据 (这里是 20 组)
$ pidstat -d 1 20
```
![](attachments/Pasted%20image%2020241218141547.png)
### 指定进程的IO使用情况
```bash
# -d 展示 I/O 统计数据，-p 指定进程号，间隔 1 秒输出 3 组数据
$ pidstat -d -p XXX 1 3
```
![](attachments/Pasted%20image%2020241218141543.png)

# 参考
```c

```