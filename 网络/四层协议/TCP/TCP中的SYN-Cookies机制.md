```table-of-contents
```
# 其他
## SYN-Cookie设置
SYN-Cookie 功能生效由内核 tcp_synkooie 确定：
- 值为 0 :始终不生效
- 值为 1 :一般情况下不生效，但当 listen sock 的 accept 队列满时(应用程序没有及时使用 accpet()) 生效。
- 值为 2 :始终生效
# 参考
```c
# 深入浅出TCP中的SYN-Cookies
https://switch-router.gitee.io/blog/TCP-SYN-Cookies/
```