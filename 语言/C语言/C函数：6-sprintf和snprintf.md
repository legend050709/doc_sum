```table-of-contents
```

# sprintf
## sprintf的缺陷
```c
char buffer[10]; 
sprintf(buffer, "%s", "This is a very long string"); // 风险：缓冲区溢出
```
- **无长度限制**：不检查目标缓冲区的大小，直接将格式化后的字符串写入目标缓冲区。
- **缓冲区溢出风险**：如果格式化后的字符串长度超过缓冲区大小，会覆盖相邻内存，导致崩溃或安全漏洞。

# snprintf
## `snprintf` 的安全优势
```c
char buffer[10]; 
int needed = snprintf(buffer, sizeof(buffer), "%s", "This is a long string");
```
- **显式指定缓冲区大小**：第二个参数指明缓冲区的最大容量（包括结尾的 `\0` 字符）。
- **自动截断**：若格式化后的字符串超出缓冲区大小，`snprintf` 会截断内容并确保字符串以 `\0` 结尾。
- **返回实际需要的长度**：返回值是格式化后的字符串长度（不包含 `\0`），即使截断也会返回完整长度。可通过此值判断是否需要扩大缓冲区。

## 安全使用 `snprintf` 的最佳实践
### 始终指定缓冲区大小
```c
char buffer[256]; 
snprintf(buffer, sizeof(buffer), "User: %s, Age: %d", username, age);
```

- 使用 `sizeof(buffer)` 而非硬编码数字，避免缓冲区大小变化时出错。

### 检查返回值
```c
int needed = snprintf(buffer, sizeof(buffer), "...");
if (needed >= sizeof(buffer)) {
    // 缓冲区不足，需要动态扩容
    char *new_buf = malloc(needed + 1);
    snprintf(new_buf, needed + 1, "...");
    // 使用 new_buf ...
    free(new_buf);
}
```
- 如果返回值 `>= 缓冲区大小`，说明原缓冲区太小，需要扩容。

### 处理字符串终止符
- `snprintf` 保证在缓冲区非空时，结果字符串一定以 `\0` 结尾。


# 总结
在 Linux 的 C/C++ 开发中，**`snprintf` 比 `sprintf` 更安全**，因为它明确限制了缓冲区的大小，从而避免缓冲区溢出（Buffer Overflow）这一经典安全漏洞。

# 参考
```bash

```