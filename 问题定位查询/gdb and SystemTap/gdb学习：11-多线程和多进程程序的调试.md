```table-of-contents
```

# 给指定线程设置断点
```bash
- `info threads`：查看所有线程（编号、状态、所属进程）。
- `thread [线程号]`：切换到指定线程。
- `break [位置] thread [线程号]`：给指定线程设断点。
- `set scheduler-locking on`：锁定当前线程执行（防止其他线程干扰，调试单线程逻辑时常用）。
- `thread apply [线程号] bt`：查看指定线程的调用栈（如 `thread apply all bt` 查看所有线程栈）。
```
# 参考
```bash

```