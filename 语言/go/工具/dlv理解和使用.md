```table-of-contents
```
# 使用
## 输出到文件
**背景**
某个dlv命令输出太多，标准输出放不下；

**使用**
```bash
(dlv) transcript -x /home/xxx/debug/sources
(dlv) goroutines -t 20 
(dlv) transcript -off
```
其中-x是标准输出就不打印了，直接输出到文件中；
收集完数据后，记得 `transcript -off`关掉

## 输出字符串限制
通过`config -list` 可以查看可以配置的选项，其中 `max-string-len`可以配置输出的最大字符串长度.

# 参考
```bash

```