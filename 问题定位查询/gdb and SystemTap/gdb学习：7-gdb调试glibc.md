```table-of-contents
```
# 前言
glibc 这个 c 库，封装了很多代码，可以通过 gdb 调试进去深入理解底层源码。

## glibc 库作用
glibc 很多时候作为应用程序与内核之间交互的过渡角色，处理了很多源码工作细节。
![](attachments/Pasted%20image%2020250601124511.png)

# gdb调试
## gdb 调试效果
![](attachments/Pasted%20image%2020250601124526.png)

## 调试视频
视频链接：[gdb 调试 glibc (Centos)](https://www.zhihu.com/zvideo/1447448716474585088)

## 插件安装

（1）查看内核版本：
```bash
# uname -r
3.10.0-957.21.3.el7.x86_64
```

（2）配置 yum， 安装调试插件仓库路径。
```bash
# 设置镜像路径。
vim /etc/yum.repos.d/CentOS-Debuginfo.repo

# 填充下面内容。
#------------------------------
[base-debuginfo]
name=Centos-7 - Debuginfo
baseurl=http://debuginfo.centos.org/7/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-Debug-7
enable=1
#------------------------------
```

（3）安装相应的调试插件
`debuginfo-install`， 应该安装哪个版本的调试信息，如果不清楚，可以写测试代码通过 gdb 调试，缺少的插件，gdb 会提示，安装即可。
```text
yum install nss-softokn-debuginfo –nogpgcheck
yum install yum-utils gdb gcc-c++
debuginfo-install glibc-2.17-260.el7_6.6.x86_64 libgcc-4.8.5-44.el7.x86_64 libstdc++-4.8.5-44.el7.x86_64
```

## 测试源码
```c
/* gcc -g -O0 test.c -o test */
#include <stdio.h>
#include <string.h>

int main(int argc, char** argv) {
    char test[2];
    const char* p = "hello world";

    printf("%s", "snprintf: ");
    snprintf(test, sizeof(test), "%s", p);
    printf("%s\n", test);
}
```


# 参考
```bash

```