```table-of-contents
```
# errno简介
Linux 中**系统调用函数/库函数**发生错误。
**errno** 就会被自动设置为一个特定的正值，用来反馈具体错误，而函数本身则会返回 `-1` 或者 `NULL`「-1 from most system calls; -1 or NULL from most library functions」 ，以说明函数发生了错误；
如果函数返回`0`或`正值`，即没有错误发生，则 errno 没有定义。

errno 是一个包含在 <errno.h> 中的预定义的外部 int 变量，用于表示最近一个函数调用是否产生了错误。
- 若为0，则无错误；
- 其它值均表示某一种错误。

代码中需要包含 `#include <errno.h>`，当一个系统调用或着库函数的调用失败时，将会重置错误码 errno。用户在判断程序出错后，**立即**检验 errno 的值可以获取错误码和错误信息。


# errno列表
## base-errno
```c
# cat /usr/include/asm-generic/errno-base.h
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _ASM_GENERIC_ERRNO_BASE_H
#define _ASM_GENERIC_ERRNO_BASE_H

#define	EPERM		 1	/* Operation not permitted */
#define	ENOENT		 2	/* No such file or directory */
#define	ESRCH		 3	/* No such process */
#define	EINTR		 4	/* Interrupted system call */
#define	EIO		 5	/* I/O error */
#define	ENXIO		 6	/* No such device or address */
#define	E2BIG		 7	/* Argument list too long */
#define	ENOEXEC		 8	/* Exec format error */
#define	EBADF		 9	/* Bad file number */
#define	ECHILD		10	/* No child processes */
#define	EAGAIN		11	/* Try again */
#define	ENOMEM		12	/* Out of memory */
#define	EACCES		13	/* Permission denied */
#define	EFAULT		14	/* Bad address */
#define	ENOTBLK		15	/* Block device required */
#define	EBUSY		16	/* Device or resource busy */
#define	EEXIST		17	/* File exists */
#define	EXDEV		18	/* Cross-device link */
#define	ENODEV		19	/* No such device */
#define	ENOTDIR		20	/* Not a directory */
#define	EISDIR		21	/* Is a directory */
#define	EINVAL		22	/* Invalid argument */
#define	ENFILE		23	/* File table overflow */
#define	EMFILE		24	/* Too many open files */
#define	ENOTTY		25	/* Not a typewriter */
#define	ETXTBSY		26	/* Text file busy */
#define	EFBIG		27	/* File too large */
#define	ENOSPC		28	/* No space left on device */
#define	ESPIPE		29	/* Illegal seek */
#define	EROFS		30	/* Read-only file system */
#define	EMLINK		31	/* Too many links */
#define	EPIPE		32	/* Broken pipe */
#define	EDOM		33	/* Math argument out of domain of func */
#define	ERANGE		34	/* Math result not representable */

#endif
```

## 扩展的errno

```c
# cat /usr/include/asm-generic/errno.h
/* SPDX-License-Identifier: GPL-2.0 WITH Linux-syscall-note */
#ifndef _ASM_GENERIC_ERRNO_H
#define _ASM_GENERIC_ERRNO_H

#include <asm-generic/errno-base.h>

#define	EDEADLK		35	/* Resource deadlock would occur */
#define	ENAMETOOLONG	36	/* File name too long */
#define	ENOLCK		37	/* No record locks available */

/*
 * This error code is special: arch syscall entry code will return
 * -ENOSYS if users try to call a syscall that doesn't exist.  To keep
 * failures of syscalls that really do exist distinguishable from
 * failures due to attempts to use a nonexistent syscall, syscall
 * implementations should refrain from returning -ENOSYS.
 */
#define	ENOSYS		38	/* Invalid system call number */

#define	ENOTEMPTY	39	/* Directory not empty */
#define	ELOOP		40	/* Too many symbolic links encountered */
#define	EWOULDBLOCK	EAGAIN	/* Operation would block */
#define	ENOMSG		42	/* No message of desired type */
#define	EIDRM		43	/* Identifier removed */
#define	ECHRNG		44	/* Channel number out of range */
#define	EL2NSYNC	45	/* Level 2 not synchronized */
#define	EL3HLT		46	/* Level 3 halted */
#define	EL3RST		47	/* Level 3 reset */
#define	ELNRNG		48	/* Link number out of range */
#define	EUNATCH		49	/* Protocol driver not attached */
#define	ENOCSI		50	/* No CSI structure available */
#define	EL2HLT		51	/* Level 2 halted */
#define	EBADE		52	/* Invalid exchange */
#define	EBADR		53	/* Invalid request descriptor */
#define	EXFULL		54	/* Exchange full */
#define	ENOANO		55	/* No anode */
#define	EBADRQC		56	/* Invalid request code */
#define	EBADSLT		57	/* Invalid slot */

#define	EDEADLOCK	EDEADLK

#define	EBFONT		59	/* Bad font file format */
#define	ENOSTR		60	/* Device not a stream */
#define	ENODATA		61	/* No data available */
#define	ETIME		62	/* Timer expired */
#define	ENOSR		63	/* Out of streams resources */
#define	ENONET		64	/* Machine is not on the network */
#define	ENOPKG		65	/* Package not installed */
#define	EREMOTE		66	/* Object is remote */
#define	ENOLINK		67	/* Link has been severed */
#define	EADV		68	/* Advertise error */
#define	ESRMNT		69	/* Srmount error */
#define	ECOMM		70	/* Communication error on send */
#define	EPROTO		71	/* Protocol error */
#define	EMULTIHOP	72	/* Multihop attempted */
#define	EDOTDOT		73	/* RFS specific error */
#define	EBADMSG		74	/* Not a data message */
#define	EOVERFLOW	75	/* Value too large for defined data type */
#define	ENOTUNIQ	76	/* Name not unique on network */
#define	EBADFD		77	/* File descriptor in bad state */
#define	EREMCHG		78	/* Remote address changed */
#define	ELIBACC		79	/* Can not access a needed shared library */
#define	ELIBBAD		80	/* Accessing a corrupted shared library */
#define	ELIBSCN		81	/* .lib section in a.out corrupted */
#define	ELIBMAX		82	/* Attempting to link in too many shared libraries */
#define	ELIBEXEC	83	/* Cannot exec a shared library directly */
#define	EILSEQ		84	/* Illegal byte sequence */
#define	ERESTART	85	/* Interrupted system call should be restarted */
#define	ESTRPIPE	86	/* Streams pipe error */
#define	EUSERS		87	/* Too many users */
#define	ENOTSOCK	88	/* Socket operation on non-socket */
#define	EDESTADDRREQ	89	/* Destination address required */
#define	EMSGSIZE	90	/* Message too long */
#define	EPROTOTYPE	91	/* Protocol wrong type for socket */
#define	ENOPROTOOPT	92	/* Protocol not available */
#define	EPROTONOSUPPORT	93	/* Protocol not supported */
#define	ESOCKTNOSUPPORT	94	/* Socket type not supported */
#define	EOPNOTSUPP	95	/* Operation not supported on transport endpoint */
#define	EPFNOSUPPORT	96	/* Protocol family not supported */
#define	EAFNOSUPPORT	97	/* Address family not supported by protocol */
#define	EADDRINUSE	98	/* Address already in use */
#define	EADDRNOTAVAIL	99	/* Cannot assign requested address */
#define	ENETDOWN	100	/* Network is down */
#define	ENETUNREACH	101	/* Network is unreachable */
#define	ENETRESET	102	/* Network dropped connection because of reset */
#define	ECONNABORTED	103	/* Software caused connection abort */
#define	ECONNRESET	104	/* Connection reset by peer */
#define	ENOBUFS		105	/* No buffer space available */
#define	EISCONN		106	/* Transport endpoint is already connected */
#define	ENOTCONN	107	/* Transport endpoint is not connected */
#define	ESHUTDOWN	108	/* Cannot send after transport endpoint shutdown */
#define	ETOOMANYREFS	109	/* Too many references: cannot splice */
#define	ETIMEDOUT	110	/* Connection timed out */
#define	ECONNREFUSED	111	/* Connection refused */
#define	EHOSTDOWN	112	/* Host is down */
#define	EHOSTUNREACH	113	/* No route to host */
#define	EALREADY	114	/* Operation already in progress */
#define	EINPROGRESS	115	/* Operation now in progress */
#define	ESTALE		116	/* Stale file handle */
#define	EUCLEAN		117	/* Structure needs cleaning */
#define	ENOTNAM		118	/* Not a XENIX named type file */
#define	ENAVAIL		119	/* No XENIX semaphores available */
#define	EISNAM		120	/* Is a named type file */
#define	EREMOTEIO	121	/* Remote I/O error */
#define	EDQUOT		122	/* Quota exceeded */

#define	ENOMEDIUM	123	/* No medium found */
#define	EMEDIUMTYPE	124	/* Wrong medium type */
#define	ECANCELED	125	/* Operation Canceled */
#define	ENOKEY		126	/* Required key not available */
#define	EKEYEXPIRED	127	/* Key has expired */
#define	EKEYREVOKED	128	/* Key has been revoked */
#define	EKEYREJECTED	129	/* Key was rejected by service */

/* for robust mutexes */
#define	EOWNERDEAD	130	/* Owner died */
#define	ENOTRECOVERABLE	131	/* State not recoverable */

#define ERFKILL		132	/* Operation not possible due to RF-kill */

#define EHWPOISON	133	/* Memory page has hardware error */

#endif
```

## `bits/errno.h`
```c
# cat /usr/include/bits/errno.h

#ifdef _ERRNO_H

# undef EDOM
# undef EILSEQ
# undef ERANGE
# include <linux/errno.h>

/* Linux has no ENOTSUP error code.  */
# define ENOTSUP EOPNOTSUPP

/* Older Linux versions also had no ECANCELED error code.  */
# ifndef ECANCELED
#  define ECANCELED 125
# endif

/* Support for error codes to support robust mutexes was added later, too.  */
# ifndef EOWNERDEAD
#  define EOWNERDEAD        130
#  define ENOTRECOVERABLE   131
# endif

# ifndef ERFKILL
#  define ERFKILL       132
# endif

# ifndef EHWPOISON
#  define EHWPOISON     133
# endif

# ifndef __ASSEMBLER__
/* Function to get address of global `errno' variable.  */
extern int *__errno_location (void) __THROW __attribute__ ((__const__));

#  if !defined _LIBC || defined _LIBC_REENTRANT
/* When using threads, errno is a per-thread value.  */
#   define errno (*__errno_location ())
#  endif
# endif /* !__ASSEMBLER__ */
#endif /* _ERRNO_H */

#if !defined _ERRNO_H && defined __need_Emath
/* This is ugly but the kernel header is not clean enough.  We must
   define only the values EDOM, EILSEQ and ERANGE in case __need_Emath is
   defined.  */
# define EDOM   33  /* Math argument out of domain of function.  */
# define EILSEQ 84  /* Illegal byte sequence.  */
# define ERANGE 34  /* Math result not representable.  */
#endif /* !_ERRNO_H && __need_Emath */
```

# errno的使用
## errno的取值

![](attachments/Pasted%20image%2020250524135157.png)

- 如果系统调用函数没有出错，则之前的 errno 可能不会被清除（未定义）；只有发生错误时，才会覆盖之前的错误。
- errno 不会为 0，这与多线程的 errno 处理有关。

## errno的线程安全
虽然 errno 是一个全局变量，但是在多线程环境中，每个线程都会有自己独立的 errno 副本，这是通过线程本地存储（Thread-Local Storage，TLS）实现的，因此一般不用担心多线程会相互竞争 errno。

![](attachments/Pasted%20image%2020250524135734.png)

## errno的覆盖与恢复
**虽然在多线程下不用担心 errno 的竞争问题，不过单线程下 errno 仍可能出现问题，比如在信号处理函数中被修改** 。

当发生信号时，执行流会跳转到信号处理函数，这感觉就像是多线程，但实际上它和之前的执行流位于同一个上下文，也就是说信号处理函数并不是新开的线程。
因此，如果之前的执行流在系统调用出错后修改 errno，接着被信号中断，进入了信号处理函数，而信号处理函数中也调用了某些系统函数如 write，如果此时这个系统调用出错，那么 errno 就会被修改，**这就使得之前的 errno 被覆盖！**

因此，==**作为一个通用的规则：** 在信号处理函数中，应首先保存 errno，退出时再恢复==。
```c
void sig_alarm(int signo)
{
    int errno_cpy = errno;
    //do something...
    write(....);
    errno = errno_cpy;
}
```

## 线程函数的返回值以及`errno`
![](attachments/Pasted%20image%2020250524140938.png)

**线程函数（以 pthread_ 开头的函数）遇到错误时不会设置标准 Unix 的 errno 变量，而是将 errno 的值以函数返回值的形式交给调用者**。
也就是说，返回值大于 0 则说明发生了错误，那没有发生错误呢？自然也就是返回 0 了。

```c
int n;
if ((n = pthread_mutex_lock(&ndoen_mutex)) != 0){
    fprintf(stderr,"error:%s",strerror(n));
}

```
你看，这意味着我们每次调用 `pthread_` 函数时，都要事先分配一个整形来保存错误值，这很麻烦，所以我们可以把错误处理和 pthread_ 函数包裹起来以简化代码：

![](attachments/Pasted%20image%2020250525131921.png)



# 包裹函数
## 背景
当我们在讨论线程时，将会发现线程函数遇到错误时并不设置标准`Unix`的`errno`变量，而是把`errno`的值作为函数返回值返回调用者。这意味着每次调用以`pthread_`开头的某个函数时，我们必须分配一个变量来存放函数返回值，以便在调用`err_sys`前把`errno`变量设置成该值。
## 简介
包裹函数（wrapper function）其实就是封装函数，调用一个函数来实现这个功能，但是我们通常不在这个函数里面来定义它，只是调用。
可以使用一种规定来约定包裹函数例如：==将包裹函数名第一个字母大写，它调用的实际函数的名字与包裹函数名相同==，不过以对应的小写字母开头。

## 包裹函数范例
例如，在语句`sockfd = Socket(AF_INET, SOCK_STREAM, 0);`中，函数`Socket`是函数`socket`的包裹函数，如下代码所示：
```c
/* include Socket */
int
Socket(int family, int type, int protocol)
{
    int n;
    if ( (n = socket(family, type, protocol)) < 0)
        err_sys("socket error");
    return(n);
}
```

```c
/* include Pthread_mutex_lock */
void
Pthread_mutex_lock(pthread_mutex_t *mptr)
{
    int        n;
    if ( (n = pthread_mutex_lock(mptr)) == 0)
        return;
    errno = n;
    err_sys("pthread_mutex_lock error");
}
/* end Pthread_mutex_lock */
```

包裹函数一般用来处理致命性错误，此时能干的也就只有打印错误然后退出，对于非致命性错误，比如 EINTR 错误，就需要我们自己来处理失败情况：
```c
while(1){
    if((sock_conn=accept(sock_listen,NULL,NULL))<0){
        if(errno==EINTR)
            continue;
        else
            perror("accept");
    }
}
```

## 包裹函数和宏
要是仔细推敲C代码的编写，我们可以用宏来替代函数，从而稍微提高运行时效率，不过包裹函数很少是程序性能的瓶颈所在。

## 其他包裹函数
其他包裹函数可参考[UNP-sockwarp.c](https://github.com/jyx-fyh/unp-source-code/blob/master/lib/wrapsock.c)

(1) `socket`相关的包裹函数
![](attachments/Pasted%20image%2020250525131534.png)

(2) `pthread`线程相关的包裹函数
![](attachments/Pasted%20image%2020250525131921.png)

# errno的打印
若想要打印 `errno`，需要包含头文件 `#include <errno.h>`。

## 使用 perror 打印错误信息
**函数原型：**:
`void perror(const char *s)`

**头 文 件：**:
`#include <stdio.h>`

**作 用：** 
把一个描述性错误消息输出到标准错误 `stderr`，其中 `str` 是自定义内容。该函数先输出 `str`，再输出`errno`代表的错误描述符

```c
if(-1 == connect(fd, addr, len))
    perror("connect");
//连接错误则输出以下结果：
connect: Connection refused
```

## 使用 strerror 显示错误信息
**函数原型：**：
`char *strerror(int errnum);`

**头 文 件：**
`#include <string.h>`

**作 用：**
将错误码以字符串的信息显示出来

```c
if(-1 == connect(fd, addr, len))
{
    char* str = strerror(errno);
    fputs(str, stderr);
}

```

# 其他
## 用户级库返回的errno值和内核级返回的errno值
**现象**：
```bash
ibv_post_send() returns ENOMEM 
(and not -ENOMEM which is what the drivers seem to return when kmalloc fails or something similar)
```

**解释**：
```bash
User level libraries return positive errno values and not negative ones  
(kernel level drivers return negative errno values)
```
用户级库返回正的 `errno` 值，而不是负值（内核级驱动返回负的 `errno` 值）

# 参考
```bash
# 错误处理与包裹函数
http://62.234.193.50:8080/archives/error-dispose-wrap
```