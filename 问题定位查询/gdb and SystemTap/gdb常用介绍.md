```table-of-contents
```
# 符号表的剥离和导回
## 背景
遇到了版本过大，超过了硬件限制的问题
## 思路
将版本所有的动态库（内部+三方动态库）的gdb调试辅助信息进行剥离，同时发布对应的debug包。如果需要使用gdb进行调试时，取用相应版本的debug包，解压放入设备固定目录，该目录由设备启动脚本修改.gdbinit文件，设置gdb寻找符号表文件的目录。
## 优缺点
这个方案下来，优势是相比于裁剪库、优化压缩工具、编译规则来说，版本包可以大幅度缩减50%，同时发布版本隐藏了代码的相关本地变量等信息，增加了安全项。
缺点是后期维护调试比较麻烦，需要获取同期发布的debug包，并下载解压到设备。
## Debug版本与Release版本
项目版本有Debug版本与Release版本之分。
一般而言，Release版本在编译规则中做了代码优化，在性能上往往更强一点，同时也是集成测试的测试版本。
> 正常来讲Release版本往往比Debug要小，是因为：Release版本剥离了调试信息、符号表等。
## 调试信息剥离
剥离工具有很多：objcopy、strip、eu-strip等，各有各的优势。

# 常见问题
## `No symbol table is loaded. Use the "file" command`
- 问题
使用GDB附件进程出现如下提示：No symbol table is loaded. Use the “file” command.

- 原因
该可执行文件是以release版（一般软件进行发布，用户体验模式）的方式来进行发布的。
release 版本，即编译的时候没有添加 `-g` 选项。或者 编译的时候含有 `-g` 选项，但是发布的时候将可执行文件通过 `strip -s xxxx` 去掉了 debug 符号表.


通过 file BIN 来查看可执行文件是否含有 debug 标签。
比如：可执行文件是`/bin/ipmitool`。
```c
(gdb) file /bin/ipmitool
Reading symbols from /bin/ipmitool...
Reading symbols from .gnu_debugdata for /usr/bin/ipmitool...
(No debugging symbols found in .gnu_debugdata for /usr/bin/ipmitool)
```


# 参考
```c

```