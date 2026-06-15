```table-of-contents
```
# 介绍
bthread 是 brpc(百度开源的 RPC 框架)自研的一套**用户态协程(coroutine)库**,本质上是一个 **M:N 调度模型**:M 个 bthread(用户态轻量级"线程")运行在 N 个 pthread(操作系统线程,brpc 里称为 worker)之上。

# bthread 定义

bthread, lightweight user-level thread，轻量级的用户态线程，看起来像 pthread， 但实际上：不是内核线程

```c
void* worker(void*) {
    ...
}

bthread_t tid;
bthread_start_background(
    &tid,
    NULL,
    worker,
    NULL);
```

# bthread 和 pthread
## bthread 为什么快
pthread切换 需要：用户态 -> 内核态 -> 调度 -> 返回；通常需要 1~5 us。
而 bthread：`bthread A  -> bthread B`, **用户态调度器**，仅仅 保存寄存器，切换栈，恢复寄存器，全部用户态完成，通常需要几十ns ~ 几百ns。

## 对比

|维度|pthread(OS 线程)|bthread|
|---|---|---|
|调度者|操作系统内核|bthread 自己的**用户态调度器**|
|切换开销|微秒级(涉及内核态切换、TLB flush 等)|纳秒级(纯用户态寄存器切换)|
|创建开销|较大(内核对象、独立栈通常 MB 级)|很小(栈可以做得很小,默认通常是 1MB 但可配置更小,创建本身不进内核)|
|数量级|通常几十到几百|可以是几十万|
|阻塞行为|阻塞会占用一个内核线程|bthread 阻塞(如等锁、等 IO)时会被挂起,worker 转去跑别的 bthread,不浪费 OS 线程|

## M:N 调度模型

brpc 启动时会创建固定数量的 worker pthread(通常等于 CPU 核数,可配置),所有的 bthread 在这些 worker 之上被调度,调度方式是**协作式 + 工作窃取(work-stealing)**:

- 每个 worker 维护一个本地的 bthread 运行队列(run queue)
- 当某个 bthread 主动让出(yield)、被阻塞(比如等待一个 mutex、等待 RPC 返回、调用 bthread_usleep)时,worker 立刻切换去跑队列里的下一个 bthread
- 当某个 worker 本地队列空了,会去其他 worker 的队列里"偷"一个 bthread 来跑,这样可以让负载在多核间自动均衡,这是 bthread 在多核场景下吞吐能优于很多简单协程库的关键设计

这一点和 Go 的调度器(GMP 模型)思路非常相似:G(goroutine)对应 bthread,M(OS thread)对应 worker,P(processor,本地队列+上下文)对应 worker 自带的本地队列,work-stealing 的思想也是直接对应的。

# bthread 和协程 coroutine

协程这个词在不同场景下指代差异很大:

- **有栈协程(stackful coroutine)**:每个协程有独立的栈空间,可以在任意函数调用深度处挂起/恢复,挂起时通过保存/恢复寄存器(包括栈指针)实现上下文切换。bthread 属于这一类,本质和 ucontext、libco、goroutine 是同一个范式。
- **无栈协程(stackless coroutine)**:如 C++20 的 coroutine(co_await/co_return),通过编译器把状态机展开,没有独立的调用栈,挂起点必须是显式的语法标记处。

bthread 属于**有栈协程**,实现上类似 Go 的 goroutine,核心是:
```bash
每个 bthread = 一段独立分配的栈 + 一个上下文(寄存器状态) 
切换 bthread = 保存当前寄存器到 TCB,恢复目标 bthread 的寄存器(swapcontext 风格的实现,但 bthread 是手写的汇编/ucontext 变种以追求更低开销)
```

# bthread 解决的核心问题:同步写法,异步效果

bthread 存在的意义,本质上是让 brpc 的使用者可以用**看起来同步阻塞的代码风格**写出**异步非阻塞的执行效果**。

举个例子,在 brpc 里发起一个同步 RPC 调用:
```c
brpc::Controller cntl;
stub.SomeMethod(&cntl, &request, &response, NULL);
// 这一行看起来是阻塞调用,但实际上:
// 1. RPC 请求被异步发出
// 2. 当前 bthread 被挂起(yield 给调度器),不占用 worker
// 3. worker 立刻去跑其他 bthread
// 4. RPC 响应到达后,这个 bthread 被重新唤醒、放回运行队列
// 5. 调度器轮到它时恢复执行,这一行才"返回"
```

这就是协程相比裸用 epoll+回调(传统 Reactor 写法)最大的优势所在:**避免了回调地狱(callback hell)**,业务逻辑可以用线性的、同步风格的代码表达,底层却是完全异步、不阻塞 OS 线程的。

bthread 也提供了 mutex、condition_variable、fiber-local storage 等一整套和 pthread 接口几乎对齐的原语(bthread_mutex_t、bthread_cond_t 等),这些原语在 bthread 阻塞时不会阻塞底层的 pthread,而是会触发协程切换。

## 回调地狱(callback hell)
回调地狱指的是:当一段逻辑需要依次执行多个**异步操作**,而每个异步操作的结果都依赖于前一个操作完成时,代码会以**层层嵌套的回调函数**形式堆叠起来,导致代码:

- 缩进越来越深,像金字塔或者倒三角形(俗称 "pyramid of doom")
- 逻辑上是顺序执行的几个步骤,但代码结构上完全看不出"顺序"感
- 错误处理分散在每一层,容易遗漏
- 难以阅读、难以调试、难以维护

### 范例
#### JS 范例
假设有这样一个业务需求(用伪代码风格,贴近 `Node.js/JS` 的异步回调写法,这是回调地狱最经典的展示场景):

> 先查询用户信息 → 拿到用户的好友列表 → 拿到第一个好友的最近订单 → 把订单详情发送通知;

```javascript
getUser(userId, function(err, user) {
  if (err) {
    handleError(err);
  } else {
    getFriends(user.id, function(err, friends) {
      if (err) {
        handleError(err);
      } else {
        getRecentOrder(friends[0].id, function(err, order) {
          if (err) {
            handleError(err);
          } else {
            sendNotification(order.id, function(err, result) {
              if (err) {
                handleError(err);
              } else {
                console.log("通知发送成功", result);
              }
            });
          }
        });
      }
    });
  }
});
```
- **错误处理重复且分散**:每一层都要写一遍 `if (err) { handleError(err) }`,逻辑高度重复,而且任何一层忘了写,这个错误就会被"吞掉"。
- **难以复用和重构**:这几个回调函数互相嵌套在彼此的闭包里,如果想把中间某一步抽出来单独测试或复用,会很麻烦,因为它依赖外层传进来的变量(闭包捕获)。
- **逻辑顺序和代码结构脱节**:从业务上讲这就是"第一步、第二步、第三步、第四步"的线性顺序,但代码结构上要从左往右、从外往里去理解执行顺序,可读性差。

#### Linux C范例
假设这样一个业务场景(用 Linux 网络编程的典型异步写法):
客户端先连接 Redis 拿到用户的 session token → 再用 token 连接认证服务校验身份 → 校验通过后连接数据库查询用户资料 → 把资料通过 socket 发回给客户端
这四步都是**非阻塞网络 IO**,每一步都要等对应 fd 上的 epoll 事件触发后才能继续下一步。

```c
// 几个连接的上下文,要在堆上分配,靠回调之间传递指针维持状态
typedef struct {
    int client_fd;
    int redis_fd;
    int auth_fd;
    int db_fd;
    char token[64];
    char user_id[32];
} request_ctx_t;

void on_db_query_done(int db_fd, void *arg);
void on_auth_done(int auth_fd, void *arg);
void on_redis_reply(int redis_fd, void *arg);

// 第一步：连接 redis，拿 token，注册到 epoll
void start_request(int client_fd) {
    request_ctx_t *ctx = malloc(sizeof(request_ctx_t));
    ctx->client_fd = client_fd;

    int redis_fd = connect_nonblock("127.0.0.1", 6379);
    ctx->redis_fd = redis_fd;

    // 把 ctx 作为 user_data 注册进 epoll，等可读事件触发时调用 on_redis_reply
    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = ctx;
    epoll_ctl(epfd, EPOLL_CTL_ADD, redis_fd, &ev);

    write(redis_fd, "GET session:1234\r\n", 19); // 简化，未处理写不全的情况
}

// 第二步：redis 返回了 token，再去连认证服务
void on_redis_reply(int redis_fd, void *arg) {
    request_ctx_t *ctx = (request_ctx_t *)arg;
    int n = read(redis_fd, ctx->token, sizeof(ctx->token));
    if (n <= 0) {
        close(ctx->client_fd);
        free(ctx);
        return;
    }
    epoll_ctl(epfd, EPOLL_CTL_DEL, redis_fd, NULL);
    close(redis_fd);

    int auth_fd = connect_nonblock("127.0.0.1", 9000);
    ctx->auth_fd = auth_fd;

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = ctx;
    epoll_ctl(epfd, EPOLL_CTL_ADD, auth_fd, &ev);

    write(auth_fd, ctx->token, strlen(ctx->token)); // 又是一层嵌套的开始
}

// 第三步：认证服务返回结果，再去连数据库
void on_auth_done(int auth_fd, void *arg) {
    request_ctx_t *ctx = (request_ctx_t *)arg;
    char result[8];
    int n = read(auth_fd, result, sizeof(result));
    if (n <= 0 || strncmp(result, "OK", 2) != 0) {
        close(ctx->client_fd);
        close(auth_fd);
        free(ctx);
        return;
    }
    epoll_ctl(epfd, EPOLL_CTL_DEL, auth_fd, NULL);
    close(auth_fd);

    int db_fd = connect_nonblock("127.0.0.1", 5432);
    ctx->db_fd = db_fd;

    struct epoll_event ev;
    ev.events = EPOLLIN;
    ev.data.ptr = ctx;
    epoll_ctl(epfd, EPOLL_CTL_ADD, db_fd, &ev);

    char query[128];
    snprintf(query, sizeof(query), "SELECT * FROM users WHERE id=%s", ctx->user_id);
    write(db_fd, query, strlen(query)); // 第三层嵌套
}

// 第四步：数据库返回数据，写回客户端
void on_db_query_done(int db_fd, void *arg) {
    request_ctx_t *ctx = (request_ctx_t *)arg;
    char profile[1024];
    int n = read(db_fd, profile, sizeof(profile));
    if (n > 0) {
        write(ctx->client_fd, profile, n);
    }
    epoll_ctl(epfd, EPOLL_CTL_DEL, db_fd, NULL);
    close(db_fd);
    close(ctx->client_fd);
    free(ctx); // 注意：这里必须确保所有路径都释放了 ctx，否则内存泄漏
}

// 事件循环：根据 fd 分发到对应回调
void event_loop() {
    struct epoll_event events[64];
    while (1) {
        int n = epoll_wait(epfd, events, 64, -1);
        for (int i = 0; i < n; i++) {
            request_ctx_t *ctx = (request_ctx_t *)events[i].data.ptr;
            int fd = events[i].data.fd; // 简化处理，实际要从 ctx 里判断是哪个 fd
            // 这里需要手动判断这个事件属于 redis_fd/auth_fd/db_fd 中的哪一个，
            // 才能 dispatch 到正确的回调函数 —— 这本身就是状态机的一部分
            if (fd == ctx->redis_fd) on_redis_reply(fd, ctx);
            else if (fd == ctx->auth_fd) on_auth_done(fd, ctx);
            else if (fd == ctx->db_fd) on_db_query_done(fd, ctx);
        }
    }
}
```


如果用同步阻塞写法(单线程,假设不考虑并发)：
```c
void handle_request(int client_fd) {
    char token[64], result[8], profile[1024], user_id[32];

    int redis_fd = connect_blocking("127.0.0.1", 6379);
    write(redis_fd, "GET session:1234\r\n", 19);
    read(redis_fd, token, sizeof(token));
    close(redis_fd);

    int auth_fd = connect_blocking("127.0.0.1", 9000);
    write(auth_fd, token, strlen(token));
    read(auth_fd, result, sizeof(result));
    close(auth_fd);
    if (strncmp(result, "OK", 2) != 0) { close(client_fd); return; }

    int db_fd = connect_blocking("127.0.0.1", 5432);
    char query[128];
    snprintf(query, sizeof(query), "SELECT * FROM users WHERE id=%s", user_id);
    write(db_fd, query, strlen(query));
    int n = read(db_fd, profile, sizeof(profile));
    close(db_fd);

    write(client_fd, profile, n);
    close(client_fd);
}
```

### 为什么会产生回调地狱

本质原因是:在基于事件循环(Reactor 模式)的异步编程模型里,一个异步操作不能"阻塞等待结果",只能"告诉系统:这件事做完后帮我调用这个函数"。当下一步操作需要依赖上一步的结果时,唯一的办法就是把下一步操作写在上一步的回调函数体内部——这样层层累积,就成了嵌套结构。

C 语言里最常见的异步编程模型就是基于 `epoll`(或 `select`/`poll`)的 Reactor 模式:用非阻塞 socket + epoll 监听事件,事件到来后调用预先注册的回调函数。这正是回调地狱最容易滋生的土壤,因为 C 没有 Promise、没有 async/await、没有协程语法糖,只能靠**函数指针 + 手写状态机**来组织异步逻辑。



# brpc 的整体线程/IO 模型

## IO 层:基于 epoll 的 Reactor 模式

brpc 在网络 IO 这一层,**确实是经典的 Reactor 模式**:

- 每个 IO 线程通过 epoll 监听一批 fd 的可读可写事件
- 事件到达后,不是像传统 Reactor 那样直接在事件循环线程里跑回调处理业务逻辑,而是把"处理这个事件"的任务**包装成一个新的 bthread**,丢到调度队列里去异步执行
- 这样原本 Reactor 模式里"回调函数不能阻塞,否则会卡住整个事件循环"的限制被打破了——业务回调跑在 bthread 里,即便里面有阻塞操作(比如又发起了一次同步 RPC、等锁),也只是把这个 bthread 挂起,不影响 epoll 线程继续处理其他事件。

所以可以说:**brpc 的 IO 层是 Reactor,但 Reactor 触发的事件处理被 bthread 化了,Reactor 只负责"派发",真正的执行体是协程**。

## brpc的io模型和传统 Reactor(如 Netty、libevent 裸用)的区别

|维度|传统 Reactor(回调风格)|brpc(Reactor + bthread)|
|---|---|---|
|业务逻辑写法|回调函数,容易陷入回调地狱|同步风格代码,可读性接近多线程同步编程|
|回调中能否阻塞|不能,会卡住整个 event loop|可以,阻塞只会挂起当前 bthread|
|并发单元|事件循环线程本身(通常和 CPU 核数等同)|bthread,数量远超 OS 线程,可以轻松开几万个并发 RPC|
|复杂异步流程(如先查 A 再查 B)|需要手写状态机或者 future/promise 链式调用|直接写顺序代码,等价于"伪同步"|


```bash
	┌─────────────────────────────┐
    │   epoll(Reactor 事件循环)    │  ← IO 多路复用,负责"有事件发生了"
    └───────────────┬───────────────┘
                    │ 事件到达,创建/唤醒 bthread
                    ▼
    ┌─────────────────────────────┐
    │   bthread 调度器(M:N 模型)   │  ← work-stealing,多 worker 间调度
    │  worker1  worker2  worker3...│  (worker = pthread)
    └───────────────┬───────────────┘
                    │ 每个 bthread 跑业务逻辑
                    ▼
    ┌─────────────────────────────┐
    │  业务代码(同步风格,可阻塞)    │  ← 阻塞操作只挂起 bthread,不阻塞 pthread
    └─────────────────────────────┘
```


# 参考 
```bash

```