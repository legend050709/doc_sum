```table-of-contents
```


# namespace号

## 在容器中，一个进程的 `net` namespace 号 和 `pid` namespace 号 会一样吗？
不会一样。它们一定是**不同的 namespace 实例**，对应不同类型的隔离域。

即使是同一个容器中的同一个进程，它的：
- `net:[4026532xxx]`
- `pid:[4026532yyy]`  
这两个数字也不会相同。


**分析**
Linux 的 namespace 机制本质上是：
> 针对不同的系统资源，创建不同类型的“命名空间对象”。
每种 namespace 类型都独立存在，有自己的标识（ID）和生命周期。

# 进程的pid
宿主机上不同容器内的进程，其在容器内部看到的 PID 可能相同，但在宿主机上一定不同。
换句话说：
- 容器内看到的 **PID 是容器内的（相对）PID**（namespace 内部编号）；
- 宿主机上看到的 **PID 是全局唯一的（系统 PID）**。
- **不同容器内部的 PID 可以相同**，但宿主机上看它们一定是不同的进程号。

```bash
宿主机 PID namespace（root PID NS）
   ├── 容器A 的 PID namespace
   └── 容器B 的 PID namespace
```

每个容器都有独立的 PID 计数器，从 1 开始。  
所以在容器中：
- 第一个用户进程（通常是 `/sbin/init` 或 `/bin/bash`）的 PID 是 1；
- 后续进程从 2、3、4... 开始分配。

但宿主机上看到的它们，PID 是不同的、全局唯一的。


## 已知容器的 namespace（PID ns）以及容器内的进程号（PID），如何找到该进程在宿主机上的真实 PID

### C范例
```c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <string.h>
#include <errno.h>
#include <sys/types.h>
#include <sys/stat.h>

static unsigned long get_ns_id(const char *ns_type)
{
    char path[128], buf[256];
    snprintf(path, sizeof(path), "/proc/self/ns/%s", ns_type);
    ssize_t len = readlink(path, buf, sizeof(buf) - 1);
    if (len == -1) {
        fprintf(stderr, "Failed to read %s: %s\n", path, strerror(errno));
        return 0;
    }
    buf[len] = '\0';

    unsigned long id = 0;
    sscanf(buf, "%*[^[][%lu]", &id);
    return id;
}

static unsigned long get_host_pid(void)
{
    // /proc/self is symlink to /proc/<host_pid> even inside container
    char path[64];
    ssize_t len = readlink("/proc/self", path, sizeof(path) - 1);
    if (len == -1) {
        perror("readlink /proc/self");
        return 0;
    }
    path[len] = '\0';
    unsigned long host_pid = 0;
    sscanf(path, "%*[^0-9]%lu", &host_pid);
    return host_pid;
}

int main(void)
{
    const char *namespaces[] = {"cgroup", "ipc", "mnt", "net", "pid", "user", "uts"};
    size_t n = sizeof(namespaces)/sizeof(namespaces[0]);

    pid_t self_pid = getpid();
    unsigned long host_pid = get_host_pid();

    printf("=== Process Information ===\n");
    printf("Container-view PID : %d\n", self_pid);
    printf("Host-view PID      : %lu\n", host_pid);
    printf("\n=== Namespace IDs ===\n");

    for (size_t i = 0; i < n; i++) {
        unsigned long id = get_ns_id(namespaces[i]);
        if (id)
            printf("%-8s : %lu\n", namespaces[i], id);
    }

    return 0;
}

```

- `/proc/self/ns/<type>` 是指向 namespace 的符号链接，  
    其形如 `"net:[4026532008]"` 的数字部分即 namespace ID。

- `/proc/self` 在宿主机上是 `/proc/<pid>`，  
    在容器中是指向宿主机上真实 PID 的符号链接；


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

可以看到pid为29656的cgroup是在docker里，且`docker-xxxx`，`xxxx`就是`docker`的id，也就是：`bf85501b3084601ba76b8cb303917134d58b5e7783c14c1636ff1c56a3d83c1f`。

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