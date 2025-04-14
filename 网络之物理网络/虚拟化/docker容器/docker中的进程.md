```table-of-contents
```
# 问答
## ps查看到的进程如何区分是宿主机进程还是docker中的进程
### 背景
在某些情况下，可能在宿主机上存在“看得到却摸不到”的进程；有的时候容器太多，想知道进程具体是哪个容器运行的？

### 范例1
首先在容器中的test目录下运行sleep 10000。

![](attachments/Pasted%20image%2020240425204929.png)

在宿主机ps能看到对应的进程：

![](attachments/Pasted%20image%2020240425204947.png)

看对应的proc下的cwd，也确实和容器中的路径一样，在/test目录下，但是宿主机实际上并没有这个路径

![](attachments/Pasted%20image%2020240425204957.png)

**方法1：ps -o cgroup**
大概率可以判断这个进程不是在宿主机上的，可以通过如下这个命令判断命令是否是在容器中执行的：
```bash
ps -e -o pid,cmd,comm,cgroup
```

![](attachments/Pasted%20image%2020240425205015.png)

可以看到pid为29656的cgroup是在docker里，且docker-xxxx，xxxx就是docker的id，也就是：`bf85501b3084601ba76b8cb303917134d58b5e7783c14c1636ff1c56a3d83c1f`。

**方法2：cat /proc/xxxx/cgroup **


### 范例2

![](attachments/Pasted%20image%2020240425205353.png)

已知，28116 是在宿主机上启动的程序。26748 是在 容器中启动的程序。

**方法3: ps -auxf 查看祖先进程**
其实通过ps -auxf 查看祖先进程大概也是可以看出来的。

![](attachments/Pasted%20image%2020240425205703.png)


如上，得到 进程所在的 docker id 之后，也可以看这个 docker 中的 进程信息。

![](attachments/Pasted%20image%2020240425210054.png)

# 参考
```bash
# docker(3):容器中的进程 [朱双印]
https://www.zsythink.net/archives/4321

```