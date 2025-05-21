```table-of-contents
```
# 前置条件
首先，编译动态库时指定 -g 选项，以启用 GDB 调试信息
```bash
gcc -shared -fPIC -g -o libhello.so hello.c
```

# 查看程序已加载的动态库
```bash
./gdb-11.2 -D share_gdb/ -p 29342 // 启动gdb调试
(gdb) info sharedlibrary  // 查看已经记载的动态库
```
# 参考
```c

```