```table-of-contents
```

# libpthread.so
## 安装
```bash
yum install glibc-devel -y
```

包中的文件查看, 如下所示，==`glibc-devel`包中存在`libpthread.so`, 不存在`libpthread.a`==.

```bash
# rpm -ql glibc-devel
/usr/include/gnu/stubs-64.h
/usr/lib64/Mcrt1.o
/usr/lib64/Scrt1.o
/usr/lib64/crt1.o
/usr/lib64/crti.o
/usr/lib64/crtn.o
/usr/lib64/gcrt1.o
/usr/lib64/libBrokenLocale.so
/usr/lib64/libanl.so
/usr/lib64/libbsd-compat.a
/usr/lib64/libbsd.a
/usr/lib64/libc.so
/usr/lib64/libc_nonshared.a
/usr/lib64/libcidn.so
/usr/lib64/libcrypt.so
/usr/lib64/libdl.so
/usr/lib64/libg.a
/usr/lib64/libieee.a
/usr/lib64/libm.so
/usr/lib64/libmcheck.a
/usr/lib64/libnsl.so
/usr/lib64/libnss_compat.so
/usr/lib64/libnss_db.so
/usr/lib64/libnss_dns.so
/usr/lib64/libnss_files.so
/usr/lib64/libnss_hesiod.so
/usr/lib64/libnss_nis.so
/usr/lib64/libnss_nisplus.so
/usr/lib64/libpthread.so
/usr/lib64/libpthread_nonshared.a
/usr/lib64/libresolv.so
/usr/lib64/librpcsvc.a
/usr/lib64/librt.so
/usr/lib64/libthread_db.so
/usr/lib64/libutil.so
/usr/share/info/libc.info-1.gz
/usr/share/info/libc.info-10.gz
/usr/share/info/libc.info-11.gz
/usr/share/info/libc.info-12.gz
/usr/share/info/libc.info-13.gz
/usr/share/info/libc.info-14.gz
/usr/share/info/libc.info-2.gz
/usr/share/info/libc.info-3.gz
/usr/share/info/libc.info-4.gz
/usr/share/info/libc.info-5.gz
/usr/share/info/libc.info-6.gz
/usr/share/info/libc.info-7.gz
/usr/share/info/libc.info-8.gz
/usr/share/info/libc.info-9.gz
/usr/share/info/libc.info.gz
```

## 查找
```bash
find / -type f -name libpthread.a
```

# libpthread.a
## 安装
```bash
yum install glibc-static -y
```

包中的文件查看, 如下所示：
```bash
# rpm -ql glibc-static
/usr/lib64/libBrokenLocale.a
/usr/lib64/libanl.a
/usr/lib64/libc.a
/usr/lib64/libc_stubs.a
/usr/lib64/libcrypt.a
/usr/lib64/libdl.a
/usr/lib64/libm.a
/usr/lib64/libnsl.a
/usr/lib64/libpthread.a
/usr/lib64/libresolv.a
/usr/lib64/librt.a
/usr/lib64/libutil.a
```

# sched.h 文件

## 安装

```bash
yum install -y glibc-headers
```
查看 `glibc-headers` 下的头文件：
```bash
rpm -ql glibc-headers
```

## 使用
一般 `#include <sched.h>`之后，可能依然出错。
```bash
#ifndef _GNU_SOURCE
#define _GNU_SOURCE
#endif
#include <sched.h>
#include <pthread.h>
```

# 参考
```bash

```