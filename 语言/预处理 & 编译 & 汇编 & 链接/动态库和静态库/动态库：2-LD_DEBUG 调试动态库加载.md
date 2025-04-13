```table-of-contents
```
# 介绍
LD_DEBUG 调试动态库加载。
# 使用
##  查看命令使用方法
`LD_DEBUG=help  ./your_program`
范例如下：
```bash
# LD_DEBUG=help ls
Valid options for the LD_DEBUG environment variable are:

  libs        display library search paths
  reloc       display relocation processing
  files       display progress for input file
  symbols     display symbol table processing
  bindings    display information about symbol binding
  versions    display version dependencies
  scopes      display scope information
  all         all previous options combined
  statistics  display relocation statistics
  unused      determined unused DSOs
  help        display this help message and exit

To direct the debugging output into a file instead of standard output
a filename can be specified using the LD_DEBUG_OUTPUT environment variable.
```

## 查看依赖的库的查找过程
`LD_DEBUG=libs ./your_program`

范例
```bash
LD_DEBUG=libs ./main  
      3557:	find library=libtest.so [0]; searching  
      3557:	 search cache=/etc/ld.so.cache  
      3557:	  trying file=/usr/local/lib/libtest.so
```

![](attachments/Pasted%20image%2020240305121741.png)
```bash
执行 find /usr -name libevent-1.4.so.2 
得知libevnet=1.4.so.2已经安装，但是不在默认共享库的查找路径下.
库路径在该目录下：/usr/local/lib/libevent-1.4.so.2
```
## 显示符号的查找过程
```bash
LD_DEBUG=symbols ./main
```
# 参考
```bash

```