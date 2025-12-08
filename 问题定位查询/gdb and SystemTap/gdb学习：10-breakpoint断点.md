```table-of-contents
```
# 使用技巧
## 断点条件
### 函数名
### 文件名+行号
### 条件设置

## commands
**断点命令列表**，让==GDB在每次到达某一断点时自动执行一组命令==。

### 使用方法
```bash
commands breakpoint-number // breakpoint-number为断点号，表示将以下命令添加到指定的断点上  
...
commands // 任何有效的GDB命令，一行一个，以end结束  
...
c   
end 
```


# 参考
```bash
```