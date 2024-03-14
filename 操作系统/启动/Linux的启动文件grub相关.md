```table-of-contents
```
# 修改内核启动参数
- 修改`/etc/default/grub`文件；
- 生成新的`grub`文件
```
grub2-mkconfig -o /boot/grub2/grub.cfg
```
# grubby命令
## 介绍
grubby是一个命令行工具，用于管理Linux系统的引导选项，可以用来添加、删除、编辑和列出内核引导参数。
![](attachments/Pasted%20image%2020240314181435.png)

## 常见的grubby命令
(1) 列出当前系统上已安装的内核：
```bash
grubby --info=ALL

列出指定内核的详细信息:
grubby --info=/boot/vmlinuz-5.10.0-327.el7.x86_64

or

grubby --info=0 // index=0的内核的详细信息

```

(2) 列出当前系统默认启动的内核
```bash
grubby --default-kernel
```

(3) 修改默认启动内核
```bash
grubby --set-default=/boot/vmlinuz-5.10.0-327.el7.x86_64

or

grubby --set-default=1 // 使用index来指定内核
```

(4) 添加内核引导参数
```bash
grubby --update-kernel=/boot/vmlinuz-5.10.0-327.el7.x86_64 --args="nomodeset"

设置内核引导参数:
grubby --update-kernel=/boot/vmlinuz-5.10.0-327.el7.x86_64 --args="console=tty0 console=ttyS0,115200"
```

(5) 删除内核引导参数
```bash
grubby --update-kernel=/boot/vmlinuz-5.10.0-327.el7.x86_64 --remove-args="nomodeset"
```

# 选择启动内核
如果`grub.conf`中存在多个内核，期望选择一个指定的内核版本进行启动。

## **查看存在的内核版本**
```bash
[root@host ~]# awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux 7 (Core), with Linux 3.10.0-229.14.1.el7.x86_64
1 : CentOS Linux 7 (Core), with Linux 3.10.0-229.4.2.el7.x86_64
2 : CentOS Linux 7 (Core), with Linux 3.10.0-229.el7.x86_64
3 : CentOS Linux 7 (Core), with Linux 0-rescue-605f01abef434fb98dd1309e774b72ba
```
或者
```bash
grep "^menuentry" /boot/grub2/grub.cfg | cut -d "'" -f2

or

cat /boot/grub2/grub.cfg | grep menuentry
```

**/etc/grub2.cfg** 这个文件名指向 `grub.cfg`，而它的位置视乎结构而定，一般是`/boot/grub2/grub.cfg` 。
访问文件时采用绝对路径是较佳的做法，在维修系统时更甚。缺省的项目是通过 **/etc/default/grub** 档内的 **GRUB_DEFAULT** 行来定义。不过，要是 **GRUB_DEFAULT** 行被设置为 **saved**，这个选项便存储在 **/boot/grub2/grubenv** 档内。

## **查看当前的默认启动内核**
```
grub2-editenv list

or

cat /boot/grub2/grubenv
```

## 设定要启动的内核版本
```bash
（1）设定
grub2-set-default 'CentOS Linux (3.10.0-123.9.3.el7.x86_64) 7 (Core)'
注：'CentOS Linux (3.10.0-123.9.3.el7.x86_64) 7 (Core)' 即为 `cat /boot/grub2/grub.cfg | grep menuentry` 中的输出的一个。

或

grub2-set-default 0~N 
注：`grep "^menuentry" /boot/grub2/grub.cfg | cut -d "'" -f2` 的输出从0开始计数，选择要启动的内核版本；比如：grub2-set-default 2



（2）校验
grub2-editenv list
```

**/etc/default/grub** 档内另一个有用的选项是：`GRUB_SAVEDEFAULT=true`，
连同 **GRUB_DEFAULT=saved**，它确保现时选择的开机项目会被设置下次开机采用。


# 参考
```c
# 在 CentOS 7 上设置 grub2
https://wiki.centos.org/zh(2f)HowTos(2f)Grub2.html
```