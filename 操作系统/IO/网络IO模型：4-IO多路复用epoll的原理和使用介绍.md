```table-of-contents
```
# 概述
epoll 机制的核心是 `epoll_create` 函数，它创建了一个 epoll 对象，用于存储文件描述符和事件信息。
epoll 对象可以通过 epoll_ctl 函数来添加、修改和删除文件描述符和事件信息。
当文件描述符上有事件发生时，epoll_wait 函数会返回事件信息，应用程序可以根据事件信息进行相应的处理。

# epoll的事件模式

epoll有三种工作模式：LT（水平触发： Level-Triggered, LT）模式和ET（边缘触发： Edge-Triggered, ET）模式 和 ONESHOT。

默认情况下，epoll采用 LT模式工作，这时可以处理阻塞和非阻塞套接字，而 EPOLLET表示可以将一个事件改为 ET模式。
ET模式的效率要比 LT模式高，它只支持非阻塞套接字。

## epoll的水平触发：level trigger（LT）
- 对于读操作，只要文件描述符的读缓冲区不为空，触发可读事件。
- 对于写操作，只要文件描述的写缓冲区不满，触发可写事件。
> 因此一旦我们没有要写的数据，就要从epoll中取消关注写事件，防止无效触发。

![](attachments/Pasted%20image%2020250530100904.png)

即：只要文件描述符的状态符合条件（比如可读或可写），`epoll_wait()` 每次调用都会返回该事件。　应用程序可以不立即处理该事件。下次调用 `epoll_wait()` 时，它还会再次通知应用程序该事件。

比如：当被监控的文件描述符上有可读写事件发生时，`epoll_wait()`会通知处理程序去读写。如果这次没有把数据一次性全部读写完(如读写缓冲区太小)，那么下次调用 `epoll_wait()`时，它还会通知你在上次没读写完的文件描述符上继续读写。

### LT 模式设置
`LT（Level Triggered）`是 epoll 的默认触发机制。

### 优缺点
**优点**：
这种触发机制的优点是应用程序可以不必处理所有事件，只处理它感兴趣的事件。

**缺点**：
如果应用程序处理事件的速度太慢，该epoll事件没有被处理完（没有返回 `EWOULDBLOCK` ），那么 `epoll_wait()` 将不断通知应用程序同一个事件，这将导致 CPU 占用率过高。

## epoll的边缘触发：edge trigger（ET）

![](attachments/Pasted%20image%2020250530100914.png)

当文件描述符的缓冲区**状态发生变化**时触发。

**（1）普通socket的读操作**：
- 当读缓冲区数据为空变为非空时，触发可读事件。
- 当读缓冲区之前存在未读完数据，此时接收到新数据时：==依然触发可读事件==。
- 当读缓冲区有数据可读，且进程对相应的文件描述符进行 `EPOLL_CTL_MOD` 修改 `EPOLLIN` 事件时，触发可读事件。
```bash
在 边缘触发（ET）模式下：
1. 缓冲区有未读数据时，新数据到达不会触发可读事件。
2. 只有缓冲区从空变为非空时（例如数据首次到达，或数据被完全读空后再次到达新数据）才会触发一次事件。
```

**（2）listen-socket的读操作**：
- 无论之前的全连接队列中的连接是否通过`accept`调用消耗完，只要添加了新的连接「即新连接完成了三次握手」，那么就会产生可读事件。


**（3）普通socket的写操作**：
- 当写缓冲区由不可写变为可写时，触发可写事件。
- 当写缓冲区有空间可写，且进程对相应的文件描述符进行 `EPOLL_CTL_MOD` 修改 `EPOLLOUT` 事件时，触发可写事件。


即：当 `epoll_wait()` 检测到文件描述符上有事件发生并将此事件通知应用程序后，应用程序`必须立即处理该事件`。如果不处理，下次调用 `epoll_wait()` 时将`不会再次通知应用程序该事件`。因为该模式，只在文件描述符的状态发生变化时通知一次，例如从不可读变为可读。　

比如：当被监控的文件描述符上有可读写事件发生时，epoll_wait()会通知处理程序去读写。如果这次没有把数据全部读写完(如读写缓冲区太小)，那么下次调用epoll_wait()时，它不会通知你，也就是它只会通知你一次，直到该文件描述符上出现第二次可读写事件才会通知你。

### ET模式下，缓冲区有未读数据时，新数据到达会不会触发可读事件？
ET模式下，缓冲区有未读数据时，新数据到达会不会触发可读事件？
接收缓冲区还有数据，再次有新的数据到达，认为是状态发生了改变，依然会触发事件。

#### 结论
注：以下的结论，都是通过测试程序测试过的：

（1）ET模式下，监听了`EPOLLIN` 事件，普通`sockfd`的接收缓冲区如果还存在数据没有读取完成，新的数据到达，依然会产生可读事件。

（2）ET模式下，监听了`EPOLLIN` 事件，`listen-sockfd`的全连接队列中的连接没有通过`accept`清理完成「即全连接队列中还存在连接」，此时存在新的连接到达，还是会产生新的读事件。

（3）ET模式下，如果通过`epoll_ctl`添加了`EPOLLIN` + `EPOLLRDHUP`事件，那么收到 `FIN`, 即使之前接收缓冲区存在未读取完成的数据，此时也会产生 `EPOLLRDHUP` + `EPOLLIN`事件，通过 `EPOLLRDHUP` 事件，可感知到对端连接关闭了发送。
即：只要收到 `FIN`, 无论此时是否将接收缓冲区的数据读取完，都可以立刻通过产生`EPOLLRDHUP`事件来告知本端，对端连接已经关闭了写。

（4）ET模式下，如果通过`epoll_ctl`只是添加了`EPOLLIN` 事件，那么收到`FIN`，即使之前接收缓冲区存在未读取完成的数据，此时也是会产生`EPOLLIN`事件；
因为`fin`也是在接收缓冲区中排队「此时也可以认为接收缓冲区发生了变化； 即：接收缓冲区还存在数据时，收到`FIN`也会产生可读事件」
此时调用 `read` 需要先将接收缓冲区中的数据读取完，才会读取到`FIN「即 read 返回 0」`，是通过 `read` 返回 `0` 来感知到 FIN「即对端连接关闭了写」。

（5）ET模式下，`EPOLLIN`事件产生的原因是：  
（5.1）监听了`EPOLLIN` 事件时，无论之前接收缓冲区中是否存在未读取完的数据，只要有新数据到达，都会产生`EPOLLIN` 事件。
（5.2）监听了`EPOLLIN` 事件时，无论之前接收缓冲区中是否存在未读取完的数据，对方关闭了连接或只关闭了`SEND_SHUTDOWN`「即本端收到了`FIN`」，导致我们关闭了`RCV_SHUTDOWN`，都会产生`EPOLLIN` 事件。

（6）ET模式下，`EPOLLOUT`事件产生的原因是：  
（6.1）监听了`EPOLLOUT` 事件时，无论是`client`还是`server`端，三次握手成功后，建立了TCP连接，此时就会产生`EPOLLOUT` 事件。
（6.2）监听了`EPOLLOUT` 事件时，一直调用`write`，直到返回`EAGAIN`「此时说明`send buffer`是被占满的」，然后当`send buffer`里的数据被内核发送并释放「收到对端的`ACK`才可以释放一部分空间，否则会有超时重传」到一定程度时，就会产生`EPOLLOUT` 事件「从缓冲区满到一定水位的剩余空间（表示未满），即：存在状态变化」。
> 注：==并不是说，只要发送一部分数据，并收到了对方的ACK，导致发送缓冲区的剩余空间发生变化，就会产生`EPOLLOUT` 事件；而是说：必须是一直调用`write`，直到返回`EAGAIN`「即此时发送缓冲区满了」，然后内核将发送缓冲区的数据发送出去，收到对方的ACK，释放一部分空间，内核重复该发送过程，直到剩余空间达到一定量之后，才会产生`EPOLLOUT` 事件==。


#### `EPOLLIN` 事件的测试程序1
(1) server端：
```c
# cat server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <errno.h>
#include <time.h>
#include <sys/time.h>

#define MAX_EVENTS 10
#define PORT 8080

void set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int listen_fd, epoll_fd;
    struct epoll_event ev, events[MAX_EVENTS];
    struct sockaddr_in addr;

    // 创建监听socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    set_nonblock(listen_fd);
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 5);
    // 创建epoll实例
    epoll_fd = epoll_create1(0);
    ev.events = EPOLLIN | EPOLLET; // 边缘触发模式
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    printf("Server running...\n");

    while (1) {
        int i;
        int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (i = 0; i < n; i++) {
            if (events[i].data.fd == listen_fd) {
                // 接受新连接
                int conn_fd = accept(listen_fd, NULL, NULL);
                struct timeval now;
                gettimeofday(&now, NULL);
                char timebuf[50];
                strftime(timebuf, sizeof(timebuf), "%Y-%m-%d %T", localtime(&now.tv_sec));

                set_nonblock(conn_fd);
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev);
                printf("%s.%06lu listen fd:%d get event:%d, New connection: fd=%d\n", timebuf, now.tv_usec %1000000,  events[i].data.fd, events[i].events, conn_fd);
            } else {
                // 处理数据
                int conn_fd = events[i].data.fd;
                char buf[4];
                ssize_t count;
                int total_read = 0;
                struct timeval now;
                gettimeofday(&now, NULL);
                char timebuf[50];
                strftime(timebuf, sizeof(timebuf), "%Y-%m-%d %T", localtime(&now.tv_sec));
                printf("%s.%06lu fd:%d get event:%d.\n", timebuf, now.tv_usec %1000000, events[i].data.fd, events[i].events);
                // 循环读取直到缓冲区空
                // while (1) {
                    count = read(conn_fd, buf, sizeof(buf));
                    if (count == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            printf("Data exhausted on fd=%d\n", conn_fd);
                            //break;
                        } else {
                            perror("read");
                            exit(1);
                        }
                    } else if (count == 0) {
                        printf("Connection closed by client: fd=%d\n", conn_fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, conn_fd, NULL);
                        close(conn_fd);
                        //break;
                    } else {
                        total_read += count;
                        printf("Read %zd bytes(%s) (total %d) from fd=%d\n", count, buf, total_read, conn_fd);
                    }
               //}
            }
        }
    }
    return 0;
}
```

(2) client端：
```c
# cat client.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in serv_addr;
    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(PORT);
    inet_pton(AF_INET, "10.96.83.26", &serv_addr.sin_addr);

    connect(sock, (struct sockaddr*)&serv_addr, sizeof(serv_addr));

    // 第一次发送数据
    char buf[20];
    strcpy(buf, "123456789");
    write(sock, buf, strlen(buf));
    printf("Sent first data (%d bytes:%s)\n", strlen(buf), buf);
    sleep(2); // 等待服务端处理

    // 第二次发送数据（缓冲区仍有未读数据）
    strcpy(buf, "aabbccdd");
    write(sock, buf, strlen(buf));
    printf("Sent second data (%d bytes:%s)\n", strlen(buf), buf);
    sleep(2);

    close(sock);
    return 0;
}
```

（3）测试结果：
3.1> server端查看：
```bash
# ./server1
Server running...
2025-05-28 20:39:54.829421 listen fd:3 get event:1, New connection: fd=5
2025-05-28 20:39:54.829494 fd:5 get event:1.
Read 4 bytes(1234) (total 4) from fd=5
2025-05-28 20:39:56.829531 fd:5 get event:1.
Read 4 bytes(5678) (total 4) from fd=5
2025-05-28 20:39:58.829597 fd:5 get event:1.
Read 4 bytes(9aab) (total 4) from fd=5
^C
```

如上所示，==ET模式下，普通socket的接收缓冲区中依然存在数据没有读取完成时，新的数据到达，依然会产生可读事件==。

3.2> client查看：
```bash
# ./client1
Sent first data (9 bytes:123456789)
Sent second data (8 bytes:aabbccdd)
```
![](attachments/Pasted%20image%2020250528204221.png)
如上所示，client最后收到的`rst`应该是 server 通过`ctrl + c`退出时，发送了`rst`。至于 `client`发送`fin`给`server`端, server为什么没有走到`count == 0 下， close`的流程？？
这个是因为 server端的程序的`read`都是靠着事件触发的，client调用了2次`write`，以及一次`close`，这样`server`端会收到三个可读；但是`server`端的接收缓冲区每次调用`read`的时候，都是存在数据的，那么三次`read`的返回值都是大于0的。
这个也说明：==ET模式下，如果普通的`sockfd`在没有监听`EPOLLRDHUP`事件的情况下，对端发送`fin`时，`fin`也是在接收缓冲区中排队「此时也可以认为接收缓冲区发生了变化， 即接收缓冲区还存在数据时，收到`FIN`也会产生可读事件」，只有将接收缓冲区读完了之后，才可以读取到`fin`「即 read 返回 0」==。

#### `EPOLLIN` 事件和`EPOLLRDHUP`事件的测试程序2
（1）测试程序：
其他程序，如测试程序1，只不过将`server.c`中`accept`的 `new_fd`，添加 `EPOLLRDHUP`事件，如下所示。


（2）测试结果：
2.1> server端：
```bash
# ./server3
Server running...
2025-05-28 20:54:30.014625 listen fd:3 get event:1, New connection: fd=5
2025-05-28 20:54:30.014697 fd:5 get event:1.
Read 4 bytes(1234) (total 4) from fd=5
2025-05-28 20:54:32.014728 fd:5 get event:1.
Read 4 bytes(5678) (total 4) from fd=5
2025-05-28 20:54:34.014805 fd:5 get event:8193.
Read 4 bytes(9aab) (total 4) from fd=5
^C
```
如上所示，8193 = 0x2001 = `EPOLLRDHUP` + `EPOLLIN`, 这个说明，==如果添加了`EPOLLRDHUP`事件，那么收到 `FIN`, 就会立刻产生 `EPOLLRDHUP` + `EPOLLIN`事件，通过 `EPOLLRDHUP` 事件，可感知到对端连接关闭了发送==。

```bash
while true; do date +%T.%N; netstat -aepn | grep 10.96.83.25; echo ----------------; sleep 0.5; done

# netstat -aepn | head
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name
```
![](attachments/Pasted%20image%2020250528214949.png)
如上所示，`20:57:30`左右状态从 `ESTABLISHED`到`CLOSE_WAIT`, 这个是因为收到对端的`FIN`，本端还没有调用 `close` 去发送`FIN`;
`20:57:37`不存在任何状态的连接，这个是因为执行了`ctrl+C`退出程序。

2.2> client端：
```bash
# ./client1
Sent first data (9 bytes:123456789)
Sent second data (8 bytes:aabbccdd)
```

![](attachments/Pasted%20image%2020250528215111.png)
如上所示，`20:57:37`左右收到了`rst`是因为对端执行了`ctrl+C`退出程序。

#### `EPOLLIN` 事件的测试程序3

（1）server端程序：
```c
# cat server.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <errno.h>
#include <time.h>
#include <sys/time.h>

#define MAX_EVENTS 10
#define PORT 8080

void set_nonblock(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int listen_fd, epoll_fd;
    struct epoll_event ev, events[MAX_EVENTS];
    struct sockaddr_in addr;

    // 创建监听socket
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    addr.sin_family = AF_INET;
    addr.sin_port = htons(PORT);
    addr.sin_addr.s_addr = INADDR_ANY;

    set_nonblock(listen_fd);
    bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 5);
    // 创建epoll实例
    epoll_fd = epoll_create1(0);
    ev.events = EPOLLIN | EPOLLET; // 边缘触发模式
    ev.data.fd = listen_fd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev);

    printf("Server running...\n");

    while (1) {
        int i;
        int n = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        for (i = 0; i < n; i++) {
            if (events[i].data.fd == listen_fd) {
                // 接受新连接
                int conn_fd = accept(listen_fd, NULL, NULL);
                struct timeval now;
                gettimeofday(&now, NULL);
                char timebuf[50];
                strftime(timebuf, sizeof(timebuf), "%Y-%m-%d %T", localtime(&now.tv_sec));

                set_nonblock(conn_fd);
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn_fd;
                epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev);
                printf("%s.%06lu listen fd:%d get event:%d, New connection: fd=%d\n", timebuf, now.tv_usec %1000000,  events[i].data.fd, events[i].events, conn_fd);
            } else {
                // 处理数据
                int conn_fd = events[i].data.fd;
                char buf[4];
                ssize_t count;
                int total_read = 0;
                struct timeval now;
                gettimeofday(&now, NULL);
                char timebuf[50];
                strftime(timebuf, sizeof(timebuf), "%Y-%m-%d %T", localtime(&now.tv_sec));
                printf("%s.%06lu fd:%d get event:%d.\n", timebuf, now.tv_usec %1000000, events[i].data.fd, events[i].events);
                // 循环读取直到缓冲区空
                // while (1) {
                    count = read(conn_fd, buf, sizeof(buf));
                    if (count == -1) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK) {
                            printf("Data exhausted on fd=%d\n", conn_fd);
                            //break;
                        } else {
                            perror("read");
                            exit(1);
                        }
                    } else if (count == 0) {
                        printf("Connection closed by client: fd=%d\n", conn_fd);
                        epoll_ctl(epoll_fd, EPOLL_CTL_DEL, conn_fd, NULL);
                        close(conn_fd);
                        //break;
                    } else {
                        total_read += count;
                        printf("Read %zd bytes(%s) (total %d) from fd=%d\n", count, buf, total_read, conn_fd);
                    }
               //}
            }
        }
    }
    return 0;
}
```


(2) client程序：
```c
# cat client11.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

#define PORT 8080
#define CLIENTS 5  // 发起5个连接

int main() {
    int socks[CLIENTS];
    int i;
    struct sockaddr_in serv_addr = {
        .sin_family=AF_INET,
        .sin_port=htons(PORT),
        .sin_addr.s_addr=inet_addr("10.96.83.26")
    };

    // 批量发起连接
    for (i = 0; i < CLIENTS; i++) {
        socks[i] = socket(AF_INET, SOCK_STREAM, 0);
        connect(socks[i], (struct sockaddr*)&serv_addr, sizeof(serv_addr));
        printf("[Client] 发起第 %d 个连接\n", i+1);
        sleep(1);  // 延迟确保服务端未处理完毕
    }

    // 保持连接不关闭
    pause();
    return 0;
}
```

（3）测试结果：
3.1> server端查看：
```c
# ./server1
Server running...
2025-05-28 20:36:21.217516 listen fd:3 get event:1, New connection: fd=5
2025-05-28 20:36:22.217658 listen fd:3 get event:1, New connection: fd=6
2025-05-28 20:36:23.217800 listen fd:3 get event:1, New connection: fd=7
2025-05-28 20:36:24.217958 listen fd:3 get event:1, New connection: fd=8
2025-05-28 20:36:25.218101 listen fd:3 get event:1, New connection: fd=9
2025-05-28 20:36:41.039023 fd:9 get event:1.
Connection closed by client: fd=9
2025-05-28 20:36:41.039055 fd:8 get event:1.
Connection closed by client: fd=8
2025-05-28 20:36:41.039068 fd:7 get event:1.
Connection closed by client: fd=7
2025-05-28 20:36:41.039081 fd:6 get event:1.
Connection closed by client: fd=6
2025-05-28 20:36:41.039092 fd:5 get event:1.
Connection closed by client: fd=5
^C
```
如上所示，==ET模式下，listen-sockfd的全连接队列中的连接没有通过accept清理完成，存在新的连接到达，还是会产生新的读事件==。


```bash
while true; do date +%T.%N; netstat -aepn | grep 10.96.83.25; echo ----------------; sleep 0.5; done

# netstat -aepn | head
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       User       Inode      PID/Program name
```
![](attachments/Pasted%20image%2020250528203812.png)


3.2> client查看：
```bash
# ./client11
[Client] 发起第 1 个连接
[Client] 发起第 2 个连接
[Client] 发起第 3 个连接
[Client] 发起第 4 个连接
[Client] 发起第 5 个连接
^C  // 表示ctrl+C退出。
```
![](attachments/Pasted%20image%2020250528203858.png)


#### `EPOLLOUT` 事件的测试程序4
（1）`server`端的代码：
```c
#include <arpa/inet.h>
#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define PORT 9999
#define MAX_EVENTS 10

static int tcp_listen() {
  int lfd, opt, err;
  struct sockaddr_in addr;

  lfd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  assert(lfd != -1);

  opt = 1;
  err = setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
  assert(!err);

  bzero(&addr, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = INADDR_ANY;
  addr.sin_port = htons(PORT);

  err = bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
  assert(!err);

  err = listen(lfd, 8);
  assert(!err);

  return lfd;
}

static void epoll_ctl_add(int epfd, int fd, int evts) {
  struct epoll_event ev;
  ev.events = evts;
  ev.data.fd = fd;
  int err = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
  assert(!err);
}


static void handle_events(struct epoll_event *e, int epfd) {
  int n = write(e->data.fd, "hi\n", 3);
  assert(n == 3);

  printf("events %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
  }

  assert(e->events == 0);
  printf("\n");
}


int main(int argc, char *argv[]) {
  int epfd, lfd, cfd, err, n;
  struct epoll_event events[MAX_EVENTS];

  epfd = epoll_create1(0);
  assert(epfd != -1);

  lfd = tcp_listen();
  epoll_ctl_add(epfd, lfd, EPOLLIN);

  for (;;) {
    n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    assert(n != -1);

    for (int i = 0; i < n; i++) {
      if (events[i].data.fd != lfd) {
        handle_events(&events[i], epfd);
        continue;
      }

      cfd = accept(lfd, NULL, NULL);
      assert(cfd != -1);

      err = fcntl(cfd, F_SETFL, O_NONBLOCK);
      assert(!err);

      epoll_ctl_add(epfd, cfd, EPOLLIN | EPOLLOUT | EPOLLET);
    }
  }

  return 0;
}
```

(2) 测试：
在最开始的地方加了个`write`方法。  那是不是说，在此种模式下，程序会陷入死循环呢？`tcp`一连接上，就会有`EPOLLOUT`事件发生，然后我们就往`socket`中写了些数据，内核将该数据发送完毕之后又会触发`EPOLLOUT`，然后又发数据，这样就进入了死循环。是这样吗？我们来执行看下。

(2.1) server端：

```bash
$ gcc server.c && ./a.out
events 5: EPOLLOUT
```

(2.2) client 端：

```bash
$ ncat localhost 9999
hi
```

执行流程为：
```bash
1. 开启服务端，等待客户端连接，此时服务端终端没有任何输出。
2. 用ncat模拟客户端连服务端，在连接上之后，服务端会输出`EPOLLOUT`，客户端会输出`hi`，说明服务端的数据确实发到了客户端。
3. 没有任何其他的反应了。
```

可以看到，这个输出和我们预想的并不一样，服务端的`tcp`在发送完数据后，并没有通知给我们`EPOLLOUT`事件，所以没有我们上文猜测的死循环出现。
是为什么呢？如果此时不通知，什么时候才会通知呢？
如果细心读过`epoll`文档的朋友可能会注意到下面这段话：
```bash
The suggested way to use epoll as an edge-triggered (EPOLLET) interface is as follows:
(1) with nonblocking file descriptors; and
(2) by waiting for an event only after read(2) or write(2) return EAGAIN.
```

#### `EPOLLOUT` 事件的测试程序5

和上例中的`EPOLLOUT事件的测试程序4`类似，只是将`handle_events`方法改成如下，再执行服务端程序，客户端还是用`ncat`模拟：
```c
static void handle_events(struct epoll_event *e, int epfd) {
  int n;
  char buf[8192];
  while (1) {
    n = write(e->data.fd, buf, 8192);
    if (n == -1) {
      assert(errno == EAGAIN);
      break;
    }
  }

  printf("events %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
  }

  assert(e->events == 0);
  printf("\n");
}
```

测试结果：
当`tcp`建立连接之后，我们会发现服务端终端一直输出`EPOLLOUT`，进入了死循环，和我们分析的结果是一样的。

### ET模式下的读和写以及accept
==在`epoll`的`ET`模式下，正确的读写方式为:
读：只要可读，就一直读，直到返回`0`，或者 `errno = EAGAIN`；
写：只要可写，就一直写，直到数据发送完，或者 `errno = EAGAIN`==。


#### 读
```c
ssize_t read_n(int fd, void *vptr, size_t n)
{
    size_t nleft;
    ssize_t nread;
    char *ptr;

    ptr = vptr;
    nleft = n;
    while (nleft > 0) {
        if ((nread = read(fd, ptr, nleft)) < 0) {
            if ((errno == EINTR) || (errno == EAGAIN))
                nread = 0;      /* and call read() again */
            else
                return (-1);
        } else if (nread == 0)
            break;      /* EOF */

        nleft -= nread;
        ptr += nread;
    }

    return (n - nleft);     /* return >= 0 */
}

static inline int sockopt_msg_recv(int clt_fd, struct dpvs_sock_msg_reply *reply_hdr, 
        void **out, size_t *out_len)
{
    void *msg = NULL;
    size_t len, res;

    if (!reply_hdr) {
        fprintf(stderr, "[%s] empty reply msg pointer\n", __func__);
        return -ESOCKOPT_INVAL;
    }

    if (out)
        *out = NULL; // struct ip_vs_getinfo *ipvs_info_rcv ; out = &ipvs_info_rcv; *out = ipvs_info_rcv = NULL;
    if (out_len)
        *out_len = 0;

    len = sizeof(struct dpvs_sock_msg_reply);
    memset(reply_hdr, 0, len);
    res = read_n(clt_fd, reply_hdr, len);
    if (len != res) {
        fprintf(stderr, "[%s] socket msg header recv error -- %zu/%zu recieved\n",
                __func__, res, len);
        return -ESOCKOPT_IO;
    }

    if (reply_hdr->errcode) {
        fprintf(stderr, "[%s] errcode set in socket msg#%d header: %s(%d)\n", __func__,
                reply_hdr->id, reply_hdr->errstr, reply_hdr->errcode);
        return reply_hdr->errcode;
    }

    if (reply_hdr->len > 0) {
        msg = malloc(reply_hdr->len);
        if (NULL == msg) {
            fprintf(stderr, "[%s] no memory\n", __func__);
            return -ESOCKOPT_NOMEM;
        }

        res = read_n(clt_fd, msg, reply_hdr->len);
        if (res != reply_hdr->len) {
            fprintf(stderr, "[%s] socket msg body recv error -- %zu/%zu recieved\n",
                    __func__, res, reply_hdr->len);
            free(msg);
            return -ESOCKOPT_IO;
        }
    }

    if (SOCKOPT_VERSION != reply_hdr->version) {
        fprintf(stderr, "[%s] socket msg version not match\n", __func__);
        if (reply_hdr->len > 0)
            free(msg);
        return -ESOCKOPT_VERSION;
    }

    if (out && out_len) {
        *out = msg; // ipvs_info_rcv = msg;
        *out_len = reply_hdr->len;
    } else if (reply_hdr->len > 0) {
        free(msg);
        if (out)
            *out = NULL;
        if (out_len)
            *out_len = 0;
    }

    return ESOCKOPT_OK;
}
```

分析：在读取数据时候，`(errno == EINTR) || (errno == EAGAIN)` 继续读取；
`errno == EINTR` 比较好理解，有中断时，中断完成之后应该继续读。
`errno == EAGAIN` 之后还继续读，这个是因为就是想要读取指定长度的字节数，虽然此时缓冲区已经空了，但是想要的字节数没有读取完成，还应该继续读取。
> 注：因为对端发送的指定长度的数据，可能分布在几个数据包中，分先后到达，那么最后一个数据包，如果有延迟到达，就可能出现`errno == EAGAIN`。



#### 写
##### ET模式下触发可写事件
在 ET 模式下，以下情况会触发可写事件：
1. `epoll_ctl(ADD)` 之后，fd 变为可写（通常是连接刚建立时）。
2. `epoll_ctl(MOD)` 显式修改事件。
3. 内核缓冲区从**满**变为**不满**（即腾出了空间）。
> 即：发送缓冲区满了之后，内核发送消息收到了对方的ACK，就可以删除一部分内存，可以继续写了，就会触发可写事件。
> 缓冲区满：循环调用 `send()` 直到返回 `EAGAIN` 或 `EWOULDBLOCK`。这表示你已经把内核缓冲区填满了。

```bash
在 epoll ET 模式下，只有在状态发生从“不可写”到“可写”的转变时，才会触发事件。

- 三次握手成功后：发送缓冲区是空的，状态从“无”变为“可写”，epoll 返回一次可写事件。
- 发送消息时：当你调用 `send()` 或 `write()`，数据只是从你的用户态内存拷贝到了内核的 TCP 发送缓冲区。只要缓冲区没满，`send()` 就会立即返回成功。
- ACK 的角色：对方回复 ACK 意味着数据已经安全到达，此时内核会将这些数据从本地发送缓冲区中移除，从而腾出空间。
- 关键点：只要本地发送缓冲区还没满，它就一直处于“可写状态”。由于状态没有发生“从满到不满”的切换，epoll 不会再次提醒你可写。
```



##### 范例
```c
/* send "n" bytes to a descriptor */
ssize_t send_n(int fd, const void *vptr, size_t n, int flags)
{
    size_t nleft;
    ssize_t nwritten;
    const char *ptr;

    ptr = vptr;
    nleft = n;

    while (nleft > 0) {
        if ((nwritten = send(fd, ptr, nleft, flags)) <= 0) {
            if (nwritten < 0 && (errno == EINTR || errno == EAGAIN))
                nwritten = 0;       /* and call send() again */
            else
                return (-1);        /* error */
        }
        nleft -= nwritten;
        ptr += nwritten;
    }

    return (n);
}

#define MSG_NOSIGNAL    0x80000         /* do not generate SIGPIPE on EOF */

static inline int sockopt_msg_send(int clt_fd,
        const struct dpvs_sock_msg *hdr,
        const char *data, int data_len)
{
    int len, res;

    if (!hdr) {
        fprintf(stderr, "[%s] empty socket msg header\n", __func__);
        return -ESOCKOPT_INVAL;
    }

    len = sizeof(struct dpvs_sock_msg);
    res = send_n(clt_fd, hdr, len, MSG_NOSIGNAL);
    if (len != res) {
        fprintf(stderr, "[%s] socket msg header send error -- %d/%d sent\n",
                __func__, res, len);
        return -ESOCKOPT_IO;
    }

    if (data && data_len) {
        res = send_n(clt_fd, data, data_len, MSG_NOSIGNAL);
        if (data_len != res) {
            fprintf(stderr, "[%s] scoket msg body send error -- %d/%d sent\n",
                    __func__, res, data_len);
            return -ESOCKOPT_IO;
        }
    }

    return 0;
}
```
#### accept

**ET模式下accept存在的问题**：
多个连接同时到达，服务器的非阻塞的`listen sockfd`的TCP全连接队列瞬间积累多个就绪连接，由于是边缘触发模式，`epoll`只会通知一次，`accept`只处理一个连接，导致`TCP`就绪队列中剩下的连接都得不到处理。

**解决办法**:
用`while`循环进行`accept`调用，处理完`TCP`全连接队列中的所有连接后再退出循环。
如何知道是否处理完就绪队列中的所有连接呢？
`accept`返回`-1`并且`errno`设置为`EAGAIN`就表示所有连接都处理完。

```c
while ((conn_sock = accept(listenfd,(struct sockaddr *) &remote, (size_t *)&addrlen)) >= 0)
{ 
   handle_client(conn_sock); 
} 
if (conn_sock == -1) 
{ 
   if (errno != EAGAIN && errno != ECONNABORTED && errno != EPROTO && errno != EINTR) {
   		perror("accept"); 
   }
}
```


###  ET 模式优缺点
**优点**：
这种触发机制的优点是可以避免 `epoll_wait()` 不断通知同一个事件，从而降低 CPU 占用率。

**缺点**：
应用程序必须立即处理事件，否则将会丢失事件。

### ET模式和LT模式对比

一般来说，边缘触发的效率比水平触发的效率要高。
因为水平触发时如果系统中有大量你不需要读写的就绪文件描述符，而它们每次都会返回，这样会大大降低处理程序检索自己关心的就绪文件描述符的效率；
而边缘触发不会充斥大量你不关心的就绪文件描述符，可以减少 `epoll_wait` 的系统调用次数，系统调用也是有一定的开销的的，毕竟也存在上下文的切换。


###  ET 模式注意
#### 边缘触发模式的使用
**（1）ET模式一般和非阻塞 I/O 搭配使用**
- 针对Epoll的`LT`模式，`socket fd`可以设置成阻塞也可以设置成非阻塞；
> 注：实际上，`LT`模式也是建议fd使用非阻塞模式。不建议使用阻塞模式。

- 针对Epoll的`ET`模式，`socket fd`只能设置成非阻塞；

**（2）ET模式模式的读写操作**
ET模式应该使用非阻塞套接字，在每次读时读完所有数据，每次写时要么填满发送缓冲区，要么写完当前所有数据。
ET模式的写事件触发像是一种惰性触发，不需要你在没有要写的数据时取消关注写事件。

因为如果使用边缘触发模式，`I/O` 事件发生时只会通知一次，而且我们不知道到底能读写多少数据，所以在收到通知后应尽可能地读写数据，以免错失读写的机会。因此，我们会**循环**从文件描述符读写数据，那么如果文件描述符是阻塞的，没有数据可读写时，进程会阻塞在读写函数那里，程序就没办法继续往下执行。
所以，边缘触发模式一般和非阻塞 `I/O` 搭配使用，程序会一直执行 `I/O` 操作，直到系统调用（如 `read` 和 `write`）返回错误，错误类型为 `EAGAIN` 或 `EWOULDBLOCK`。

> 注：当然如果在`LT`模式下也每次循环读取阻塞`IO`，也有类似的问题；

##### 循环读取导致其他fd延迟增大的问题
采用非阻塞循环读取方式时，如果当前`socket fd`上恰好有持续大数据量写入，则这个循环读取可能持续较长时间，从而导致其他`socket fd`上的读写操作将被延迟。
针对这种情况，我们只能是控制当前`socket fd`上的读操作，并将其保存，在下一次`event loop`中不依赖`ET`的触发，直接针对保存的`fd`继续其读操作。

#### `unblock listen-fd`在ET模式下的连接数和事件个数以及`accept`成功个数的关系

设定：`unblock listen-fd`的全连接队列中存在`N`个新建连接。 
设定：在`epoll`的`ET`模式下，会产生`K`个`unblock listen-fd`的`EPOLLIN`事件。
设定：收到可读之后：
	一次`EPOLLIN`事件，只调用一次`accept`，可以成功获取`M1`个新建连接。
	一次`EPOLLIN`事件，循环调用`accept`直到`EAGAIN`，可以成功获取`M2`个新建连接。

注： 对于服务器程序来说，N 其实是未知的，只能通过 M 来感知新建有M个。

**(1) N >= K**
`N > K`:  高并发情况下，有可能多个连接同时到达，即多个连接，只产生了一个事件。
这也是如果基于事件个数，一个事件只调用一次`accept`, 很可能会丢事件，导致连接还在全连接队列中没有被读取。

`N = K`: 少量连接情况下，每次来一个连接，都会触发一次边缘可读。
比如：现在全连接队列中有 `n1(n1>0)`个连接，此时又来了一个连接，那么此时也会产生一个边缘可读事件，而不是说不产生事件了。

**(2) N>=M1**
`N > M1`：同上。
`N = M1`：同上。

**(3) N>=M2**
`N = M2`：正常情况下， 应该是这种情况。
`N > M1`：如果是连接在全连接队列中，还没有来得及`accept`取出来，此时对端发送一个`reset`过来，那么就需要将该连接从全连接队列中清出了。但是此时事件可能已经产生了，即事件存在多余，实际去`accept`，有可能获取不到。

**(4) K 和 M1或M2 关系不定**
同上。

#### 使用 I/O 多路复用时，最好搭配非阻塞 I/O 一起使用
```bash
Under Linux, select() may report a socket file descriptor as "ready for reading", while nevertheless a subsequent read blocks. This could for example happen when data has arrived but upon examination has wrong checksum and is discarded. There may be other circumstances in which a file descriptor is spuriously reported as ready. Thus it may be safer to use O_NONBLOCK on sockets that should not block.
```
即：就算数据不会被别人读走，也可能被内核丢弃。还有文档没有明说的其它情况。
简单点理解，就是**多路复用 API 返回的事件并不一定可读写的**，如果使用阻塞 I/O， 那么在调用 `read/write` 时则会发生程序阻塞，因此最好搭配非阻塞 I/O，以便应对极少数的特殊情况。


#### IO多路复用时，使用阻塞IO的误解
**误解**：
既然 select/epoll 都返回可读了，那就表示一定能读了，阻塞函数read也就能读取了也就不会阻塞了。

**解释**：
==select 返回可读，和 read 去读，这是两个独立的系统调用，两个操作之间是有窗口的，也就是说 select 返回可读，紧接着去 read，不能保证一定可读==。

惊群现象，就是一个典型场景，多个进程或者线程通过 select 或者 epoll 监听一个 `listen socket`，当有一个新连接完成三次握手之后，所有进程都会通过 `select` 或者 `epoll` 被唤醒，但是最终只有一个进程或者线程 `accept` 到这个新连接；
若是采用了阻塞 I/O，没有`accept` 到连接的进程或者线程就 `block` 住了。
硬是要将 I/O 复用和阻塞 I/O 配合起来用的，在有些场景下，用一些奇技淫巧也是可行的，不过复杂度更高，效率也低下。

**范例**
![](attachments/Pasted%20image%2020250510172726.png)

##### io多路复用返回的可读不是确定性的
==select 返回可读，和 read 去读，这是两个独立的系统调用，两个操作之间是有窗口的，也就是说 select 返回可读，紧接着去 read，不能保证一定可读==。

比如：
（1）监听套接字「listen socket」完成队列「全连接队列」非空，返回可读，结果在读事件处理之前客户端发送RST导致该队列中的连接关闭，这时候用accept去接收阻塞的监听套接字就会阻塞。

（2）另外套接字接收缓冲区有数据到达，触发可读，结果在读事件处理之前，通过校验的方式发现接收缓冲区的数据有误，于是丢弃该数据，这时候read一个阻塞套接字仍旧会阻塞。

**对于写来说**：倒是可以根据接收缓冲区的低水位标记来发送数据，只要发送的数据小于该标记的值，理论上就不会阻塞，但是有没有什么其他的坑目前不知道。  


## ONESHOT
`ONESHOT` 触发机制是一种特殊的 ET 触发机制。当 epoll_wait() 检测到文件描述符上有事件发生并将此事件通知应用程序后，应用程序 `必须调用 epoll_ctl() 函数重新注册该文件描述符` ，否则下次调用 epoll_wait() 时将不会再次通知应用程序该事件。

### 优缺点

**优点**：
可以避免 epoll_wait() 不断通知同一个事件，从而降低 CPU 占用率。

**缺点**：
应用程序必须重新注册文件描述符，否则将会丢失事件。

## 其他

### `epoll`内的水平触发LT和边缘触发ET
**无论是ET模式还是LT模式，epoll内部维护的可读/可写状态是一样的，只是触发的准则不同。**
（这里的触发可以理解为将fd相关结构加入就绪双向列表）；
因此当一个可读事件被触发，其返回的状态**可能是可读可写**的，**但是本次fd相关结构加入就绪列表这一动作不是由于可写事件而发生。**
对方断开连接，可读事件被触发是预料之内的，但是返回的状态是可读可写的。


### `epoll/select/poll`的触发模式
`select/poll` 只有水平触发模式，`epoll` 默认的触发模式是水平触发，但是可以根据应用场景设置为边缘触发模式。


### 能不能把一个句柄（socket）添加到不同的`epoll_event`中？
Q:  比如每个线程都有一个`epoll_event`，能不能把 一个待监听的`socket`加入到多个线程的`epoll_event`中？

A: 可以的，原因是因为`sock`使用一个列表来管理等待唤醒列表，当然我们可以添加多个。

 **范例**
 我们可以构造一种使用场景，`client`两个线程，一个读，而另一个写同一个`socket`，`server`侧也有两个线程，同样是一个读一个写这个`socket`，拓扑如下：
![](attachments/Pasted%20image%2020250531210437.png)


### LT模式可以使用阻塞套接字么?
理论上可以，但是不建议使用，还是建议在LT模式下，使用非阻塞`fd`。

> 理论上`epoll`的`LT`模式能够支持阻塞模式的`socket`；比如：在阻塞模式下，当有数据到达，`epoll_wait`返回`EPOLLIN`事件，此时的处理中调用`read`读取数据，请注意，第一次调用`read`，可以保证`socket`里面有数据(`EPOLLIN`事件说明有数据)，`read`不会阻塞。第二次调用，`socket`里面有没有数据是不确定的，要是贸然调用，`read`可能就阻塞了，因此不能进行第二次调用，必须等待`epoll_wait`再次返回`EPOLLIN`才能再次`read`。因此阻塞`socket`使用就必须`read/write`一次就转到`epoll_wait`，这对于网络流量较大的应用效率是相当低的，而且一不小心就会阻塞在某个`socket`上，因此多路复用+阻塞式`socket`几乎不出现在实际应用中。

#### 小结
综上，==LT模式下也建议使用非阻塞套接字==。


### `epoll`的`listen-fd`在`ET`模式下客户端连接不上的情况

#### 问题
server端的`listen-fd` 在 ET模式下 加入到 `epoll`中，客户端发起了批量的连接。发现了一个问题。客户端的部分连接在三次握手之后，无法发送数据。从server端通过`netstat -apn `查看，会发现存在阻塞「`Recv-Q`不为0」, 如下所示。
```bash
# netstat -apn
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp       55      0 10.96.83.26:8002        10.96.83.25:37360       ESTABLISHED 60802/./krpc_echo_s
tcp       55      0 10.96.83.26:8001        10.96.83.25:37274       ESTABLISHED 60802/./krpc_echo_s
tcp       55      0 10.96.83.26:8002        10.96.83.25:37366       ESTABLISHED 60802/./krpc_echo_s
tcp       55      0 10.96.83.26:8001        10.96.83.25:37276       ESTABLISHED 60802/./krpc_echo_s
```
这个是因为部分连接三次握手完成了之后，依然在全连接队列中，没有被`accept`出来。

这个是客户端同时发起了多个连接， `listen-fd` 在 ET模式下 ，只产生了一个可读事件，进而调用`accept`只处理了一个连接；
全连接队列中的其他的连接，如果后续没有新的连接产生，就不会再次产生可读事件，导致了丢可读事件「==连接依然在全连接队列中，只是没有可读事件去触发 accept 读取新的连接==」。

#### 分析
`epoll`在`ET`边缘模式下，由于每次来的消息如果你不一次性处理完的话，下一次就不通知你了。
对监听描述符，如果是`ET`边缘模式，没有用`while`一次性全部接收的话，会出现连接不上的情况。
对监听描述符，如果是`LT`水平触发的话，就不会有这个问题，连接不上的客户，下一次`epoll`还会通知的。

#### 小结

对于`监听的sockfd「listen sockfd」`，最好使用水平触发模式，边缘触发模式会导致高并发情况下，有的客户端会连接不上。
如果需要使用边缘触发，要用`while`来循环`accept()`，记得设置`监听的sockfd「listen sockfd」`为非阻塞`io`。

```c
// 边缘模式下，listen-fd的处理

while ((new_fd = accept(listen_fd, sockaddr, addrlen)) >= 0) {
	// handle new_fd
}
if (new_fd < 0) {
	if ((EAGAIN != errno) && (EWOULDBLOCK != errno)) {
		// handle err
	}
}

```


### `epoll`的`listen-fd`使用`LT`水平触发，在特殊线程模型下产生了多个EAGAIN
#### 背景
线程模型，如下所示：

![](attachments/Pasted%20image%2020250513163433.png)

存在一个控制线程，N个worker线程。
控制线程中中，存在一个全局的 `epoll`， 一直循环调用`epoll_wait`产生事件；根据每个事件对应的conn，都可以找到其所属的`worker`， 然后通过对应的`rte_ring`传递出去。

每个worker线程中，都创建 M 个 非阻塞的 `listen-fd`(此中拿一个`listen-fd`举例),  将 非阻塞的`listen-fd` 加入到控制线程中的 `epoll`中，设置事件为`EPOLLIN`, 水平方式触发。
多个客户端的请求并发访问这个`listen-fd`，控制线程中的`epoll_wait`产生`listen-fd`的事件，通过`rte_ring`无锁环形队列传递给对应的 `worker` 线程中。
`worker`线程中，自定义一个`epoll`的结构，以及自定义`epoll_wait`。 其中，`epoll_wait` 会从`rte_ring`无锁环形队中取事件。

注：控制线程和每个`worker`线程之间都存在一个`rte_ring`；

#### 问题
正常情况下，非阻塞的 `listen-fd`使用`LT`水平触发时，`epoll_wait` 上 `listen-fd`产生可读事件，然后`accept`的时候，
**一段时间内**最多也就只有一个`accept 返回-1, errno = EAGAIN`。

但是使用上面的线程模型测试的时候，发现一个`worker`线程进行`accept`的时候，**一段时间内**会产生多个`accept 返回-1, errno = EAGAIN`的情况。

#### 分析
##### （1）为什么采用水平触发
**边缘触发的问题**
如果是边缘触发，可能实际的连接数大于产生的`EPOLLIN`事件数，则需要在非阻塞的 `listen-fd`存在`EPOLLIN`事件的时候，通过`while`循环的方式来调用`accept`来获取连接；否则可能会因为`EPOLLIN`事件过少，导致无法通过事件触发去调用`accept`, 进而导致连接还在全连接队列中。

（1）如果是用户程序，存在`EPOLLIN`事件，通过`while`循环的方式来调用`accept`来获取多个连接：
可能不存在问题。

（2）如果是用户程序，存在`EPOLLIN`事件，一次事件只调用一次`accept`来获取一个连接：
正常情况下，会出现`EPOLLIN`事件过少，导致无法通过事件触发去调用`accept`。
如果自己在通讯库中间件中通过`while`循环的方式来调用`accept`来获取多个连接，进行保存，并且自主触发`EPOLLIN`事件，可能也是可以的。

##### **（2）为什么会产生多个`accept 返回-1, errno = EAGAIN`的情况**。
主要是 控制线程中的 非阻塞的 `listen-fd`使用`LT`水平触发，==只要这个`listen-fd`的全连接队列中存在连接，都会产生一个可读事件==，通过`rte_ring`传递出去。

可能某个`worker`的`listen-fd`只有10个`client`的连接，因为worker线程中`listen-fd`存在可读事件，只会调用`accept`一次，一次只能产生一个新的连接，然后处理也需要时间。在这个期间，控制线程的循环`epoll_wait`中，由于是水平触发，可能产生了50个`listen-fd`的可读事件。

然后在对应的`worker`中就会看到`50`个`listen-fd`的可读事件，但是实际只有`10`个连接，剩余的`40`个可读事件，在调用`accept`从全连接队列中取连接，实际都是取不到连接的，此时就会`accept 返回-1, errno = EAGAIN`。

注：如果是每个`worker`都使用系统的`epoll 以及 epoll_wait`, 不存在控制连接。那么应该也不存在上诉的问题。

##### **（3）方案一：采用边缘触发，在worker线程中通过循环方式进行实际accept，将多余的连接保存，是否存在问题？**

采用边缘触发，在`worker`线程中通过循环方式进行实际`accept`，将多余的连接保存；还是存在问题。

分析：因为还是会缺乏事件，去触发`accept`读取连接。保存了连接没用，还需要产生事件，才可以。
之前说==在同一个线程中==，边缘模式，不循环调用`accept`，会丢事件，进而不会去触发`accept`；实际上并不会丢连接，连接还是在全连接队列中「不考虑全连接队列满的情况」。

##### **（4）方案二：方案一的基础上 + 存在连接就自主产生事件，是否存在问题？**
**目标**
方案二的本意是：底层使用边缘触发，实际封装一层水平触发。目标是：不丢事件， 并且不触发多余的`EAGAIN`错误。

**实际结果**
但是实际测试发现，会产生多余的`EAGAIN`错误，即产生了过多的事件「事件数大于连接数」。
注：虽然产生了`EAGAIN`错误，用户程序进行处理，不会有什么问题。 但是实际期望的效果是正常情况下，一个`EAGAIN`错误也不产生。异常情况下，可能会产生。

**分析**：
主要是：上面这种线程模型，可能无法彻底解决不产生一个`EAGAIN`。因为两个线程的频率无法同步。只要最终产生的`listen-sock`的`EPOLLIN`事件「自主产生的事件 + `epoll_wait` 系统调用产生」，比全连接队列中的连接数多，那么就会有`EAGAIN`发生。

比如：一个线程A，一直是通过`epoll_wait`系统调用，边缘触发产生`listen-sock`的`EPOLLIN`事件。
另外一个线程B，对于每一个`EPOLLIN`事件，在`while循环`中不停的`accept`，将多余的连接保存，并自主产生`EPOLLIN`事件。

举例：如线程A一下子接收了10个连接，产生了10个`EPOLLIN`事件「即在首个连接被`accept`提取之前，其他的9个连接到达了，一共产生 了10个事件」；然后第一个`EPOLLIN`事件，触发了线程B中进行`while`循环`accept`, 可以获取到`10`个连接；因为线程B发现一次可以accept多次，那么可能也会产生事件。这个时候就出现了问题，线程B自主产生的`EPOLLIN`事件 + 线程A产生的`EPOLLIN` 大于实际的连接数「10个」，那么最终就会产生了`EAGAIN`错误。


**解决**
上面这种线程模型，可能无法彻底解决不产生一个`EAGAIN`。因为两个线程的频率无法同步。只要最终产生的`listen-sock`的`EPOLLIN`事件，比全连接队列中的连接数多，那么就会有`EAGAIN`发生。
因此，==需要将系统调用`epoll_wait`和`accept`放在一个线程中，才可以解决这个问题==。

即：使用水平触发：
（1）线程A中调用`epoll_wait`+ `accept`，将新连接封装通过`rte_ring`传递给线程B，同时上报`EPOLLIN`事件。
（2）线程B中不自主产生事件，只需要在执行自定义的`accpet`的时候，取连接即可。

#### 小结
（1）当`epoll_wait`和`accept`==在一个线程中时==，非阻塞的 `listen-fd`使用`LT`水平触发：
就不会丢事件；正常情况下，也不会产生多余的`EAGAIN`错误「异常情况下，可能会产生」。
因为：在一个线程中时，水平触发，`epoll_wait`感知到一个链接，立马就产生了一个`EPOLLIN`事件, 然后`accept`去读取了。

（2）当`epoll_wait`和`accept`==不在一个线程中时==，非阻塞的 `listen-fd`使用`LT`水平触发：
就可能会产生多余的事件，进而出现`EAGAIN`的问题。
因为两个线程的频率无法同步，即线程A通过`epoll_wait`产生事件的频率，和线程B被事件触发通过`accept`来获取连接的频率，这2个不同步。

（3）当`epoll_wait`和`accept`==不在一个线程中时==，非阻塞的 `listen-fd`使用`ET`边缘触发：
边缘触发，可能实际的连接数大于事件数，导致部分连接无法被读取。因此，事件触发后，需要`while`循环`accept`，并且自主产生事件，来防止连接无法读取的情况。但是，此时可能会产生过多的事件，出现事件数大于连接数，出现`EAGAIN`的情况。


因此，==`epoll_wait`和`accept` ，这2个都需要在一个线程中执行==。

### `epoll`在 LT水平模式触发时不断的产生事件，如何处理？
#### 问题
使用Linux epoll模型，水平触发模式；当socket可写时，会不停的触发socket可写的事件，如何处理？

因为：LT模式下不需要读写的文件描述符仍会不停地返回就绪，这样就会影响我们监测需要关心的文件描述符的效率。
#### 解决

平时不要把该描述符放进`epoll`结构体中，当需要写该`fd`的时候，调用`epoll_ctl`把`fd`加入`epoll`里监听，可写的时候就往里写，写完再次调用`epoll_ctl`把fd移出`epoll`。
> 注：**这种方式的缺点是，即使发送很少的数据，也要把`socket`加入`epoll`，写完后在移出`epoll`，有一定操作代价。**

改进一下就是：平时不要把该描述符放进`epoll`结构体中，需要写的时候直接调用`write`或者`send`写数据；如果返回值是`EAGAIN`（说明写缓冲区满了），在`epoll`的驱动下，存在写事件时，说明缓冲区有空闲， 此时通过`send/wirte`进行写数据，写完了之后，再将`fd`从`epoll`中移除。下次继续重复上面的操作。
> 注：**这种方式的优点是：数据不多的时候可以避免epoll的事件处理，提高效率。**


## 总结
### ET模式总结
**ET模式（边缘触发）**
- 边沿触发模式很大程度上降低了同一个`epoll`事件被重复触发的次数，所以效率更高；
- 对于读写的`sockfd`「非`listen的 sockfd`， 比如 `accept 产生的sockfd` 或者 `client的 connect fd`」，边缘触发模式下，必须使用非阻塞IO，并要一次性全部读写完数据。
- ET的编程可以做到更加简洁，某些场景下更加高效，但另一方面**容易遗漏事件，容易产生bug**；

### LT模式总结
**LT 模式（水平触发，默认）**
- 只要有数据都会触发，缓冲区剩余未读尽的数据会导致epoll_wait返回；
- LT比ET多了一个开关EPOLLOUT事件(系统调用消耗，上下文切换）的步骤；
- 对于监听的 sockfd「listen sockfd」，最好使用水平触发模式（参考nginx），边缘触发模式会导致高并发情况下，有的客户端会连接不上，**LT适合处理紧急事件**；
- 对于读写的 sockfd，水平触发模式下，阻塞和非阻塞效果都一样，不过为了防止特殊情况，还是建议设置非阻塞；
-  poll/select只有LT模式：LT的编程与poll/select接近，符合一直以来的习惯，不易出错；

### IO多路复用搭配非阻塞IO
IO多路复用搭配非阻塞IO，主要是三方面原因：
（1）多线程环境下，一个`listenfd`会被多个IO多路复用器对象管理，当一个连接到来时所有的IO多路复用器都收到通知，所有的IO多路复用器都会去响应，但最终只有一个accept成功；如果使用阻塞，那么其他的IO多路复用器将一直被阻塞着。所以最好使用非阻塞IO及时返回。
（2）ET边缘触发下，读事件触发之后才会读操作，那么需要在一次事件循环中把缓冲区读空，否则没有读空，后续不会再产生读事件；
如果使用阻塞模式，那么当读缓冲区的数据被读完后，就会一直阻塞住无法返回。
（3）select bug。当某个socket接收缓冲区有新数据分节到达，然后select报告这个socket描述符可读，但随后，协议栈检查到这个新分节检验和错误，然后丢弃了这个报文，这时调用recv/read则无数据可读；如果socket没有设置成`blocking`，此recv/read将阻塞当前线程。


# epoll的原理
## 原理
### 整体概述
```c
int epoll_create(int size); 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);
```
![](attachments/Pasted%20image%2020250511222753.png)

一颗红黑树，一张准备就绪句柄链表，少量的内核cache，就帮我们解决了大并发下的socket处理问题.

- 执行epoll_create() 时，创建了红黑树和就绪链表;
- 执行 epoll_ctl() 时，如果增加 socket 句柄，则检查在红黑树中是否存在，存在立即返回，不存在则添加到树干上，然后向内核注册回调函数，用于当中断事件来临时向准备就绪链表中插入数据;
- 执行 epoll_wait() 时立刻返回准备就绪链表里的数据即可。

### “事件”的就绪通知方式
epoll使用“事件”的就绪通知方式，通过`epoll_ctl`注册fd，一旦该fd就绪，内核就会采用类似`callback`的回调机制来激活该fd（即加入到就绪链表中），`epoll_wait`便可以收到通知。

## 使用范例
### server端
```c

```
### client端
```c

```
## 数据结构
```c
   #include <sys/epoll.h>

   struct epoll_event {
       uint32_t      events;  /* Epoll events */
       epoll_data_t  data;    /* User data variable */
   };

   union epoll_data {
       void     *ptr;
       int       fd;
       uint32_t  u32;
       uint64_t  u64;
   };

   typedef union epoll_data  epoll_data_t;
```

说明：
```bash
 The data member of the epoll_event structure specifies data that
       the kernel should save and then return (via epoll_wait(2)) when
       this file descriptor becomes ready.
```
## 函数

### epoll_create
```c
int epoll_create(int size);
```
#### 作用
内核会产生一个epoll 实例数据结构，并返回一个文件描述符epfd。**当不需要时需要调用close函数关闭该文件描述符**。
这个特殊的epfd描述符就是epoll实例的句柄，后面的两个接口都以它为中心。同时也会创建红黑树和就绪列表，红黑树来管理注册fd，就绪列表来收集所有就绪fd。
size参数表示所要监视文件描述符的最大值，不过在后来的Linux版本中已经被弃用。它并不是一个硬性限制，只是一个提示。如果 size 小于等于 0，epoll_create 会返回一个错误「会报`invalid argument`错误」。

### epoll_create1
```c
int epoll_create1(int flags);
```
### epoll_ctl
```c
// eventpoll.h
#define EPOLL_CTL_ADD 1
#define EPOLL_CTL_DEL 2
#define EPOLL_CTL_MOD 3

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);  
```
#### 作用
将被监听的socket文件描述符添加到红黑树或从红黑树中删除或者对监听事件进行修改；同时向内核中断处理程序注册一个回调函数，内核在检测到某文件描述符可读/可写时会调用回调函数，该回调函数将文件描述符放在就绪链表中。


#### 删除fd
在`kernel 2.6.9`之前，调用`epoll_ctl`解除文件描述符的监视时，第三个参数不可被设置为`nullptr`，尽管这种情况下这个参数没有意义。

#### client端 非阻塞的fd 的 epoll_ctl  和 connect的先后顺序问题
因为`epoll`本身没有明确提出当非阻塞的fd进行`connect`成功之后会返回什么样的信号；

通过测试有如下结果：
（1）当本地还没调用`connect`函数，却将套接字送交`epoll`检测，`epoll`会产生一次 `EPOLLOUT | EPOLLHUP`， 也就是产生一个值为`0x14`的`events`.
（2）当本地`connect`事件发生了，但建立连接失败，则`epoll`会产生一次 `EPOLLIN | EPOLLERR | EPOLLHUP`， 也就是一个值为`0x19`的`events`.
（3）当`connect`函数也调用了，而且连接也顺利建立了，则`epoll`会产生一次 `EPOLLOUT`， 值为`0x4`，即表明套接字已经可写。

因而，对于`client`要判断连接建立，只需要判断该套接字有可写事件且仅有可写事件即可。

##### 分析


**（1）`connect` 的非阻塞行为：** 
非阻塞 `connect` 会立即返回，但返回 `EINPROGRESS` 表示连接正在进行。
内核会在后台完成三次握手，此时通过 `epoll` 监听 `EPOLLOUT` 事件来判断握手是否完成。

（2）在 `connect` 执行前，socket 处于 `CLOSED` 状态。
此时注册 `EPOLLOUT` 事件可能会触发问题。因此，尽量避免在未连接时监听可写事件。
若必须在 `connect` 前监听事件，需确保：
- 仅监听 `EPOLLERR` 或等待 `connect` 调用后再添加 `EPOLLOUT`。
- 在 `connect` 后主动检查连接状态，而非依赖产生的事件。

##### client 使用  epoll_ctl  和 connect的先后顺序小结
遵循 ==先`connect`，再 `epoll_ctl`== 顺序。

```bash
4. 创建非阻塞 socket。
5. 调用 `connect`，捕获 `EINPROGRESS` 错误。
6. 将 socket 加入 `epoll` 监听 `EPOLLOUT` 事件。
```

```c
int sockfd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK, 0);

struct sockaddr_in addr = {/* 目标地址配置 */};
int ret = connect(sockfd, (struct sockaddr*)&addr, sizeof(addr));
if (ret < 0 && errno != EINPROGRESS) {
    // 立即失败的错误处理
}

// 添加至 epoll 监听可写事件
struct epoll_event event;
event.events = EPOLLOUT;
event.data.fd = sockfd;
epoll_ctl(epoll_fd, EPOLL_CTL_ADD, sockfd, &event);


int epoll_ret = epoll_wait(epoll_fd, events, MAX_EVENTS, timeout);
for (int i = 0; i < epoll_ret; i++) {
    if (events[i].events & EPOLLERR) {
    	// 出现 EPOLLERR 时候，通过getsockopt 获取具体的错误。
        int error = 0;
        socklen_t len = sizeof(error);
        getsockopt(events[i].data.fd, SOL_SOCKET, SO_ERROR, &error, &len);
        printf("连接错误: %s\n", strerror(error));
        close(events[i].data.fd);
        continue;
    }

    if (events[i].events & EPOLLOUT) {
        // 连接成功，继续处理数据
    }
}
```

### epoll_wait
```c
int epoll_wait(int epfd,struct epoll_event* events,int maxevents,int timeout);
```
#### 作用
收集在 `epoll`监控的事件中已经发生的事件，如果`epoll`中没有任何一个事件发生，则最多等待`timeout`毫秒后返回。
`epoll_wait`的返回值表示当前发生的事件个数，如果返回`0`，则表示本次调用中没有事件发生，如果返回`–1`，则表示出现错误，需要检查 `errno`错误码判断错误类型。

#### 说明
##### 参数说明
**epfd**：epoll的描述符。

**events**：分配好的 epoll_event结构体数组，epoll将会把发生的事件复制到 events数组中（events不可以是空指针，内核只负责把数据复制到这个 events数组中，不会去帮助我们在用户态中分配内存。内核这种做法效率很高）。

**maxevents**：表示本次可以返回的最大事件数目，通常 maxevents参数与预分配的events数组的大小是相等的。

**timeout**：表示在没有检测到事件发生时最多等待的时间（单位为毫秒）;
如果 timeout为0，则表示 epoll_wait在 rdylist链表中为空时，立刻返回，不会等待。
如果 timeout为-1，则表示一直等下去，直到有事件返回;
如果 timeout为任意正整数，则表示等这么长的时间，如果一直没有事件，则返回。

##### 返回值
`epoll_wait`返回有三种情况：  
(1) 有事件发生。  
(2) `timeout`被设置且时间到达。  
(3) 被信号中断。  
注：使用`epoll_pwait`可以设置阻塞期间阻塞的信号。

返回值：无出错返回值大于等于0，出错则返回-1并设置errno。

#### 工作流程
epoll_wait 的相关工作流程：
- 当内核监控的 fd 产生用户关注的事件，内核将 fd (`epi`)节点信息添加进就绪队列。
- 内核发现就绪队列有数据，唤醒进程工作。
- 内核先将 fd 信息从就绪队列中删除。
- 然后将 fd 对应就绪事件信息从内核空间拷贝到用户空间。
- 事件数据拷贝完成后，内核检查事件模式是 lt 还是 et，如果不是 et，重新将 fd 信息添加回就绪队列，下次重新触发 epoll_wait。

![](attachments/Pasted%20image%2020250531220453.png)


#### `epoll_wait`是否是阻塞的
`epoll_wait` 函数**通常是阻塞的**，但您可以非常灵活地控制它的阻塞行为，包括实现非阻塞或带超时阻塞。
`epoll_wait` 的主要目的是**等待**内核通知您在 epoll 实例上注册的文件描述符 (FDs) 上有事件发生（例如，Socket 可读、可写）。
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

 阻塞行为取决于 `timeout` 参数：
 
|**timeout 值**|**行为描述**|
|---|---|
|**`timeout > 0` (正值)**|**带超时阻塞**。`epoll_wait` 最多会阻塞 `timeout` 毫秒。如果在超时时间内有事件发生，它会立即返回；如果超时时间到了仍没有事件，它会返回 0。|
|**`timeout = 0`**|**非阻塞 (Non-blocking)**。`epoll_wait` 会立即返回。如果当前有事件就绪，它会返回事件数量；如果没有事件就绪，它会立即返回 0。|
|**`timeout = -1` (负值)**|**无限期阻塞**。`epoll_wait` 将一直阻塞，直到至少有一个注册的事件发生，或者被信号中断。|

#### `epoll_wait`是否阻塞和加入到`epoll`的`fd`的非阻塞的关系

您可能会混淆设置文件描述符的 `O_NONBLOCK` 标志和 `epoll_wait` 的行为。

**(1)`O_NONBLOCK` 的作用（设置在 Socket/FD 上）：** 
它决定了对该文件描述符执行的 I/O 操作（如 `read()`, `write()`, `accept()`, `connect()`）是否会阻塞。
设置后： 这些 I/O 函数会立即返回，如果数据未就绪，则返回 `-1` 并设置 `errno` 为 `EAGAIN` 或 `EWOULDBLOCK`。

**(2)`epoll_wait` 的作用（设置在 Epoll 实例上）：** 
它决定了**等待 I/O 事件通知**的函数是否阻塞。


##### 小结

`epoll_wait` 的阻塞行为**完全由其自身的 `timeout` 参数控制**，与您在注册的 Socket 上是否设置了 `O_NONBLOCK` 无关。
但在高性能网络编程中，最佳实践是：

1. **将所有注册到 epoll 上的 Socket 都设置为 `O_NONBLOCK`。**
2. **通过调整 `epoll_wait` 的 `timeout` 参数来控制事件等待的阻塞时长。**

这样可以确保程序在接收到事件后，对 Socket 进行的 `read`/`write` 操作不会意外地再次阻塞整个线程。


#### 唤醒的源码流程
`epoll_wait` 唤醒：
```c
tcp_v4_rcv -> socket.wq -> __wake_up_common -> ep_poll_callback -> eventpoll.wq -> wake_up_locked -> epoll_wait
注：socket.wq: listen-socket的wq；
```

详细代码，如下所示：
```c
/*
 * This is the callback that is passed to the wait queue wakeup
 * mechanism. It is called by the stored file descriptors when they
 * have events to report.
 */
static int ep_poll_callback(wait_queue_entry_t *wait, unsigned mode, int sync, void *key) {
    int pwake = 0;
    unsigned long flags;
    struct epitem *epi = ep_item_from_wait(wait);
    struct eventpoll *ep = epi->ep;
    __poll_t pollflags = key_to_poll(key);
    int ewake = 0;
    ...
    /* 检查已经发生的就绪事件 pollflags，是否是用户关注（epoll_ctl）的就绪事件，
     * 如果不是就返回。*/
    if (pollflags && !(pollflags & epi->event.events))
        goto out_unlock;
    ...
    /* 如果发生的事件是用户关注的事件，而且就绪列表上还没有添加这个 fd 节点 epi，
     * 那么将 epi 添加到就绪队列尾部。 */
    if (!ep_is_linked(epi)) {
        list_add_tail(&epi->rdllink, &ep->rdllist);
        ep_pm_stay_awake_rcu(epi);
    }

    /*
     * Wake up ( if active ) both the eventpoll wait list and the ->poll()
     * wait list.
     */
    if (waitqueue_active(&ep->wq)) {
        ...
        /* 唤醒通过 epoll_wait 睡眠的进程。 */
        wake_up_locked(&ep->wq);
    }
    ...
}

/* include/linux/wait.h */
#define wake_up_locked(x) __wake_up_locked((x), TASK_NORMAL, 1)

/* kernel/sched/wait.c */
void __wake_up_locked(struct wait_queue_head *wq_head, unsigned int mode, int nr) {
    __wake_up_common(wq_head, mode, nr, 0, NULL, NULL);
}

/* kernel/sched/wait.c */
static int __wake_up_common(struct wait_queue_head *wq_head, unsigned int mode,
            int nr_exclusive, int wake_flags, void *key,
            wait_queue_entry_t *bookmark) {
    wait_queue_entry_t *curr, *next;
    int cnt = 0;
    ...
    list_for_each_entry_safe_from(curr, next, &wq_head->head, entry) {
        unsigned flags = curr->flags;
        int ret;

        if (flags & WQ_FLAG_BOOKMARK)
            continue;

        /* 唤醒进程。 */
        ret = curr->func(curr, mode, wake_flags, key);
        if (ret < 0)
            break;
        /* 排它性唤醒属性，而且 nr_exclusive == 1，所以循环只执行一次就退出，
         * 也就是只唤醒了一个进程。 */
        if (ret && (flags & WQ_FLAG_EXCLUSIVE) && !--nr_exclusive)
            break;
        ...
    }
    ...
}
```


#### `epoll_wait`的源码实现流程
```bash
#------------------- *用户空间* ---------------------------
epoll_wait
#------------------- *内核空间* ---------------------------
|-- do_epoll_wait
    |-- ep_poll
        |-- ep_send_events
            |-- ep_scan_ready_list
                |-- ep_send_events_proc

```

详细代码，如下所示：
```c
/* ./fs/eventpoll.c */
struct eventpoll {
    ...
    /* List of ready file descriptors */
    struct list_head rdllist; /* 就绪队列。*/
    ...
}

/* fs/eventpoll.c */
SYSCALL_DEFINE4(epoll_wait, int, epfd, struct epoll_event __user *, events,
        int, maxevents, int, timeout) {
    return do_epoll_wait(epfd, events, maxevents, timeout);
}

/* fs/eventpoll.c */
static int do_epoll_wait(int epfd, struct epoll_event __user *events,
             int maxevents, int timeout) {
    ...
    error = ep_poll(ep, events, maxevents, timeout);
    ...
}

/* 检查就绪队列，如果就绪队列有就绪事件，就将事件信息从内核空间发送到用户空间。 */
static int ep_poll(struct eventpoll *ep, struct epoll_event __user *events, int maxevents, long timeout) {
    ...
    /* 检查就绪队列，如果有就绪事件就进入发送环节。 */
    ...
send_events:
    /* 有就绪事件就发送到用户空间，否则继续获取数据，直到阻塞等待超时。 */
    if (!res && eavail && !(res = ep_send_events(ep, events, maxevents)) &&
        !timed_out)
        goto fetch_events;
    ...
}

static int ep_send_events(struct eventpoll *ep,
              struct epoll_event __user *events, int maxevents) {
    struct ep_send_events_data esed;

    esed.maxevents = maxevents;
    esed.events = events;

    /* 遍历事件就绪队列，发送就绪事件到用户空间。 */
    ep_scan_ready_list(ep, ep_send_events_proc, &esed, 0, false);
    return esed.res;
}

static __poll_t ep_scan_ready_list(struct eventpoll *ep,
                  __poll_t (*sproc)(struct eventpoll *,
                       struct list_head *, void *),
                  void *priv, int depth, bool ep_locked) {
    ...
    /* 将就绪队列分片链接到 txlist 链表中。 */
    list_splice_init(&ep->rdllist, &txlist);
    /* 执行 ep_send_events_proc，事件数据从内核空间拷贝到内核空间的逻辑。*/
    res = (*sproc)(ep, &txlist, priv);
    ...
}

static __poll_t ep_send_events_proc(struct eventpoll *ep, struct list_head *head, void *priv) {
    ...
    // 遍历处理 txlist（原 ep->rdllist 数据）就绪队列结点，获取事件拷贝到用户空间。
    list_for_each_entry_safe (epi, tmp, head, rdllink) {
        if (esed->res >= esed->maxevents)
            break;
        ...
        /* 先从就绪队列中删除 epi。 */
        list_del_init(&epi->rdllink);

        /* 获取 epi 对应 fd 的就绪事件。 */
        revents = ep_item_poll(epi, &pt, 1);
        if (!revents)
            /* 如果没有就绪事件就返回（这时候，epi 已经从就绪队列中删除了。） */
            continue;

        /* 内核空间通过 __put_user 向用户空间拷贝传递数据。 */
        if (__put_user(revents, &uevent->events) ||
            __put_user(epi->event.data, &uevent->data)) {
            /* 如果拷贝失败，将 epi 重新保存回就绪队列，以便下一次处理。 */
            list_add(&epi->rdllink, head);
            ep_pm_stay_awake(epi);
            if (!esed->res)
                esed->res = -EFAULT;
            return 0;
        }

        /* 增加成功处理就绪事件的个数。 */
        esed->res++;
        uevent++;
        if (epi->event.events & EPOLLONESHOT)
            /* #define EP_PRIVATE_BITS (EPOLLWAKEUP | EPOLLONESHOT | EPOLLET | EPOLLEXCLUSIVE) */
            epi->event.events &= EP_PRIVATE_BITS;
        else if (!(epi->event.events & EPOLLET)) {
            /* lt 模式，重新将前面从就绪队列删除的 epi 添加回去。
             * 等待下一次 epoll_wait 调用，重新走上面的逻辑。
             * et 模式，前面从就绪队列里删除的 epi 将不会被重新添加，
             * 直到用户关注的事件再次发生。*/
            list_add_tail(&epi->rdllink, &ep->rdllist);
            ep_pm_stay_awake(epi);
        }
    }

    return 0;
}
```


### close


### 其他
#### read函数
man手册中read的关键信息如下。
```c
SYNOPSIS 
      #include <unistd.h>
       ssize_t read(int fd, void *buf, size_t count);
    
RETURN VALUE
       On  success, the number of bytes read is returned (zero indicates end of file), and the file position is advanced by this number.  It is not an error if this number
       is smaller than the number of bytes requested; this may happen for example because fewer bytes are actually available right now (maybe because we were close to end-
       of-file,  or  because  we  are  reading from a pipe, or from a terminal), or because read() was interrupted by a signal.  On error, -1 is returned, and errno is set
       appropriately.  In this case it is left unspecified whether the file position (if any) changes.
```
简单来说，「**read函数每次不一定能读取到指定的字节数。**」
例如，你要读取1024个字节的数据，而是实际上你read调用成功时，读取到的数据量很可能小于1024个字节。


##### `epoll/select`为什么要搭配非阻塞的IO的其他说明
我们现在知道了，`read`函数调用是不保证一定能读取到指定大小的字节数据的，且不说`select`存在`bug`，把不可读的`fd`返回了。

如果此时`fd`确实可读，「**那你知道要读取多少字节的数据？你是不知道的，你也无法知道。**」

即：多路复用`IO`只会通知有某`fd`有数据可读，却不会通知有多少数据可读。
所以没法通过一次`read(socket)`操作来保证把所有数据读完，你只能在一个`while`循环里不断地`read(socket)`，每次循环中都尝试读取一定大小的数据；

如果`socket`是`non-blocking`的，那么当你读完数据时最后一次`read`会返回没有可读数据的错误信息，然后你的读数据循环就可以结束；
如果此`socket`是`blocking`的，那么你的最后一个`read`操作会阻塞，直到新的数据到来，这不会是你想要的...；如果只是一次读入，那么实际要读的数据大于`read`的参数设置的缓冲区怎么办? 需要设置水平模式，每次触发读事件的产生，但是每次只进行一次读取，这样会导致性能不高。


###### 阻塞IO
此时如果fd被设置成阻塞的，则你可能前几次的read调用因为有数据，都立马返回了，但是下一次调用read函数就阻塞了，整个进程就被挂起了，直到read读到数据或则失败才会返回。

「**进程一旦被挂起，就无法充分使用cpu，进程就丧失了并发处理能力，就只能串行的处理每个请求。**」

###### 非阻塞IO

此时如果fd被设置成非阻塞的，则可以一直调用`read`函数，直到`read`函数调用失败，并返回`EWOULDBLOCK`或者`EAGAIN`的错误码（暂无数据可读），不会阻塞整个进程，进程可以去处理其他有数据可读的`fd`，从而才能充分使用`cpu`，提升进程的处理请求的能力。

##### read 和 write 对比
**（1）read**
`read`总是在接收缓冲区有数据时立即返回「水平触发模式下，有数据就返回；如果边缘触发，也不会有数据就返回，而缓冲区的可读数据长度发生变化时才返回」，而不是等到给定的`read buffer`填满时返回。  
只有当`receive buffer`为空时，`blocking`模式才会等待；
而`nonblock`模式下会立即返回`-1`（`errno = EAGAIN`或`EWOULDBLOCK`）

**(2) write**
`blocking`的`write`只有在发送缓冲区足以放下整个`buffer`时才返回（与`blocking read`并不相同）
> 即：`blocking`的`write` 如果发送缓冲区满了，或者发送缓冲区的空间不足以放下`write`设置的`buffer`大小，则会阻塞。
> 注：对于`blocking`的`write`有个特例：当`write`正阻塞等待时对面关闭了`socket`，则`write`则会立即将剩余缓冲区填满并返回所写的字节数，再次调用则`write`失败（`connection reset by peer`）。

`nonblock write`，即使发送缓冲区的空间不足以放下`write`设置的`buffer`大小，也进行返回，返回的是能够放下的字节数；发送缓冲区满了之后继续调用，则返回-1（`errno = EAGAIN`或`EWOULDBLOCK`）  


## 事件
epoll 支持的事件包括：
```bash
- EPOLLIN: 文件描述符可读
- EPOLLPRI: 优先级数据可读
- EPOLLOUT: 文件描述符可写
- EPOLLRDNORM: 文件描述符可读
- EPOLLRDBAND: 优先级数据可读
- EPOLLWRNORM: 文件描述符可写
- EPOLLWRBAND: 优先级数据可写
- EPOLLMSG: 消息可读
- EPOLLERR: 文件描述符发生错误
- EPOLLHUP: 文件描述符被挂起
- EPOLLRDHUP: 对端关闭了连接
- EPOLLEXCLUSIVE: 独占模式
- EPOLLWAKEUP: 唤醒模式
- EPOLLONESHOT: 一次性事件
- EPOLLET: 边缘触发模式
```

![](attachments/Pasted%20image%2020250512114224.png)

```c
/* Epoll event masks */
#define EPOLLIN		(__force __poll_t)0x00000001
#define EPOLLPRI	(__force __poll_t)0x00000002
#define EPOLLOUT	(__force __poll_t)0x00000004
#define EPOLLERR	(__force __poll_t)0x00000008
#define EPOLLHUP	(__force __poll_t)0x00000010
#define EPOLLNVAL	(__force __poll_t)0x00000020
#define EPOLLRDNORM	(__force __poll_t)0x00000040
#define EPOLLRDBAND	(__force __poll_t)0x00000080
#define EPOLLWRNORM	(__force __poll_t)0x00000100
#define EPOLLWRBAND	(__force __poll_t)0x00000200
#define EPOLLMSG	(__force __poll_t)0x00000400
#define EPOLLRDHUP	(__force __poll_t)0x00002000

/* Set exclusive wakeup mode for the target file descriptor */
#define EPOLLEXCLUSIVE (__force __poll_t)(1U << 28)

/*
 * Request the handling of system wakeup events so as to prevent system suspends
 * from happening while those events are being processed.
 *
 * Assuming neither EPOLLET nor EPOLLONESHOT is set, system suspends will not be
 * re-allowed until epoll_wait is called again after consuming the wakeup
 * event(s).
 *
 * Requires CAP_BLOCK_SUSPEND
 */
#define EPOLLWAKEUP (__force __poll_t)(1U << 29)

/* Set the One Shot behaviour for the target file descriptor */
#define EPOLLONESHOT (__force __poll_t)(1U << 30)

/* Set the Edge Triggered behaviour for the target file descriptor */
#define EPOLLET (__force __poll_t)(1U << 31)

```
参考：[man epoll_ctrl 说明](https://man7.org/linux/man-pages/man2/epoll_ctl.2.html)
### EPOLLIN
表示对应的文件描述符可以读（包括对端SOCKET正常关闭）；

#### 描述符读就绪条件
（1）套接字接收缓冲区中的字节数大于等于套接字接收缓冲区低水位标记的大小。低水位标记可以通过`SO_RCVLOWAT`套接字选项设置该套接字的低水位标记，对于TCP和UDP而言该值默认为1。

（2）连接半关闭：即接收到了`FIN`。这时`read`会返回0。

（3）监听套接字（`listen socket`）已完成的连接数(全连接队列)不为0。（但对该套接字调用`accept`可能阻塞。原因如下：在监听套接字「`listen socket`」为阻塞模式下，`accept`调用在已完成队列「全连接队列」为空时会阻塞。
如果某一个已完成连接的套接字触发了监听套接字「`listen socket`」可读，但是在`accept`之前客户端发送RST报文导致服务器内核从已完成队列里删除了这一项，此时已完成队列为空，`accept`阻塞。因此`io`复用中监听套接字也要是非阻塞的。
）

（4）套接字上有错误待处理，这时`read`返回`-1`，并将`errno`设置为对应错误。

### EPOLLOUT
表示对应的文件描述符可以写；

#### 描述符写就绪条件
（1）套接字发送缓冲区可用空间字节数大于等于套接字发送缓冲区低水位标记的大小，如果套接字是非阻塞的，我们调用`write`函数则会返回一正值。低水位标记可以通过`SO_SNDLOWAT`套接字选项设置套接字的低水位标记，对`TCP`和`UDP`而言该值通常默认为2048。

（2）该连接的写半部关闭，对这样的套接字继续写会产生`SIGPIPE`信号。（==一般我们将`SIGPIPE`信号设置为忽略，这样写操作将会返回`-1`并将`errno`设置为`EPIPE`==。）

（3）非阻塞模式的`connect`套接字完成连接或者产生错误。

（4）该套接字上有错误待处理。

#### 处理
对于`EPOLLOUT`：

（1）有写需要时才通过`epoll_ctl`添加相应`fd`的`EPOLLOUT`事件，不然在LT模式下会频繁触发;
（2）对于写操作，大部分情况下都处于可写状态，可先直接调用`write`来发送数据，直到返回 `EAGAIN「对于非阻塞的fd而言」`后再使能`EPOLLOUT`，待触发后再继续`write`。

### EPOLLERR
描述符产生错误时触发，默认监测事件。例如向一个远端close的套接字写数据。

### EPOLLHUP 和 EPOLLRDHUP
参考： [How do I use EPOLLHUP](https://stackoverflow.com/questions/6437879/how-do-i-use-epollhup)

#### EPOLLRDHUP
##### 背景
相信很多时候，大家都是通过检测 `read/recv` 返回 0 来判断对端是否关闭了连接，如果返回 0，我们通常也会 close 该连接。这没有问题，但在很多场景下有个缺陷：FIN 报文和普通数据报文一样，也是需要在缓冲区中排队的，只有当 read 读取到 FIN 以后才会返回 0；而且 FIN 报文无法和数据同时被读取，也就是说，必须将数据 read 完毕后，再调用一次 read 才能读取到 FIN 并返回 0 。这也是为什么在网络读取时需要将 read 放在循环中的原因之一，不仅是为了将数据读取完整，也是为了能够读到 FIN 报文。

> **注意，FIN 报文虽然会排队，但当本端收到 FIN 后，内核网络栈会立刻回复 ACK，而不会管你是否 read 到这个 FIN 报文。**

那么这个缺陷会引发什么问题呢？

- （1）在一个高并发网络场景下，服务器收到了对端发来的 FIN 报文（对端 close），但没有立即读取（正忙于处理已接收的数据），所以现在此连接处于半关状态，因为服务器的 read 还没有返回 0。直到服务器处理完其他事情后 read 并返回 0 才会 close 此连接。问题在于，这段时间内该连接被白白占用了，浪费了服务器的端口，这对高并发处理是不利的。服务器完全可以先关闭该连接，再去处理数据。

- （2）客户端向服务器发送文件，而文件的末端是 EOF，所以当服务器 read 到文件末端的 EOF 后返回 0，进而关闭连接。问题来了，万一客户还想继续发送文件呢？也就是说，**此时 read 返回 0 并不代表客户端想关闭连接** 。

因此，我们==应该尽量避免使用 read/recv 返回 0 来判断对端的关闭状态==。那还有什么方法？答案是 epoll 的 EPOLLRDHUP 和 EPOLLHUP 事件。这两者很容易混淆，下面略作区分。

##### 理解
EPOLLRDHUP **表示读关闭**，一般是对端描述符产生一个挂断事件(即对端关闭)，或者本端主动关闭读。
EPOLLRDHUP 可以作为一种读关闭的标志，注意**不能读的意思，是指内核不能再往内核缓冲区中增加新的内容。已经在内核缓冲区中的内容，用户态依然能够读取到。**

> 注： 对端关闭包括：`ctrl + c`, `kill`, `kill -9`。对端正常关闭「`close()`，`shell`下`kill`或`ctr+c`」，触发`EPOLLIN`和`EPOLLRDHUP`，但是不触发`EPOLLERR`和`EPOLLHUP`。

**当对方关闭（close）连接或者关闭写（shutdown(SHUT_WR)）时，EPOLLRDHUP事件就会被触发** 。所以 `EPOLLRDHUP` 被用来监听对方的连接状态。
==与前面 read 不同，**使用EPOLLRDHUP时，只要 FIN 报文进入了缓冲区，不管是否读取，都会引发 EPOLLRDHUP 事件==**。
因此，引发 `EPOLLRDHUP`的事件之后，此时缓冲区可能还是有数据需要读的，那么应该先处理`EPOLLIN` 事件读取缓冲区剩余数据，再执行`EPOLLRDHUP`事件的关闭处理。

##### 产生的场景
（1）对端发送 FIN (对端调用close 或者 shutdown(SHUT_WR)).
（2）本端调用 shutdown(SHUT_RD). 当然，关闭 SHUT_RD 的场景很少。


##### 处理动作
那么当 EPOLLRDHUP 发生时，我们该做什么呢？
**因为我不知道对方是 close 还是 shutdown(SHUT_WR)，如果是后者，我就还能够将处理好的数据发给对方；如果是前者，则发送数据后则会收到对方发来的 RST 报文，从而直接结束连接。** 
该做什么应该取决于应用场景，如果是 http 服务，那就不应该直接关闭连接，因为对端可能是 shutdown，且还需要接收数据（比如请求图片）；
如果是上传文件到服务器，那么就可以直接关闭连接，因为服务器不需要再向对端回复数据。

**存在`EPOLLRDHUP + EPOLLIN/EPOLLOUT`事件组合时，对`EPOLLRDHUP`的处理应该放在`EPOLLIN`和`EPOLLOUT`前面还是后面 ??**；
如果对 `EPOLLRDHUP`的处理，是本端关闭连接，那么就应该先处理`EPOLLIN`，因此此时缓冲区可能还有数据可读，然后再处理 `EPOLLRDHUP`事件。
对于`EPOLLOUT`的处理，可以放在 `EPOLLRDHUP`之后，或者不处理，因为 `EPOLLRDHUP`是对端关闭连接之后，本端才会触发的。此时再处理可写事件，给对端发送数据，没有意义。


##### 注意
注意：`EPOLLRDHUP`想要被触发，需要显式地在`epoll_ctl`调用时设置在`events`中；


#### EPOLLHUP

EPOLLHUP 则令人困惑，官方文档的解释是：当套接字挂起时，本事件被激发。然而什么是“挂起”却没有解释，网络讨论也说法不一。
一般认为，EPOLLHUP**表示读写都关闭**。本端描述符产生一个挂断事件，默认监测事件。

##### 产生的场景
（1）本端调用shutdown(SHUT_RDWR)。 不能是close，close 之后，文件描述符已经失效。
（2）本端调用 shutdown(SHUT_WR)，对端调用 shutdown(SHUT_WR)。
（3）本地调用 shutdown（SHUT_WR）并且收到了FIN。
（4）接收到了对端发送的RST。

注意：**值得一提的是，对端 close 连接时，不会触发本端的 EPOLLHUP；但对端同时 shutdown 读和写时（即 shutdown(SHUT_RDWR) ），则会触发本端的 EPOLLHUP** 。因为调用 shutdown(SHUT_RDWR) 只会关闭连接的读端和写端，不会释放文件描述符和其他相关资源，但此时该套接字已经处于“聋哑”状态，没有作用了，所以相当与被“挂起”。

（1）对已经关闭的socket继续读或者写：
**socket能检测到对方出错吗？目前为止，好像我还不知道如何检测**。  
**只有采取动作时，才能知道是否对方异常。即对方突然断掉（rst除外），是不会有此事件发生的。只有自己采取动作（当然自己此刻也不知道），read，write时，出EPOLLERR错，说明对方已经异常断开。**
但是，在给对端已经关闭的socket写时，会发生EPOLLERR；也就是说，只有在采取行动（比如  
读一个本端已经关闭的socket，或者写一个已经关闭的socket）时候，才知道对方是否关闭了。  
这个时候，如果对方已经关闭了，则会出现EPOLLERR，出现Error时，把对方DEL掉，close就可以  
了。

##### 范例
下面给出两种已经被实验证实会产生`EPOLLHUP`的情况：
- **(1) 收到对端发来的 RST 报文**
RST 报文用来重置连接，当一方发送RST报文后，对端会立即关闭连接，并释放相关资源。所以收到 RST 后，套接字相当于残废，被“挂起”。

- **(2) 将一个不可能触发该事件发生的套接字加入 EPOLL**
比如，使用 socket() 返回了一个套接字，既不 listen 也不 connect，这个套接字将没有任何事情发生（这也许就是“挂起”的含义），此时如果将其加入到 EPOLL 中，则会产生 EPOLLHUP 事件。



##### 发送 RST 的常见场景
1》系统崩溃重启(进程崩溃，只要内核是正常工作都还能兜底，发送的是FIN，不是这里讨论的RST)，四元组消失。此时收到任何数据，都会响应 RST.
2》设置 linger 参数，l_onoff 为 1 开启，但是 l_linger = 0 超时参数为0. 此时close() 将直接发送 RST.
3》接收缓冲区中还有数据，直接 close()， 接收缓冲区中的内容丢弃，直接发送 RST.
4》调用 close 时，close 会立马发送一个 FIN。注意：仅仅从 FIN 数据包上，无法断定对端是 close 还是仅仅 shutdown(SHUT_WR) 半关闭。往对端发送数据，若对端已经 close()，对端会回复 RST.


#### 注意
（1）EPOLLHUP 不能用来监听对方的关闭状态！
（2）对端发来 RST 信号，触发本端的 EPOLLIN + EPOLLRDHUP + EPOLLHUP + EPOLLERR 事件
（3）对端不管是 close 套接字，还是 shutdown 写，本端触发的都是 EPOLLIN + EPOLLRDHUP 事件，**因此，本端无从区分对端是 close 了套接字，还是 shutdown 了写** 。但有一点可以区分，如果对端是 close 了套接字，则本端在套接字上发送数据时，本端会收到对端发来的 RST 报文从而本端会触发 EPOLLIN + EPOLLRDHUP + EPOLLHUP + EPOLLERR 事件；而如果对端只是 shutdown 了写，则本端可以正常发送数据不会触发任何信号。
（4）当关闭（close）套接字时，内核会自动将套接字描述符从 epoll 中删除，因此本端不会再触发任何事件。


### EPOLLONESHOT
`ONESHOT`: 一次性。这个一次性是指我只接收一次该`fd`的可读/可写等通知，后续有的话也不要通知我了；
如果我再想要得到通知的话，就再使用：`epoll_ctl (efd, EPOLL_CTL_ADD, fd, &event)` 来添加一下。

#### EPOLLONESHOT时，如果缓冲区有未读取的数据，会立刻触发事件么？
Q：我调用`epoll_ctl (efd, EPOLL_CTL_ADD, fd, &event)`来把`socket`的`fd`添加到`ep`中时，那如果这个时候该`socket`是可读/可写的，后面我`epoll_wait`时能通知到我吗？
答案是：可以的。具体分析如下：
（1）重设`EPOLLONESHOT`在前，数据到来在后，这没有问题，因为重设`EPOLLONESHOT`时，同时`app`要带原有的`event`标志，等数据来时，就可以放入就绪列表中了。
（2）数据到来在前，重设`EPOLLONESHOT`在后，在重设`EPOLLONESHOT`时，设置完`event`标志后，内核会做一次`poll`动作，如果是就绪状态（可读、可写等），内核就把这个`fd`放入就绪列表中。


### EPOLLEXCLUSIVE
#### 背景
将同一个`listen-sockfd` 加入到多个线程本地的`epollfd`中, 以便达到`listen-sockfd`的`accept`的水平扩展（`scale out`）问题, 但是这样会带来一个问题，
惊群效应，即不必要的唤醒。


#### 解决
`EPOLLEXCLUSIVE` 是 2016 年 4.5+ 内核新添加的一个 epoll 的标识（代码改动较小，详看：[github](https://link.zhihu.com/?target=https%3A//github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898)）。

从 `Linux 4.5+` 开始出现的**LT水平触发**模式新增的 **`EPOLLEXCLUSIVE` 独占**标志，它降低了多个进程/线程通过 `epoll_ctl` 添加**共享 fd** 引发的惊群概率，使得一个事件发生时，只唤醒一个正在 `epoll_wait` 阻塞等待唤醒的进程/线程（而不是全部唤醒）。这个标志会保证一个事件只有一个 `epoll_wait`会被唤醒，避免了 “惊群效应”，并且可以在多个 CPU 之间很好的水平扩展。

当内核不支持`EPOLLEXCLUSIVE` 时，可以==通过 **ET 模式**下的 `EPOLLONESHOT`一次性 来模拟 `LT` + `EPOLLEXCLUSIVE` 的效果==，当然这样是有代价的，需要在每个事件处理完之后额外多调用一次 `epoll_ctl(EPOLL_CTL_MOD)` 重置这个 fd。这样做可以将负载均分到不同的 `CPU` 上，但是同一时刻，只能有一个 `worker` 调用 `accept(2)`。显然，这样又限制了处理 `accept(2)` 的吞吐。

#### 注意
(1) `EPOLLEXCLUSIVE` 是 **Linux 4.5 引入**的选项，用于解决多进程/线程下的`epoll`监听`同一 socket` 时的 **惊群问题**（thundering herd）。
(2) `EPOLLEXCLUSIVE` **必须与 `EPOLLLT`（水平触发，默认模式）配合使用**。
如果尝试在 `EPOLLET`（边缘触发）模式下使用，内核会直接忽略该标志（不会报错，但行为等同于未设置）。


### 其他
#### `EPOLLERR`和`EPOLLHUP`是默认监测事件
对于 `EPOLLERR`和`EPOLLHUP`，不需要在`epoll_ctl`中`epoll_event`时针对`fd`作设置，一样也会触发；

#### read方法返回0后还会有epollin事件吗
**（1）问题**
完整的问题是：
当`read`方法返回0，即我们收到了对方发给我们的fin包，使我们的`socket`处于`RCV_SHUTDOWN`状态，此后，该`socket`还会有`epollin`事件发生吗？
同理，我们调用`shutdown`方法，关闭了`send`端，使我们的`socket`处于`SEND_SHUTDOWN`状态，此后，还会有`epollout`事件吗？

**（2）分析**
当有任意`epoll`事件发生时，内核只是把该`socket`放到`epoll`的事件就绪队列里，等我们下次调用`epoll_wait`方法时，`epoll`内部会再调用这个队列里的各个`socket`的`tcp_poll`方法，检查该`socket`此时所有就绪的事件，然后将这些==事件集合==返回给用户。

`tcp/epoll`体系中关键的`tcp_poll`方法如下所示：该方法就是`epoll`检查`tcp-socket`有哪些就绪事件时调用的方法。
```c
// net/ipv4/tcp.c
__poll_t tcp_poll(struct file *file, struct socket *sock, poll_table *wait)
{
        __poll_t mask;
        struct sock *sk = sock->sk;
        const struct tcp_sock *tp = tcp_sk(sk);
        int state;
        ...
        state = inet_sk_state_load(sk);
        ...
        mask = 0;
        ...
        // 该socket的既是RCV_SHUTDOWN，又是SEND_SHUTDOWN，或者状态是TCP_CLOSE
        // 对应的epoll事件都是EPOLLHUP
        if (sk->sk_shutdown == SHUTDOWN_MASK || state == TCP_CLOSE)
                mask |= EPOLLHUP;
        
        // 该socket是RCV_SHUTDOWN，比如对方用shutdown(sockfd, SHUT_WR)方法
        // 关闭它的SEND_SHUTDOWN，也就是关闭了我们的RCV_SHUTDOWN
        // 又比如，我们用shutdown(sockfd, SHUT_RD)方法，关闭我们自己的RCV_SHUTDOWN
        // 在此模式下，epoll事件为EPOLLIN
        if (sk->sk_shutdown & RCV_SHUTDOWN)
                mask |= EPOLLIN | EPOLLRDNORM | EPOLLRDHUP;

        // 当我们的socket处于TCP_ESTABLISHED等状态时
        if (state != TCP_SYN_SENT &&
            (state != TCP_SYN_RECV || tp->fastopen_rsk)) {
                ...
                // 如果我们的socket里有可读字节，epoll对应的事件就是EPOLLIN
                if (tcp_stream_is_readable(tp, target, sk))
                        mask |= EPOLLIN | EPOLLRDNORM;

                if (!(sk->sk_shutdown & SEND_SHUTDOWN)) {
                        // 如果我们的socket有可写空间，epoll事件就是EPOLLOUT
                        if (sk_stream_is_writeable(sk)) {
                                mask |= EPOLLOUT | EPOLLWRNORM;
                        } else {
                                ...
                        }
                } else
                        // 如果我们的socket关闭了SEND_SHUTDOWN，epoll事件就是EPOLLOUT
                        mask |= EPOLLOUT | EPOLLWRNORM;
                ...
        } else if (state == TCP_SYN_SENT && inet_sk(sk)->defer_connect) {
                ...
        }
        ...
        // 如果我们的socket发生错误了，epoll事件就是EPOLLERR
        if (sk->sk_err || !skb_queue_empty(&sk->sk_error_queue))
                mask |= EPOLLERR;

        return mask;
}
EXPORT_SYMBOL(tcp_poll);
```
由该方法可见：
1> 只要`socket`处于`RCV_SHUTDOWN`状态，就一直有`epollin`事件;
2> 只要`socket`处于`SEND_SHUTDOWN`状态，就一直有`epollout`事件。


**(3) 范例代码**

```c
#include <arpa/inet.h>
#include <assert.h>
#include <errno.h>
#include <fcntl.h>
#include <stdio.h>
#include <stdlib.h>
#include <strings.h>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <unistd.h>

#define PORT 9999
#define MAX_EVENTS 10

static int tcp_listen() {
  int lfd, opt, err;
  struct sockaddr_in addr;

  lfd = socket(AF_INET, SOCK_STREAM, IPPROTO_TCP);
  assert(lfd != -1);

  opt = 1;
  err = setsockopt(lfd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
  assert(!err);

  bzero(&addr, sizeof(addr));
  addr.sin_family = AF_INET;
  addr.sin_addr.s_addr = INADDR_ANY;
  addr.sin_port = htons(PORT);

  err = bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
  assert(!err);

  err = listen(lfd, 8);
  assert(!err);

  return lfd;
}

static void epoll_ctl_add(int epfd, int fd, int evts) {
  struct epoll_event ev;
  ev.events = evts;
  ev.data.fd = fd;
  int err = epoll_ctl(epfd, EPOLL_CTL_ADD, fd, &ev);
  assert(!err);
}

static void handle_events(struct epoll_event *e, int epfd) {
  int err, m;
  char buf[8192];
  static int n;

  printf("sockfd %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN");
    e->events &= ~EPOLLIN;
    m = read(e->data.fd, buf, 1);
    assert(m == 0);
    printf("(read返回0) ");
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
        n++;
  }

  if (e->events & EPOLLHUP) {
    printf("EPOLLHUP ");
    e->events &= ~EPOLLHUP;
  }

  if (e->events & EPOLLERR) {
    printf("EPOLLERR ");
    e->events &= ~EPOLLERR;
  }

  assert(e->events == 0);
  printf("\n");

  if (n == 1) { // 连接建立成功后直接关闭receive端
    err = shutdown(e->data.fd, SHUT_RD);
    assert(!err);
  }

  while (n > 1) {
    err = write(e->data.fd, buf, sizeof(buf) / sizeof(buf[0]));
    if (err == -1) {
      assert(errno == EAGAIN);
      break;
    }
  }
}

int main(int argc, char *argv[]) {
  int epfd, lfd, cfd, err, n;
  struct epoll_event events[MAX_EVENTS];

  epfd = epoll_create1(0);
  assert(epfd != -1);

  lfd = tcp_listen();
  epoll_ctl_add(epfd, lfd, EPOLLIN);

  for (;;) {
    n = epoll_wait(epfd, events, MAX_EVENTS, -1);
    assert(n != -1);

    for (int i = 0; i < n; i++) {
      if (events[i].data.fd != lfd) {
        handle_events(&events[i], epfd);
        continue;
      }

      cfd = accept(lfd, NULL, NULL);
      assert(cfd != -1);

      err = fcntl(cfd, F_SETFL, O_NONBLOCK);
      assert(!err);

      epoll_ctl_add(epfd, cfd, EPOLLIN | EPOLLOUT | EPOLLET);
    }
  }

  return 0;
}

```

**(4) `epollin`事件的测试结果以及分析**
执行该程序后，用`ncat`对其进行连接，该程序所在终端的输出如下：
```c
$ gcc server.c && ./a.out
sockfd 5: EPOLLOUT
sockfd 5: EPOLLIN(read返回0) EPOLLOUT
sockfd 5: EPOLLIN(read返回0) EPOLLOUT
sockfd 5: EPOLLIN(read返回0) EPOLLOUT
sockfd 5: EPOLLIN(read返回0) EPOLLOUT
# 一直输出上面相同行 #
```

可以看到，当我们用`write`方式一直触发`epollout`事件时，`epollin`事件也在同时发生。
所以，即使我们`read`返回`0`，也不能保证之后不会发生`epollin`事件。




**(5) `epollout`事件的测试结果以及分析**
我们再来看下`epollout`事件是否也是这样。  
先将`handle_events`方法改成下面这样：

```c
static void handle_events(struct epoll_event *e, int epfd) {
  int err;
  static int n;

  printf("sockfd %d: ", e->data.fd);

  if (e->events & EPOLLIN) {
    printf("EPOLLIN ");
    e->events &= ~EPOLLIN;
  }

  if (e->events & EPOLLOUT) {
    printf("EPOLLOUT ");
    e->events &= ~EPOLLOUT;
    n++;
  }

  if (e->events & EPOLLHUP) {
    printf("EPOLLHUP ");
    e->events &= ~EPOLLHUP;
  }

  if (e->events & EPOLLERR) {
    printf("EPOLLERR ");
    e->events &= ~EPOLLERR;
  }

  assert(e->events == 0);
  printf("\n");

  if (n == 1) { // 连接建立成功后直接关闭send端
    err = shutdown(e->data.fd, SHUT_WR);
    assert(!err);
  }
}
```

运行该程序后，用`ncat`对其建立`tcp`连接，然后一直在`ncat`终端输入数据，你会看到运行我们程序的终端有如下输出：
```c
$ gcc server.c && ./a.out
sockfd 5: EPOLLOUT
sockfd 5: EPOLLOUT
sockfd 5: EPOLLOUT
sockfd 5: EPOLLIN EPOLLOUT
sockfd 5: EPOLLIN EPOLLOUT
sockfd 5: EPOLLIN EPOLLOUT
# 一直输出上面相同行 #
```

由上可见，即使我们关闭了`send`端，`epollout`事件还是会返回。


**(6)小结**：
1> 结论：
当`read`方法返回0，即我们收到了对方发给我们的`fin`包，使我们的`socket`处于`RCV_SHUTDOWN`状态，此后，该`socket`还会有`epollin`事件。
同理，我们调用`shutdown`方法，关闭了`send`端，使我们的`socket`处于`SEND_SHUTDOWN`状态，此后，还会有`epollout`事件。


2> 处理：
如果我们不想要这种结果呢？比如说，当`read`返回0后，就不要再返回`epollin`事件，这怎么做呢？  
其实说来也简单，你只要把你不想要的事件从`epoll`注册中移除就好了。虽然`epoll`还是会调用`tcp_poll`方法，返回的`socket`事件还是包含所有的就绪事件，但它在返回给用户时，会过滤掉我们不感兴趣的事件。所以，当`read`返回`0`时，你只要把`epollin`事件从`epoll`注册中取消，以后就再也不会有这个事件发生了。

#### accept相关
accept接收对端连接之前，会触发`EPOLLIN`事件的产生，然后进行accept创建新的连接。
这里可以循环多次调用`accept`, 直至返回 `EAGAIN`， 同时适用于LT和ET。

##### 场景：client`connect` 成功后，Server 尚未 `accept`，Client 又关闭了连接
无论是TCP socket, 还是 UDS(unix domain socket), client`connect` 成功（连接建立）后，Server 尚未 `accept`，Client 又关闭连接。此时server端的行为。

##### 问题一：server 端还没有来得及调用epoll_wait, client关闭后，server调用epoll_wait是否会产生事件
##### 问题二：server端epoll_wait产生了事件，还没来得及accept，client关闭后，server进行accept会产生什么
##### 问题三：server端完成了accept，然后client关闭后，server在新的new_fd（没有通过epoll_add添加epoo中）上直接读写会产生什么
Server 对 `new_fd` 的读写操作将根据 Client 关闭的方式和 Server 的操作类型产生不同的结果。

注意：由于 `new_fd` 没有加入 `epoll`，Server 无法通过 I/O 多路复用机制异步感知 Client 的关闭，必须通过主动进行 `read()` 或 `write()` 操作（即同步 I/O 或轮询）才能发现连接已断开。


|**Server 操作**|**结果**|**解释**|
|---|---|---|
|**`read(new_fd)`**|**返回 0**。|Client 正常关闭连接（发送 FIN）。Server 读取到 EOF（文件结束）。这是检测对端正常关闭的标准方法。|
|**`write(new_fd)`**|**返回 `-1`，`errno` 为 `EPIPE`**。|Client 关闭连接后，如果 Server 仍然尝试写入，会触发 `SIGPIPE` 信号。如果 Server 忽略或处理了 `SIGPIPE` 信号（这是推荐做法），则 `write()` 调用会失败，并返回 `errno=EPIPE` (Broken pipe)。|



##### 问题四：server端完成了accept，然后client关闭后，server在新的new_fd（通过epoll_add添加到epoll中）上读写会产生什么

Server 将通过 `epoll_wait` 异步收到关闭事件。

**分析：**
3. **`epoll_add`：** Server 将 `new_fd` 加入 `epoll` 实例，通常监听 `EPOLLIN` 和 **`EPOLLRDHUP`** (Read Hang Up)。
4. **Client 关闭：** Client 关闭连接（发送 FIN）。
5. **`epoll_wait` 结果：** `epoll_wait` 会返回 `new_fd` 上的事件：
    - **`EPOLLIN`：** 因为 FIN 包被内核视为可读事件（表示可以读取 0 字节）。
    - **`EPOLLRDHUP`：** 这是 Linux 专有的事件，**明确指示**对端已关闭连接的写入端，但 Server 自己的写入端仍可使用。

6. **Server 对事件的响应：**
    - 当 Server 收到 `EPOLLIN` 或 `EPOLLRDHUP` 事件后，应该对 `new_fd` 调用 **`read()`**。
    - **`read()` 将返回 0**，确认连接已正常关闭。
    - 收到这些事件后，Server 应该调用 `close(new_fd)` 并将该文件描述符从 `epoll` 实例中移除 (`EPOLL_CTL_DEL`)。


##### TCP socket 测试
```c
# cat epoll_tcp_robust_test.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/epoll.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <sys/wait.h>
#include <time.h>

#define TCP_PORT 8888
#define MAX_EVENTS 10
#define LISTEN_BACKLOG 5
#define TIMEOUT_MS 500
#define SERVER_IP "127.0.0.1"

// --- 辅助函数 ---

/**
 * @brief 设置文件描述符为非阻塞模式
 */
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        perror("fcntl F_GETFL");
        exit(EXIT_FAILURE);
    }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl F_SETFL O_NONBLOCK");
        exit(EXIT_FAILURE);
    }
}

/**
 * @brief 尝试在 listen_fd 上调用 accept() 并打印结果
 */
void try_accept(int listen_fd, const char* name) {
    int conn_fd;
    printf("[%s] Server: 尝试调用 accept()...\n", name);
    conn_fd = accept(listen_fd, NULL, NULL);

    if (conn_fd == -1) {
        if (errno == EWOULDBLOCK || errno == EAGAIN) {
            printf("[%s] Server 成功: accept 阻塞/队列为空。符合预期。\n", name);
        } else if (errno == ECONNABORTED || errno == EPROTO) {
            // TCP 中止连接的典型错误
            printf("[%s] Server 成功: accept 失败，errno=%d (%s)。连接在被接受前中止。\n",
                   name, errno, (errno == ECONNABORTED ? "ECONNABORTED" : "EPROTO"));
        } else {
            perror("Server 失败: accept 失败，错误");
        }
    } else {
        // UDS 场景中这里成功了，对于 TCP，这通常是失败的信号
        printf("[%s] Server 失败: accept 意外成功，返回新的 fd %d。\n", name, conn_fd);

        // 关键验证：检查新连接的状态 (如果 read() 返回 0，说明连接已断开)
        char test_buf[1];
        ssize_t nread = read(conn_fd, test_buf, 1);
        if (nread == 0) {
            printf("[%s] Server 验证: new_fd 上的 read() 返回 0 (EOF)。连接实际已断开。\n", name);
        } else {
            printf("[%s] Server 验证: new_fd 上的 read() 返回 %zd。连接意外存活。\n", name, nread);
        }

        close(conn_fd);
    }
}

// --- Client 进程函数 ---

/**
 * @brief 客户端连接并立即关闭
 */
void client_connect_and_close(const char *name) {
    int sfd;
    struct sockaddr_in addr;

    if ((sfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("Client socket");
        _exit(EXIT_FAILURE);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(TCP_PORT);
    inet_pton(AF_INET, SERVER_IP, &addr.sin_addr);

    // 尝试连接
    if (connect(sfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        // connect 可能会在 server 尚未完全准备好时失败
        // perror("Client connect failed");
    } else {
        printf("[%s] Client: 三次握手成功。\n", name);
        usleep(50000); // 50ms 确保 Server 内核将连接放入完成队列
        close(sfd);
        printf("[%s] Client: 连接立即关闭 (四次挥手/RST)。\n", name);
    }

    _exit(EXIT_SUCCESS);
}

/**
 * @brief 客户端连接并保持连接打开，直到收到信号
 */
void client_connect_and_hold(const char *name, int pipe_read_fd) {
    int sfd;
    struct sockaddr_in addr;
    char buffer;

    if ((sfd = socket(AF_INET, SOCK_STREAM, 0)) == -1) {
        perror("Client holder socket");
        _exit(EXIT_FAILURE);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(TCP_PORT);
    inet_pton(AF_INET, SERVER_IP, &addr.sin_addr);

    if (connect(sfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        // perror("Client holder connect failed");
        close(sfd);
        _exit(EXIT_FAILURE);
    }

    printf("[%s] Client: 三次握手成功，保持打开。\n", name);

    // 阻塞在管道上，等待父进程发出关闭信号
    if (read(pipe_read_fd, &buffer, 1) == 1 && buffer == 'C') {
        // 收到关闭信号
        printf("[%s] Client: 收到关闭指令，正在关闭连接。\n", name);
        close(sfd); // 发送 FIN
    } else {
        close(sfd);
    }

    close(pipe_read_fd);
    _exit(EXIT_SUCCESS);
}

// --- 场景实现 ---

/**
 * @brief 场景一：Server 尚未调用 epoll_wait，Client 关闭后，Server 调用 epoll_wait 是否会产生事件
 */
void run_scenario_one(int epoll_fd, int listen_fd) {
    printf("\n--- 场景一: epoll_wait 前 Client 关闭 (预期: epoll_wait 无事件, accept 失败) ---\n");

    // 1. 模拟 Client: 连接成功并立即关闭
    pid_t pid = fork();
    if (pid == 0) {
        client_connect_and_close("S1");
    }
    waitpid(pid, NULL, 0); // 等待 Client 进程退出，确保连接已关闭

    // 2. Server: 稍后调用 epoll_wait
    struct epoll_event events[MAX_EVENTS];
    printf("[S1] Server: Client已关闭。调用 epoll_wait 验证是否有事件。\n");
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, TIMEOUT_MS);

    if (nfds == 0) {
        printf("[S1] Server 成功: epoll_wait 返回 0 (超时)，符合预期（内核已清理中止连接）。\n");
    } else if (nfds > 0) {
        printf("[S1] Server: 接收到 %d 个事件 (与预期不符)。尝试 accept 验证...\n", nfds);
        try_accept(listen_fd, "S1");
    } else {
        perror("[S1] epoll_wait error");
    }
}

/**
 * @brief 场景二：epoll_wait 产生事件，未 accept，Client 关闭后，Server 进行 accept 会产生什么
 */
void run_scenario_two(int epoll_fd, int listen_fd) {
    printf("\n--- 场景二: epoll_wait 后，accept 前 Client 关闭 (预期: epoll_wait 有事件，accept 失败) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    // 1. 模拟 Client: 连接成功，保持打开
    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]);
        client_connect_and_hold("S2", pipefd[0]);
    }
    close(pipefd[0]);

    usleep(100000); // 100ms 确保 Client 连接成功

    // 2. Server: 等待事件
    struct epoll_event events[MAX_EVENTS];
    printf("[S2] Server: 等待连接事件...\n");
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);

    if (nfds > 0 && events[0].data.fd == listen_fd) {
        printf("[S2] Server: 收到 EPOLLIN 事件 (%d)。\n", events[0].events);

        // 3. Client: 在 Server 尚未 accept 时关闭连接
        write(pipefd[1], "C", 1);
        close(pipefd[1]);
        waitpid(pid, NULL, 0);
        printf("[S2] Client: 已关闭连接。\n");

        // 4. Server: 尝试 accept
        try_accept(listen_fd, "S2");

    } else {
        printf("[S2] Server 失败: 未收到预期的连接事件。\n");
        if (pid > 0) { kill(pid, SIGTERM); waitpid(pid, NULL, 0); }
    }
}

// 场景三和四的实现与 UDS 场景一致，因为一旦连接建立，读写行为与协议栈无关

/**
 * @brief 场景三：Server 完成 accept，Client 关闭后，Server 在 new_fd 上直接读写会产生什么
 */
void run_scenario_three(int listen_fd) {
    printf("\n--- 场景三: accept 后，未加 epoll，Client 关闭 (预期: read=0, write=EPIPE) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]);
        client_connect_and_hold("S3", pipefd[0]);
    }
    close(pipefd[0]);

    usleep(100000);

    // 2. Server: 接受连接
    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd == -1) {
        perror("[S3] Server accept failed");
        close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }
    printf("[S3] Server: 成功 accept 新连接 fd %d。\n", conn_fd);

    // 3. Client: 关闭连接
    write(pipefd[1], "C", 1);
    close(pipefd[1]);
    waitpid(pid, NULL, 0);
    printf("[S3] Client: 已关闭连接。\n");

    // 4. Server: 验证读操作
    char buf[10];
    ssize_t nread = read(conn_fd, buf, 10);
    if (nread == 0) {
        printf("[S3] Server 成功: read() 返回 0 (EOF)，符合预期。\n");
    } else {
        printf("[S3] Server 失败: read() 返回 %zd, 不符合预期。\n", nread);
    }

    // 5. Server: 验证写操作
    signal(SIGPIPE, SIG_IGN);
    ssize_t nwrite = write(conn_fd, "test", 4);
    if (nwrite == -1 && errno == EPIPE) {
        printf("[S3] Server 成功: write() 失败，errno=%d (EPIPE/Broken pipe)，符合预期。\n", EPIPE);
    } else {
        printf("[S3] Server 失败: write() 返回 %zd, errno=%d，不符合预期。\n", nwrite, errno);
    }
    signal(SIGPIPE, SIG_DFL);

    close(conn_fd);
}

/**
 * @brief 场景四：Server 完成 accept，Client 关闭后，Server 在 new_fd (已加入 epoll) 上读写会产生什么
 */
void run_scenario_four(int epoll_fd, int listen_fd) {
    printf("\n--- 场景四: accept 后，已加 epoll，Client 关闭 (预期: new_fd 收到 EPOLLRDHUP/EPOLLIN) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]);
        client_connect_and_hold("S4", pipefd[0]);
    }
    close(pipefd[0]);

    usleep(100000);

    // 2. Server: 接受连接
    struct epoll_event listen_events[MAX_EVENTS];
    epoll_wait(epoll_fd, listen_events, MAX_EVENTS, -1);

    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd == -1) {
        perror("[S4] Server accept failed");
        close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }
    set_nonblocking(conn_fd);
    printf("[S4] Server: 成功 accept 新连接 fd %d。\n", conn_fd);

    // 3. Server: 注册 new_fd 到 epoll
    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLRDHUP | EPOLLET;
    ev.data.fd = conn_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev) == -1) {
        perror("[S4] epoll_ctl: conn_fd");
        close(conn_fd); close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }

    // 4. Client: 关闭连接
    write(pipefd[1], "C", 1);
    close(pipefd[1]);
    waitpid(pid, NULL, 0);
    printf("[S4] Client: 已关闭连接。\n");

    // 5. Server: 等待 new_fd 上的事件
    struct epoll_event conn_events[MAX_EVENTS];
    printf("[S4] Server: 等待 new_fd 上的关闭事件...\n");
    int nfds = epoll_wait(epoll_fd, conn_events, MAX_EVENTS, TIMEOUT_MS * 2);

    if (nfds > 0 && conn_events[0].data.fd == conn_fd) {
        unsigned int events_mask = conn_events[0].events;
        printf("[S4] Server 成功: 收到 new_fd 事件掩码 0x%X。\n", events_mask);

        if (events_mask & EPOLLRDHUP) {
            printf("[S4] Server 成功: 捕获到 EPOLLRDHUP (对端关闭写入端) 事件。\n");
        }
        if (events_mask & EPOLLIN) {
            char buffer[10];
            ssize_t nread = read(conn_fd, buffer, 10);
            if (nread == 0) {
                printf("[S4] Server 成功: read() 返回 0 (EOF)，连接已正常关闭。\n");
            } else {
                printf("[S4] Server 失败: read() 返回 %zd, 连接未正确关闭。\n", nread);
            }
        }
    } else {
        printf("[S4] Server 失败: 未收到 new_fd 上的关闭事件。\n");
    }

    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, conn_fd, NULL);
    close(conn_fd);
}


// --- 主函数 ---
int main() {
    int listen_fd, epoll_fd;
    struct sockaddr_in addr;
    int optval = 1; // 用于 setsockopt

    // 1. 创建和配置 Server Listen Socket
    if ((listen_fd = socket(AF_INET, SOCK_STREAM, 0)) == -1) { perror("socket"); exit(EXIT_FAILURE); }

    // 允许地址重用，避免 TIME_WAIT 导致的 bind 失败
    if (setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &optval, sizeof(optval)) == -1) {
        perror("setsockopt SO_REUSEADDR");
        close(listen_fd);
        exit(EXIT_FAILURE);
    }

    set_nonblocking(listen_fd);

    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_port = htons(TCP_PORT);
    addr.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) == -1) { perror("bind"); goto cleanup; }
    if (listen(listen_fd, LISTEN_BACKLOG) == -1) { perror("listen"); goto cleanup; }
    printf("Server: TCP 监听中于 %s:%d\n", SERVER_IP, TCP_PORT);

    // 2. 创建 epoll 实例并添加 listen_fd
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) { perror("epoll_create1"); goto cleanup; }

    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) { perror("epoll_ctl: listen_fd"); goto cleanup_epoll; }

    // 运行测试场景
    run_scenario_one(epoll_fd, listen_fd);
    run_scenario_two(epoll_fd, listen_fd);

    // 准备 S3/S4 前，清理可能残留的 listen_fd 事件，并确保队列是空的
    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, listen_fd, NULL);
    int temp_fd;
    while((temp_fd = accept(listen_fd, NULL, NULL)) != -1) { close(temp_fd); }

    // 重新添加 listen_fd (S4需要它来触发)
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) { perror("epoll_ctl re-add"); goto cleanup_epoll; }

    run_scenario_three(listen_fd);
    run_scenario_four(epoll_fd, listen_fd);

    // 清理
cleanup_epoll:
    close(epoll_fd);
cleanup:
    close(listen_fd);
    printf("\n--- 所有测试完成，资源已清理。---\n");

    return 0;
}
```

```bash
# ./epoll_tcp_robust_test
Server: TCP 监听中于 127.0.0.1:8888

--- 场景一: epoll_wait 前 Client 关闭 (预期: epoll_wait 无事件, accept 失败) ---
[S1] Client: 三次握手成功。
[S1] Client: 连接立即关闭 (四次挥手/RST)。
[S1] Server: Client已关闭。调用 epoll_wait 验证是否有事件。
[S1] Server: 接收到 1 个事件 (与预期不符)。尝试 accept 验证...
[S1] Server: 尝试调用 accept()...
[S1] Server 失败: accept 意外成功，返回新的 fd 5。
[S1] Server 验证: new_fd 上的 read() 返回 0 (EOF)。连接实际已断开。

--- 场景二: epoll_wait 后，accept 前 Client 关闭 (预期: epoll_wait 有事件，accept 失败) ---
[S2] Client: 三次握手成功，保持打开。
[S2] Server: 等待连接事件...
[S2] Server: 收到 EPOLLIN 事件 (1)。
[S2] Client: 收到关闭指令，正在关闭连接。
[S2] Client: 已关闭连接。
[S2] Server: 尝试调用 accept()...
[S2] Server 失败: accept 意外成功，返回新的 fd 5。
[S2] Server 验证: new_fd 上的 read() 返回 0 (EOF)。连接实际已断开。

--- 场景三: accept 后，未加 epoll，Client 关闭 (预期: read=0, write=EPIPE) ---
[S3] Client: 三次握手成功，保持打开。
[S3] Server: 成功 accept 新连接 fd 5。
[S3] Client: 收到关闭指令，正在关闭连接。
[S3] Client: 已关闭连接。
[S3] Server 成功: read() 返回 0 (EOF)，符合预期。
[S3] Server 失败: write() 返回 4, errno=11，不符合预期。

--- 场景四: accept 后，已加 epoll，Client 关闭 (预期: new_fd 收到 EPOLLRDHUP/EPOLLIN) ---
[S4] Client: 三次握手成功，保持打开。
[S4] Server: 成功 accept 新连接 fd 5。
[S4] Client: 收到关闭指令，正在关闭连接。
[S4] Client: 已关闭连接。
[S4] Server: 等待 new_fd 上的关闭事件...
[S4] Server 成功: 收到 new_fd 事件掩码 0x2001。
[S4] Server 成功: 捕获到 EPOLLRDHUP (对端关闭写入端) 事件。
[S4] Server 成功: read() 返回 0 (EOF)，连接已正常关闭。

--- 所有测试完成，资源已清理。---
```

##### unix domain socket（uds）测试

```c
# cat epoll_unix_robust_test.c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/un.h>
#include <sys/epoll.h>
#include <errno.h>
#include <fcntl.h>
#include <signal.h>
#include <sys/wait.h>
#include <time.h>

#define SOCKET_PATH "/tmp/epoll_uds_robust_test.sock"
#define MAX_EVENTS 10
#define LISTEN_BACKLOG 5
#define TIMEOUT_MS 500

// --- 辅助函数 ---

/**
 * @brief 设置文件描述符为非阻塞模式
 */
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    if (flags == -1) {
        perror("fcntl F_GETFL");
        exit(EXIT_FAILURE);
    }
    if (fcntl(fd, F_SETFL, flags | O_NONBLOCK) == -1) {
        perror("fcntl F_SETFL O_NONBLOCK");
        exit(EXIT_FAILURE);
    }
}

/**
 * @brief 尝试在 listen_fd 上调用 accept() 并打印结果
 */
void try_accept(int listen_fd, const char* name) {
    int conn_fd;
    printf("[%s] Server: 尝试调用 accept()...\n", name);
    conn_fd = accept(listen_fd, NULL, NULL);

    if (conn_fd == -1) {
        if (errno == EWOULDBLOCK || errno == EAGAIN) {
            printf("[%s] Server 成功: accept 阻塞/队列为空。符合预期。\n", name);
        } else if (errno == ECONNABORTED || errno == EPROTO) {
            // Client 在 accept 前关闭的典型错误
            printf("[%s] Server 成功: accept 失败，errno=%d (%s)。连接在被接受前中止。\n",
                   name, errno, (errno == ECONNABORTED ? "ECONNABORTED" : "EPROTO"));
        } else {
            perror("Server 失败: accept 失败");
        }
    } else {
        printf("[%s] Server 失败: accept 成功，返回新的 fd %d。\n", name, conn_fd);
        close(conn_fd);
    }
}

// --- Client 进程函数 ---

/**
 * @brief 客户端连接并立即关闭
 */
void client_connect_and_close(const char *name) {
    int sfd;
    struct sockaddr_un addr;

    if ((sfd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
        perror("Client socket");
        _exit(EXIT_FAILURE);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    // 尝试连接
    if (connect(sfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        // 如果连接失败，打印错误但不一定退出，让Server进程决定下一步
        // (在某些时序下，connect可能会失败)
        // perror("Client connect failed");
    } else {
        printf("[%s] Client: 连接建立成功。\n", name);
        usleep(50000); // 50ms 确保 Server 内核将连接放入队列
        close(sfd);
        printf("[%s] Client: 连接立即关闭。\n", name);
    }

    _exit(EXIT_SUCCESS);
}

/**
 * @brief 客户端连接并保持连接打开，直到收到信号或父进程终止
 */
void client_connect_and_hold(const char *name, int pipe_read_fd) {
    int sfd;
    struct sockaddr_un addr;
    char buffer;

    if ((sfd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) {
        perror("Client socket");
        _exit(EXIT_FAILURE);
    }

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (connect(sfd, (struct sockaddr*)&addr, sizeof(addr)) == -1) {
        // perror("Client connect failed");
        _exit(EXIT_FAILURE);
    }

    printf("[%s] Client: 连接建立成功，保持打开。\n", name);

    // 阻塞在管道上，等待父进程发出关闭信号
    if (read(pipe_read_fd, &buffer, 1) == 1 && buffer == 'C') {
        // 收到关闭信号
        printf("[%s] Client: 收到关闭指令，正在关闭连接。\n", name);
        close(sfd);
    } else {
        // 发生错误或父进程终止，直接关闭
        close(sfd);
    }

    close(pipe_read_fd);
    _exit(EXIT_SUCCESS);
}

// --- 场景实现 ---

/**
 * @brief 场景一：Server 尚未调用 epoll_wait，Client 关闭后，Server 调用 epoll_wait 是否会产生事件
 */
void run_scenario_one(int epoll_fd, int listen_fd) {
    printf("\n--- 场景一: epoll_wait 前关闭 (预期: epoll_wait 无事件) ---\n");

    // 1. 模拟 Client: 连接成功并立即关闭
    pid_t pid = fork();
    if (pid == 0) {
        client_connect_and_close("S1");
    }
    waitpid(pid, NULL, 0); // 等待 Client 进程退出，确保连接已关闭和内核已处理

    // 2. Server: 稍后调用 epoll_wait
    struct epoll_event events[MAX_EVENTS];
    printf("[S1] Server: Client已关闭。调用 epoll_wait 验证是否有事件。\n");
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, TIMEOUT_MS);

    if (nfds == 0) {
        printf("[S1] Server 成功: epoll_wait 返回 0 (超时)，符合预期（内核已清理中止连接）。\n");
    } else if (nfds > 0) {
        printf("[S1] Server 失败: 接收到 %d 个事件 (但预期是 0)。尝试 accept 验证...\n", nfds);
        // 如果收到事件，调用 accept 验证其状态
        try_accept(listen_fd, "S1");
    } else {
        perror("[S1] epoll_wait error");
    }
}

/**
 * @brief 场景二：epoll_wait 产生事件，未 accept，Client 关闭后，Server 进行 accept 会产生什么
 */
void run_scenario_two(int epoll_fd, int listen_fd) {
    printf("\n--- 场景二: epoll_wait 后，accept 前关闭 (预期: epoll_wait 有事件，accept 失败) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    // 1. 模拟 Client: 连接成功，保持打开
    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]); // 关闭写端
        client_connect_and_hold("S2", pipefd[0]);
    }
    close(pipefd[0]); // 关闭读端 (Server 持有写端)

    usleep(100000); // 100ms 确保 Client 连接成功

    // 2. Server: 等待事件
    struct epoll_event events[MAX_EVENTS];
    printf("[S2] Server: 等待连接事件...\n");
    int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);

    if (nfds > 0 && events[0].data.fd == listen_fd) {
        printf("[S2] Server: 收到 EPOLLIN 事件 (%d)。\n", events[0].events);

        // 3. Client: 在 Server 尚未 accept 时关闭连接
        write(pipefd[1], "C", 1); // 发出关闭指令
        close(pipefd[1]); // 关闭管道，并等待 Client 进程退出
        waitpid(pid, NULL, 0);
        printf("[S2] Client: 已关闭连接。\n");

        // 4. Server: 尝试 accept
        try_accept(listen_fd, "S2");

    } else {
        printf("[S2] Server 失败: 未收到预期的连接事件。\n");
        // 如果超时，确保关闭 Client 进程
        if (pid > 0) { kill(pid, SIGTERM); waitpid(pid, NULL, 0); }
    }
}

/**
 * @brief 场景三：Server 完成 accept，Client 关闭后，Server 在 new_fd 上直接读写会产生什么
 */
void run_scenario_three(int listen_fd) {
    printf("\n--- 场景三: accept 后，未加 epoll，Client 关闭 (预期: read=0, write=EPIPE) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    // 1. 模拟 Client: 连接成功，保持打开
    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]); // 关闭写端
        client_connect_and_hold("S3", pipefd[0]);
    }
    close(pipefd[0]); // Server 持有写端

    usleep(100000); // 确保 Client 连接成功

    // 2. Server: 接受连接
    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd == -1) {
        perror("[S3] Server accept failed");
        close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }
    printf("[S3] Server: 成功 accept 新连接 fd %d。\n", conn_fd);

    // 3. Client: 关闭连接
    write(pipefd[1], "C", 1); // 发出关闭指令
    close(pipefd[1]);
    waitpid(pid, NULL, 0); // 等待 Client 进程退出
    printf("[S3] Client: 已关闭连接。\n");

    // 4. Server: 验证读操作
    char buf[10];
    ssize_t nread = read(conn_fd, buf, 10);
    if (nread == 0) {
        printf("[S3] Server 成功: read() 返回 0 (EOF)，符合预期。\n");
    } else {
        printf("[S3] Server 失败: read() 返回 %zd, 不符合预期。\n", nread);
    }

    // 5. Server: 验证写操作（需要屏蔽 SIGPIPE 信号）
    signal(SIGPIPE, SIG_IGN); // 临时忽略 SIGPIPE
    ssize_t nwrite = write(conn_fd, "test", 4);
    if (nwrite == -1 && errno == EPIPE) {
        printf("[S3] Server 成功: write() 失败，errno=32 (EPIPE/Broken pipe)，符合预期。\n");
    } else {
        printf("[S3] Server 失败: write() 返回 %zd, errno=%d，不符合预期。\n", nwrite, errno);
    }
    signal(SIGPIPE, SIG_DFL); // 恢复默认 SIGPIPE 处理

    close(conn_fd);
}

/**
 * @brief 场景四：Server 完成 accept，Client 关闭后，Server 在 new_fd (已加入 epoll) 上读写会产生什么
 */
void run_scenario_four(int epoll_fd, int listen_fd) {
    printf("\n--- 场景四: accept 后，已加 epoll，Client 关闭 (预期: new_fd 收到 EPOLLRDHUP/EPOLLIN) ---\n");

    int pipefd[2];
    if (pipe(pipefd) == -1) { perror("pipe"); return; }

    // 1. 模拟 Client: 连接成功，保持打开
    pid_t pid = fork();
    if (pid == 0) {
        close(pipefd[1]); // 关闭写端
        client_connect_and_hold("S4", pipefd[0]);
    }
    close(pipefd[0]); // Server 持有写端

    usleep(100000); // 确保 Client 连接成功

    // 2. Server: 接受连接
    struct epoll_event listen_events[MAX_EVENTS];
    epoll_wait(epoll_fd, listen_events, MAX_EVENTS, -1); // 确保 listen fd 有事件

    int conn_fd = accept(listen_fd, NULL, NULL);
    if (conn_fd == -1) {
        perror("[S4] Server accept failed");
        close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }
    set_nonblocking(conn_fd);
    printf("[S4] Server: 成功 accept 新连接 fd %d。\n", conn_fd);

    // 3. Server: 注册 new_fd 到 epoll
    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLRDHUP | EPOLLET;
    ev.data.fd = conn_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev) == -1) {
        perror("[S4] epoll_ctl: conn_fd");
        close(conn_fd); close(pipefd[1]); kill(pid, SIGTERM); waitpid(pid, NULL, 0);
        return;
    }

    // 4. Client: 关闭连接
    write(pipefd[1], "C", 1); // 发出关闭指令
    close(pipefd[1]);
    waitpid(pid, NULL, 0);
    printf("[S4] Client: 已关闭连接。\n");

    // 5. Server: 等待 new_fd 上的事件
    struct epoll_event conn_events[MAX_EVENTS];
    printf("[S4] Server: 等待 new_fd 上的关闭事件...\n");
    int nfds = epoll_wait(epoll_fd, conn_events, MAX_EVENTS, TIMEOUT_MS * 2);

    if (nfds > 0 && conn_events[0].data.fd == conn_fd) {
        unsigned int events_mask = conn_events[0].events;
        printf("[S4] Server 成功: 收到 new_fd 事件掩码 0x%X。\n", events_mask);

        if (events_mask & EPOLLRDHUP) {
            printf("[S4] Server 成功: 捕获到 EPOLLRDHUP (对端关闭写入端) 事件。\n");
        }
        if (events_mask & EPOLLIN) {
            // 尝试 read 验证 EOF
            char buffer[10];
            ssize_t nread = read(conn_fd, buffer, 10);
            if (nread == 0) {
                printf("[S4] Server 成功: read() 返回 0 (EOF)，连接已正常关闭。\n");
            } else {
                printf("[S4] Server 失败: read() 返回 %zd, 连接未正确关闭。\n", nread);
            }
        }
    } else {
        printf("[S4] Server 失败: 未收到 new_fd 上的关闭事件。\n");
    }

    // 清理 epoll 和 fd
    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, conn_fd, NULL);
    close(conn_fd);
}


// --- 主函数 ---
int main() {
    int listen_fd, epoll_fd;
    struct sockaddr_un addr;

    // 0. 清理旧 socket 文件
    unlink(SOCKET_PATH);

    // 1. 创建和配置 Server Listen Socket
    if ((listen_fd = socket(AF_UNIX, SOCK_STREAM, 0)) == -1) { perror("socket"); exit(EXIT_FAILURE); }
    set_nonblocking(listen_fd);

    memset(&addr, 0, sizeof(addr));
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, SOCKET_PATH, sizeof(addr.sun_path) - 1);

    if (bind(listen_fd, (struct sockaddr*)&addr, sizeof(addr)) == -1) { perror("bind"); goto cleanup; }
    if (listen(listen_fd, LISTEN_BACKLOG) == -1) { perror("listen"); goto cleanup; }
    printf("Server: UDS 监听中于 %s\n", SOCKET_PATH);

    // 2. 创建 epoll 实例并添加 listen_fd
    epoll_fd = epoll_create1(0);
    if (epoll_fd == -1) { perror("epoll_create1"); goto cleanup; }

    struct epoll_event ev;
    ev.events = EPOLLIN | EPOLLET;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) { perror("epoll_ctl: listen_fd"); goto cleanup_epoll; }

    // 运行测试场景
    run_scenario_one(epoll_fd, listen_fd);
    run_scenario_two(epoll_fd, listen_fd);

    // 移除 listen_fd 上的事件，防止影响场景三和四的 accept 调用
    epoll_ctl(epoll_fd, EPOLL_CTL_DEL, listen_fd, NULL);

    // 场景三和四需要清空队列才能确保 accept 拿到新的连接
    int temp_fd;
    while((temp_fd = accept(listen_fd, NULL, NULL)) != -1) { close(temp_fd); }

    // 重新添加 listen_fd for S4 (必须在 accept 之前添加，因为它在 S4 场景中也要用于 epoll_wait)
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) == -1) { perror("epoll_ctl re-add"); goto cleanup_epoll; }

    run_scenario_three(listen_fd); // S3 不使用 epoll，所以 listen_fd 状态不重要

    // S4 需要 epoll_wait(listen_fd) 再次触发
    run_scenario_four(epoll_fd, listen_fd);

    // 清理
cleanup_epoll:
    close(epoll_fd);
cleanup:
    close(listen_fd);
    unlink(SOCKET_PATH);
    printf("\n--- 所有测试完成，资源已清理。---\n");

    return 0;
}
```


```bash
# ./epoll_unix_robust_test
Server: UDS 监听中于 /tmp/epoll_uds_robust_test.sock

--- 场景一: epoll_wait 前关闭 (预期: epoll_wait 无事件) ---
[S1] Client: 连接建立成功。
[S1] Client: 连接立即关闭。
[S1] Server: Client已关闭。调用 epoll_wait 验证是否有事件。
[S1] Server 失败: 接收到 1 个事件 (但预期是 0)。尝试 accept 验证...
[S1] Server: 尝试调用 accept()...
[S1] Server 失败: accept 成功，返回新的 fd 5。

--- 场景二: epoll_wait 后，accept 前关闭 (预期: epoll_wait 有事件，accept 失败) ---
[S2] Client: 连接建立成功，保持打开。
[S2] Server: 等待连接事件...
[S2] Server: 收到 EPOLLIN 事件 (1)。
[S2] Client: 收到关闭指令，正在关闭连接。
[S2] Client: 已关闭连接。
[S2] Server: 尝试调用 accept()...
[S2] Server 失败: accept 成功，返回新的 fd 5。

--- 场景三: accept 后，未加 epoll，Client 关闭 (预期: read=0, write=EPIPE) ---
[S3] Client: 连接建立成功，保持打开。
[S3] Server: 成功 accept 新连接 fd 5。
[S3] Client: 收到关闭指令，正在关闭连接。
[S3] Client: 已关闭连接。
[S3] Server 成功: read() 返回 0 (EOF)，符合预期。
[S3] Server 成功: write() 失败，errno=32 (EPIPE/Broken pipe)，符合预期。

--- 场景四: accept 后，已加 epoll，Client 关闭 (预期: new_fd 收到 EPOLLRDHUP/EPOLLIN) ---
[S4] Client: 连接建立成功，保持打开。
[S4] Server: 成功 accept 新连接 fd 5。
[S4] Client: 收到关闭指令，正在关闭连接。
[S4] Client: 已关闭连接。
[S4] Server: 等待 new_fd 上的关闭事件...
[S4] Server 成功: 收到 new_fd 事件掩码 0x2011。
[S4] Server 成功: 捕获到 EPOLLRDHUP (对端关闭写入端) 事件。
[S4] Server 成功: read() 返回 0 (EOF)，连接已正常关闭。

--- 所有测试完成，资源已清理。---
```

#### 对已经close的fd继续操作
**read**：返回-1, `errno = 9, Bad file descriptor` ;
**close**：同上；
**write**：同上；

#### 如何判断对端关闭
**优先使用上面介绍的`EPOLLRDHUP`**;
**使用`EPOLLIN`， 然后调用`read`, 此时返回的ssize_t类型结果为0**；

注：对端关闭包括：对端调用`close`，`ctrl + c`, `kill`, `kill -9`。

#### 对端正常 close时本端行为
**（1）对端close时，如果对端的接收缓冲区内已无数据，则走tcp四次挥手流程，发送`FIN` 包**；
此时本端会触发事件如下：
EPOLLRDHUP （需要主动在`epoll_ctl`时加入该`events「EPOLLRDHUP」`） +  EPOLLIN  + EPOLLOUT
1》此时应优先处理`EPOLLRDHUP`,它明确表明对端已经关闭，处理时close相应fd后，无需再继续处理其他事件；
2》如果不处理`EPOLLRDHUP`的话，也可以处理`EPOLLIN`事件，此时`read`返回0, 同样表明对端已经关闭；
3》如果以上两个事件都没有处理，而是在`EPOLLOUT`事件里又向fd写了数据，数据只是写入到本地tcp发送缓冲区，此时`write`调用会返回成功，但是紧接着`epoll_wait`又会返回如下事件组合：
> EPOLLERR + EPOLLHUP + EPOLLIN + EPOLLOUT + EPOLLRDHUP （需要主动在`epoll_ctl`时加入该`events「EPOLLRDHUP」`）;
> 可以看到相比之前多了`EPOLLERR`和`EPOLLHUP`，是因为之前收到了对端close时发送的`FIN` 包，此时再给对端发送数据，对端会返回`RST`包。
> 如果在收到`RST`包后，又向对端发送数据，会收到`sigpipe`异常，其默认处理是终止当前进程，此时可通过忽略此异常解决，忽略后`write`会返回-1, `erron =32, Broken pipe`:
> 注：`signal(SIGPIPE, SIG_IGN)`; `Broker pipie`这个异常，说到底是应用层没有对相应的`fd`在收到对端关闭通知时，作正确的处理所致，它并不是`tcp/ip`通讯层面的问题。

**(2)对端close（kill, kill -9）时，如果接收缓冲区内还有数据，不会发送`FIN`包，而是发送`RST`**；
此时本端：
1> 收到`RST`后的第一次写操作，写失败，`errno = 104,  Connection reset by peer`; 之后将触发下列事件：

   ```
   EPOLLIN
   EPOLLOUT
   EPOLLHUP
   EPOLLRDHUP（需要主动在epoll_ctrl时加入该events）
   ```
2> 收到`RST`后的第二次及后序的写操作，在忽略了`SIGPIPE`信号后，写失败(返回`-1`)，`erron =32, Broken pipe`;

3> 收到`RST`后的读操作：`errno = 104,  Connection reset by peer`


#### close的行为

##### 调用close时，如果接收缓冲区还有数据未read到应用层
则不会走四次挥手流程，直接发`RST`包。

##### 调用close时，如果发送缓冲区还有数据未发送
则close立即返回，系统接管这个socket, 将尽力将发送缓冲区数据发送到对端，然后发送`FIN`包，走四次挥手流程。

##### `SO_LINGER`改变`close`默认行为

#### close之前是否需要将fd从epoll中删除

当关闭（`close`）某个`fd`套接字时，如果该`fd`对应的打开文件描述（打开文件句柄）「open file description/handle， 是内核对打开文件的内部表示」在本进程内不存在其他的`fd`指向这个打开文件描述（打开文件句柄），内核会自动将套接字描述符从 `epoll` 中删除，因此本端不会再触发任何事件。如果还有其他的`fd`指向这个打开文件描述（打开文件句柄），那么内核中打开文件描述（打开文件句柄），那么调用`close`了这个`fd`，也是相当于这个打开文件描述（打开文件句柄）的引用计数--，`epoll`还是可能会继续上报这个打开文件描述（打开文件句柄）的事件的。

参见：`man epoll`.

![](attachments/Pasted%20image%2020250512114757.png)

因此，为了防止异常情况发生， 还是==建议 在 `close` 之前，先通过`epoll_ctl`将 `fd`从 `epollfd` 中删除==。


#### 常见流程的事件触发结果
**(1) listen fd**：
1> `listen fd`，设置的监听事件：`EPOLLIN` 或者`EPOLLET | EPOLLIN`；
由于此`socket`只监听有无连接，谈不上写和其他操作。  故只有这两类。（默认是`LT`模式，即`EPOLLLT |EPOLLIN`）。 

2> 有新连接请求时候，`listen fd`触发`EPOLLIN`。  然后`accept`产生一个`accept_fd`。

3> 说明：
`listen fd`只需要`EpollIn`就足够了，`EpollErr`和`EpollHup`会自动加上。
`listen fd` 建议是水平触发，而非边缘触发，防止漏事件。
`listen fd`上也设置`EPOLLOUT`等，也不会出错，只是这个`socket`不会收到这样的消息。 

**(2) 连接正常关闭**
`client` 端`close()`连接
`server` 会报某个`accept` 后的 `sockfd`可读，即`epollin`来临。
然后进行循环`recv/read` ， 如果返回0，再调用`epoll_ctl` 中的`EPOLL_CTL_DEL` , 同时`close(sockfd)`。
有些系统会收到一个`EPOLLRDHUP`，当然检测这个是最好不过了。只可惜是有些系统。
注：`read/recv` 返回`0` + `EPOLLRDHUP`，两个都进行考虑那就是万能的了。

**(3) 连接异常关闭**
`EPOLLERR`这种错误一般而言有动作才能检测出来。`Epoll`中向已经断开的`socket`写或者读，会发生`EPOLLERR` ，即表明已经断开。
服务器再给一个已经关闭的`socket`写数据时，会出错，这时候，服务器才明白对方可能已经异常断开了（读也可以）。

**(4)keepalive保活检查**
当客户端的机器在发送请求前，就崩溃了（或者网络断掉了），此时`FIN/RST`无法发送给对端，则服务器一端是无从知晓的。
按照现在的这个请求响应方式，无论是否使用`epoll`，都必须要做超时检查。

因此，这个问题与`epoll`无关。`EPOLLERR`这种错误一般是有动作才能检测出来。服务器不可能经常的向客户端写一个东西，就无法依照有没有`EPOLLERR`来判断客户端是不是死了。
因此，服务器中的超时检查是很重要的。这也是以前服务器中进行`keepalive`检查的原因。

#### 当多个事件（如 `EPOLLERR`、`EPOLLHUP`、`EPOLLRDHUP`、`EPOLLIN`、`EPOLLOUT`）同时触发时，处理的顺序?

`Deepseek`的回答，如下所示：

![](attachments/Pasted%20image%2020250520104748.png)

##### 注意
**（1）边缘触发模式（EPOLLET）**：
在边缘触发模式下，必须彻底处理所有数据，否则可能丢失事件（如未读取完数据可能导致 `EPOLLIN` 不再触发）。

- **（2）`EPOLLERR`（错误）**: 
表示文件描述符发生错误（如连接被reset重置）。若不优先处理，后续读写操作可能无效或导致程序崩溃。



# 线程安全
## epoll_ctr 对红黑树加锁

```text
只允许一个线程 在rbtree.insert()/delete();的时候
```

## epoll_wait对就绪队列加锁

```text
用自旋锁；对于队列添加， 避免SMP体系下 多核竞争，采用自旋锁 ，能够快速的操作list
```

# epoll的应用范例
## nginx中对于epoll的使用

![](attachments/Pasted%20image%2020250531210918.png)

如上， `nginx` 的 `epoll_ctl` 系统调用，除了 `listen socket` 的操作是 `lt` 模式，其它的 `socket` 处理几乎所有都是 `et` 模式。

### `listen-sockfd`的`accept`的惊群问题的解决
#### 方案一：EPOLLEXCLUSIVE 
`EPOLLEXCLUSIVE` 是 2016 年 4.5+ 内核新添加的一个 epoll 的标识（代码改动较小，详看：[github](https://link.zhihu.com/?target=https%3A//github.com/torvalds/linux/commit/df0108c5da561c66c333bb46bfe3c1fc65905898)）。

它降低了多个进程/线程通过 epoll_ctl 添加共享 fd 引发的惊群概率，使得一个事件发生时，只唤醒一个正在 epoll_wait 阻塞等待唤醒的进程/线程（而不是全部唤醒）。

而 Ngnix 在 1.11.3 之后相应添加了 `NGX_EXCLUSIVE_EVENT` 功能标识（代码改动较小，详看：[github](https://link.zhihu.com/?target=https%3A//github.com/nginx/nginx/commit/5c2dd3913aad5c4bf7d9056e1336025c2703586b)），它使用了 EPOLLEXCLUSIVE 特性。

对比 nginx 在应用层的解决方案：[accept_mutex](https://link.zhihu.com/?target=https%3A//wenfh2020.com/2021/10/10/nginx-thundering-herd-accept-mutex/)，`NGX_EXCLUSIVE_EVENT` 它从内核层面避免惊群问题，它更简洁高效。该功能的工作原和使用相对简单：进程使用 epoll_ctl 添加 `listen socket fd` 时，把 `EPOLLEXCLUSIVE` 属性添加进去就可以了。多个进程通过 `epoll_wait` 等待 `listen socket` 事件，当有新链接到来时，内核只唤醒一个等待的进程。

（1）`nginx` 的 `EPOLLEXCLUSIVE` 工作模型，如下所示。
![](attachments/Pasted%20image%2020250531213513.png)

（2）`socket` 唤醒流程：
![](attachments/Pasted%20image%2020250531213708.png)


#### 方案二：reuseport
`SO_REUSEPORT (reuseport)` 是网络的一个选项设置，它能开启内核功能：网络链接分配内核负载均衡。
该功能允许多个进程/线程 `bind/listen` 相同的 `IP/PORT`，提升了新链接的分配性能。

`reuseport` 也是内核解决 `惊群问题` 的优秀方案。
(1)  每个进程可以 `bind/listen` 相同的 `IP/PORT`，相当于==每个进程/线程拥有独立的 `listen socket` 的完全队列，避免了共享 `listen socket` 的资源争抢==，提升了并发的吞吐。
(2) 内核通过哈希算法，将新链接相对均衡地分配到各个开启了 `reuseport` 属性的进程，所以资源的负载均衡得到解决。

# epoll 与 select 和 poll 

linux上最常见的几种io复用模型有：select，poll和epoll。  


## select/poll
select 实现多路复用的方式是，将已连接的 Socket 都放到一个文件描述符集合，然后调用 select 函数将文件描述符集合拷贝到内核里，让内核来检查是否有网络事件产生，检查的方式很粗暴，就是通过遍历文件描述符集合的方式，当检查到有事件产生后，将此 Socket 标记为可读或可写， 接着再把整个文件描述符集合拷贝回用户态里，然后用户态还需要再通过遍历的方法找到可读或可写的 Socket，然后再对其处理。

所以，对于 select 这种方式，需要进行 2 次「遍历」文件描述符集合，一次是在内核态里，一个次是在用户态里 ，而且还会发生 2 次「拷贝」文件描述符集合，先从用户空间传入内核空间，由内核修改后，再传出到用户空间中。


select 使用固定长度的 BitsMap，表示文件描述符集合，而且所支持的文件描述符的个数是有限制的，在 Linux 系统中，由内核中的 FD_SETSIZE 限制， 默认最大值为 1024，只能监听 0~1023 的文件描述符。

poll 不再用 BitsMap 来存储所关注的文件描述符，取而代之用动态数组，以链表形式来组织，突破了 select 的文件描述符个数限制，当然还会受到系统文件描述符限制。

但是 poll 和 select 并没有太大的本质区别，都是使用「线性结构」存储进程关注的 Socket 集合，因此都需要遍历文件描述符集合来找到可读或可写的 Socket，时间复杂度为 O(n)，而且也需要在用户态与内核态之间拷贝文件描述符集合，这种方式随着并发数上来，性能的损耗会呈指数级增长。

### select 的缺点总结
缺点1：select能够监视的文件描述符数量存在最大限值，通常是1024。  
缺点2：采用轮询方式扫描文件描述符，文件描述符越多，性能越差。  
缺点3：每次select都需要将句柄数据结构从用户空间复制到内核空间，开销很大。  
缺点4：select采用数组记录监视的文件描述符，每次返回之后需要遍历查找触发的事件。  
缺点5：select采用水平触发模式。

### poll 的缺点总结
poll采用链表结构保存文件描述符，因此没有了最大监视限制，解决了缺点1。但其他缺点依然存在。

## epoll 与 select 和 poll 对比
select 和 poll 是 Linux 下的两种 I/O 多路复用机制，相比于 epoll，它们在用户态和内核态的切换上更加耗时，同时还存在文件描述符数量限制的问题。

![](attachments/Pasted%20image%2020250507200941.png)

epoll 接口属于Linux下多路I/O复用接口中select/poll的增强。其经常应用于Linux下高并发服务型程序，特别是==在大量并发连接中只有少部分连接处于活跃下的情况== (通常是这种情况)，在该情况下能显著的提高程序的CPU利用率。

设想一个场景：有100万个用户同时与一个进程保持着TCP连接，而每一时刻只有几十个或几百个TCP连接是活跃的(接收TCP包)，也就是说在每一时刻进程只需要处理这100万连接中的一小部分连接。那么，如何才能高效的处理这种场景呢？进程是否在每次询问操作系统收集有事件发生的TCP连接时，把这100万个连接告诉操作系统，然后由操作系统找出其中有事件发生的几百个连接呢？实际上，在 Linux2.4 版本以前，那时的select 或者 poll 事件驱动方式是这样做的。

这里有个非常明显的问题，即在某一时刻，进程收集有事件的连接时，其实这100万连接中的大部分都是没有事件发生的。因此如果每次收集事件时，都把100万连接的套接字传给操作系统（这首先是用户态内存到内核态内存的大量复制），而由操作系统内核寻找这些连接上有没有未处理的事件，将会是巨大的资源浪费，然后select和poll就是这样做的，因此它们最多只能处理几千个并发连接。而epoll不这样做，它在Linux内核中申请了一个简易的文件系统，把原先的一个select或poll调用分成了3部分：

```c
int epoll_create(int size); 
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event); 
int epoll_wait(int epfd, struct epoll_event *events,int maxevents, int timeout);

说明：
1、调用 epoll_create 建立一个 epoll 对象（在epoll文件系统中给这个句柄分配资源）； 
2、调用 epoll_ctl 向 epoll 对象中添加这100万个连接的套接字；
3、调用 epoll_wait 收集发生事件的连接。
```

这样只需要在进程启动时建立 1 个 epoll 对象，并在需要的时候向它添加或删除连接就可以了，因此，在实际收集事件时，epoll_wait 的效率就会非常高，因为调用 epoll_wait 时并没有向它传递这100万个连接，内核也不需要去遍历全部的连接。

![](attachments/Pasted%20image%2020250511222417.png)

### 文件描述符数量
- select通过线性表描述文件描述符集合，文件描述符有上限，一般是1024，但可以修改源码，重新编译内核，不推荐

- poll是链表描述，突破了文件描述符上限，最大可以打开文件的数目

- epoll通过红黑树描述，最大可以打开文件的数目，可以通过命令`ulimit -n number`修改，仅对当前终端有效

### 将 fd 传入内核的方式

- select：从用户态创建拷贝到内核态，每次调用都需要拷贝

- poll：从用户态拷贝到内核态，每次调用都需要拷贝

- epoll：通过 `epoll_create` 直接在内核态创建一棵红黑树，通过 `epoll_ctl` 将要监听的文件描述符注册到红黑树上。

即：epoll 在内核里使用红黑树来跟踪进程所有待检测的文件描述字，把需要监控的 socket 通过 `epoll_ctl()` 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 O(logn)。而 select/poll 内核里没有类似 epoll 红黑树这种保存所有待检测的 socket 的数据结构，所以 select/poll 每次操作时都传入整个 socket 集合给内核，而 epoll 因为在内核维护了红黑树，可以保存所有待检测的 socket ，所以只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。

### 内核态检测 fd 就绪状态的方式
- select：轮询机制遍历所有的 fd，判断哪个文件描述符上有事件发生

- poll：轮询机制遍历所有的 fd，判断哪个文件描述符上有事件发生

- epoll：回调机制，调用 epoll_ctl 时会在内核态注册回调函数，内核除了帮我们在epoll文件系统里建了个红黑树用于存储以后 epoll_ctl 传来的 fd 外，还会再建立一个list的 ready链表，用于存储准备就绪的事件，当 epoll_wait 调用时，仅仅观察这个 list 链表里有没有数据即可。epoll 是根据每个 fd 上面的回调函数(中断函数)判断，只有发生了事件的 socket 才会主动的去调用 callback 函数，其他空闲状态 socket 则不会，若是就绪事件，插入 list。

即：epoll 使用事件驱动的机制，内核里维护了一个链表来记录就绪事件，当某个 socket 有事件发生时，通过回调函数内核会将其加入到这个就绪事件列表中，当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符的个数，不需要像 select/poll 那样轮询扫描整个 socket 集合，大大提高了检测的效率。


### 应用程序索引就绪文件描述符

- select/poll 只返回发生了事件的文件描述符的个数，若知道是哪个发生了事件，同样需要遍历。

- epoll 返回的发生了事件的个数和结构体数组，结构体包含 socket 的信息，因此直接处理返回的数组即可。

### 工作模式
- select 和 poll 都只能工作在相对低效的 LT 模式下

- epoll 则可以工作在 ET 高效模式，并且 epoll 还支持 EPOLLONESHOT 事件，该事件能进一步减少可读、可写和异常事件被触发的次数。

## epoll 更高效的原因

- 对于select和poll来说，所有文件描述符都是在用户态被加入其文件描述符集合的，每次调用都需要将整个集合拷贝到内核态；epoll 则将整个文件描述符集合维护在内核态，每次添加文件描述符的时候都需要执行一个系统调用。系统调用的开销是很大的，而且在有很多短期活跃连接的情况下，由于这些大量的系统调用开销，epoll 可能会慢于 select 和 poll。

- select 和 poll 的动作基本一致，只是 poll 采用链表的方式来存储 fd，而 select 采用 fd 标注位来存放，所以 select 会受到最大连接数的限制而 poll 不会。epoll 底层通过红黑树来描述，并且维护一个 ready list，将事件表中已经就绪的事件添加到这里，在使用 epoll_wait 调用时，仅观察这个 list 中有没有数据即可。

- select、poll、epoll 虽然都会返回就绪的 fd 数量，但是 select 和 poll 并不会明确指出是哪些 fd 就绪，而 epoll 会。这造成的区别就是：系统调用返回后，调用 select 和 poll 的程序需要遍历监听的整个文件描述符找到是哪些处于就绪状态，产生了大量的开销，而 epoll 则不需要去以这种方式检查，当有活动产生时，会自动触发epoll回调函数通知epoll文件描述符，然后内核将这些就绪的文件描述符放到之前提到的ready list中等待epoll_wait调用后被处理。随着 fd 数量的增加，select 和 poll 的效率会越来越低，而 epoll 则不会受到太大影响。

- select 和 poll 都只能工作在相对低效的LT模式下，而 epoll 同时支持 LT 和 ET 模式。epoll 的 ET 模式效率高，系统不会充斥大量不关心的就绪 fd。


## 具体场景的选择

- 当监测的 fd 数目较小，且各个 fd 都比较活跃，建议使用 select 或者 poll。
即 fd 数量有限，且所有的fd都是活跃连接，使用epoll，需要建立文件系统，红黑树和链表对于此来说，效率反而不高，不如 selece 和 poll。

- 当监测的 fd 数目非常大，成千上万，且单位时间只有其中的一部分 fd 处于就绪状态，这个时候使用 epoll 能够明显提升性能。

 使用 epoll 的方式，即使监听的 Socket 数量越多的时候，效率也不会大幅度降低，能够同时监听的 Socket 的数目也非常的多了，上限就为系统定义的进程打开的最大文件描述符个数。因而，**epoll 被称为解决 C10K 问题的利器**。


# QA

![](attachments/Pasted%20image%2020250525221154.png)

![](attachments/Pasted%20image%2020250525223343.png)


## 文件描述符（file descriptor）和 打开文件描述（open file description）
Foom 在 [LWN](https://lwn.net/Articles/430804/) 上说道：
```text
显然 epoll 存在巨大的设计缺陷，任何懂得 file descriptor 的人应该都能看得出来。事实上当你回望 epoll 的历史，你会发现当时实现 epoll 的人们显然并不怎么了解 file descriptor 和 file description 的区别。
```

实际上，epoll() 的这个问题主要在于它混淆了==用户态的 file descriptor== (我们平常说的数字 fd) 和==内核态中真正用于实现的 file description==。

`epoll_ctl(EPOLL_CTL_ADD)` 实际上并不是注册一个 `file descriptor (fd)`，而是将 `fd` 和 一个指向内核` file description` 的指针的对 (tuple) 一块注册给了 `epoll`，导致问题的根源在于，==`epoll` 里管理的 fd 的生命周期，并不是`fd` 本身的，而是内核中相应的 `file description` 的==。


当使用 `close()` 这个系统调用关掉一个 `fd` 时，如果这个 `fd` 是内核中 `file description` 的唯一引用时，内核中的 `file description` 也会跟着一并被删除，这样是 OK 的；但是当内核中的 `file description` 还有其他引用时，`close` 并不会删除这个 `file descrption`。这样会导致当这个 `fd` 还没有从 `epoll` 中挪出就被直接 `close` 时，`epoll()` 还会在这个已经 `close()` 掉了的 fd 上上报事件。

### 范例
```c
rfd, wfd = pipe()
write(wfd, "a")             # Make the "rfd" readable

epfd = epoll_create()
epoll_ctl(efpd, EPOLL_CTL_ADD, rfd, (EPOLLIN, rfd))

rfd2 = dup(rfd)
close(rfd)

r = epoll_wait(epfd, -1ms)  # What will happen?
```
由于 `close(rfd)` 关掉了这个 `rfd`，你可能会认为这个 `epoll_wait()` 会一直阻塞不返回，而实际上并不是这样。由于调用了 `dup()`，内核中相应的 `file description` 仍然还有一个引用计数而没有被删除，所以这个 `file descption` 的事件仍然会上报给 `epoll`。

因此 `epoll_wait()` 会给一个已经不存在的 fd 上报事件。 更糟糕的是，一旦你 `close()` 了这个 fd，再也没有机会把这个死掉的 `fd` 从 `epoll` 上摘除了，下面的做法都不行：
```c
epoll_ctl(efpd, EPOLL_CTL_DEL, rfd)
epoll_ctl(efpd, EPOLL_CTL_DEL, rfd2)
```

因此，存在 `close` 掉了一个 `fd`，却还一直从这个 `fd` 上收到 `epoll` 事件的可能性。并且这种情况一旦发生，不管你做什么都无法恢复了。

### 小结
==永远记着先在调用 `close()` 之前，显示的调用 `epoll_ctl(EPOLL_CTL_DEL)`==

# 参考
```bash
# 揭开epoll面纱：Nginx，Redis等都在用的多路复用，到底是什么？
https://mp.weixin.qq.com/s/yvH5SaweAUnym6bQS3Y8Sg

# I/O 多路复用：select/poll/epoll 【小林coding】
https://www.xiaolincoding.com/os/8_network_system/selete_poll_epoll.html#%E6%80%BB%E7%BB%93

# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （1）
https://zhuanlan.zhihu.com/p/63179839

# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！（2）
https://zhuanlan.zhihu.com/p/64138532

# 如果这篇文章说不清epoll的本质，那就过来掐死我吧！ （3）
https://zhuanlan.zhihu.com/p/64746509

## Linux epoll 机制
https://www.expoli.tech/articles/2023/03/25/what-is-linux-epoll


# Socket编程中的几点问题总结
https://cloud.tencent.com/developer/article/1649165


# EPOLLHUP/EPOLLRDHUP/read返回0的区别
http://62.234.193.50:8080/categories/socket-network-programming
http://62.234.193.50:8080/archives/EPOLLHUP-EPOLLRDHUP


# 腾讯二面：epoll性能那么高，为什么？
https://mp.weixin.qq.com/s/ByxSt2CMNkg1knrN6lcCvw


```