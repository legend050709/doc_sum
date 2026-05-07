```table-of-contents
```

# 性能差的函数
## random
random函数存在加锁，会导致性能问题，如下所示。另外不要使用 lrand48（非线程安全，多线程读写存在缓存失效的情况）；建议使用 lrand48_r，其是线程安全且无锁的。

![](attachments/image%20(28).png)

lrand48存在读写全局变量/静态变量，多线程同时调用lrand48，存在并发更改全局变量/静态变量，会导致其他core中的缓存无效。对于缓存一致性的原理，可参见: [缓存一致性 MESI 协议原理演示](https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESI.htm)。

## 常见的线程不安全函数
下图为常见的线程不安全函数，在使用之前线程不安全函数之前需要在安全性以及性能上进行考虑是否存在问题。

![](attachments/image%20(29).png)

### gethostbyaddr
### gethostbyname
### strerror
### ctime


# 参考
```bash
# POSIX线程不安全函数
https://kimi.pub/506.html

# 缓存一致性 MESI 协议原理演示
https://www.scss.tcd.ie/Jeremy.Jones/VivioJS/caches/MESI.htm
```