```table-of-contents
```
# 相关问题
## `function used but not defined`问题

## 在动态链接库中使用static静态函数的问题
### 问题
我们知道static静态函数相比于全局函数，会把范围限定在自己所在文件的范围，对其他的编译单元是不可见的！
那么若在动态库中定义了static静态函数并完成库的编译，那么在主程序加载并试图调用这个static静态函数的时候会找不到它！或者 `objdump -t xx.so | grep xxx` 发现在编译生成的动态库中也是找不到这个`static` 函数的符号表。

因此需要在主程序中直接调用的函数在动态库中不要定义和声明为`static`类型！
# 参考
```bash

```