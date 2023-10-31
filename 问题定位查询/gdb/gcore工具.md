```table-of-contents
```
# 介绍
gcore是原生程序的内存转储工具，可以将程序的整个内存空间转储为文件，转储出来的dump文件，可以直接使用gdb进行调试。
```c
# 其中10235是进程的pid  
$ gcore -o coredump 10235  
  
$ ll coredump*  
-rw-r--r-- 1 work work 3.7G 2021-11-07 23:05:46 coredump.10235
```
# 参考
```c

```