```table-of-contents
```


# pthread_cond_wait 和 pthread_cond_timedwait

## 注意

### `pthread_cond_wait()` 在等待过程中会自动释放锁


![](attachments/Pasted%20image%2020251110124539.png)

虽然消费者在进入 `pthread_cond_wait()` 前确实持有锁，  但 **`pthread_cond_wait()` 在等待过程中会自动释放锁**，  让生产者能获得锁修改状态并发信号；  当信号到达后，它再自动重新加锁并返回。

`pthread_cond_wait()` 在内部会 **自动释放互斥锁**，并在被唤醒时再自动重新加锁返回。

它的行为不是简单地“等待”，而是以下**四个原子步骤**：
1. **当前线程持有锁**（你必须在加锁后调用它）；
2. `pthread_cond_wait` 会：
    - 自动把当前线程**放入等待队列**；
    - **释放互斥锁（mutex）**；
    - **阻塞进行信号等待**（`pthread_cond_signal()` 或 `pthread_cond_broadcast()`）；

3. 当别的线程（例如生产者）发出信号后，这个等待线程会被唤醒；
4. 被唤醒后，`pthread_cond_wait` 会在返回之前：
    - **重新获得互斥锁**（阻塞直到锁可用）；
    - 然后返回给用户代码（此时又拿到了锁）。

## 完整流程
```bash
消费者线程：
1. pthread_mutex_lock()      --> 拿到锁
2. while(!ready)             --> 条件未满足
3. pthread_cond_wait()       --> 释放锁 + 等待信号（自动完成）
   [此时锁空出来]
   
生产者线程：
4. pthread_mutex_lock()      --> 成功获得锁
5. 修改状态 ready = 1
6. pthread_cond_signal()     --> 发信号（唤醒一个等待者）
7. pthread_mutex_unlock()    --> 释放锁

消费者线程（被唤醒）：
8. 自动重新加锁成功
9. pthread_cond_wait 返回
10. 再次检查条件并执行逻辑
11. pthread_mutex_unlock()   --> 释放锁

```

```bash
时间轴 →
Consumer: [lock]--[wait(cond)]--{释放锁并睡眠}-------------------{被唤醒并重新lock}--[继续执行]
Producer:                     -------------[lock]--[修改状态]--[signal]--[unlock]-----------
```

**整个过程中锁被正确交替使用，不会死锁。**


```c
（1）消费者：
pthread_mutex_lock(&mutex);
while (!resource_ready) {
    pthread_cond_wait(&cond, &mutex);
}
// 到这里表示 resource_ready == 1，且重新持有锁
printf("Consumer: got resource!\n");
pthread_mutex_unlock(&mutex);


（2）生产者：
pthread_mutex_lock(&mutex);
resource_ready = 1;
pthread_cond_signal(&cond);  // 唤醒一个等待的消费者
pthread_mutex_unlock(&mutex);
```


# 虚假唤醒（spurious wakeup）问题


# 唤醒丢失(lost wakeup)问题
如果 一个线程先执行了 `pthread_cond_broadcast/pthread_cond_signal`，而另外一个线程随后才执行到 `pthread_cond_timedwait/pthread_cond_wait`，那么这次唤醒信号就会被丢失。


## 范例
库的初始化中可能会产生多个子线程，子线程也会进行初始化。只有所有的 子线程初始化成功之后，以及库中自己的初始化成功之后，才算是库的初始化成功。
应用程序调用库的初始化，库中调用 `lib_context_wait_all_ready`，进行 `pthread_cond_timedwait`；子线程调用 `lib_context_report_success` 或者 `lib_context_report_failure` 进行 `pthread_cond_broadcast`；如果子线程先执行了 `pthread_cond_broadcast`，然后库中才执行到 `pthread_cond_timedwait`, 此时会发生什么？


### 异常情况讲解

发生过程详解：
**子线程初始化完成：** 假设子线程先于主线程完成初始化，并调用 `lib_context_report_success`，其中包含 `pthread_cond_broadcast`。
**信号发送：** `pthread_cond_broadcast` 发送了一个唤醒信号。
**无等待者：** 此时，主线程（库初始化）**尚未**进入 `pthread_cond_timedwait` 状态，即条件变量上**没有线程在等待**。
**信号丢失：** 条件变量的设计机制决定了它**不保存历史信号**。这个 `broadcast` 信号没有唤醒任何线程，并且立即消失了。
**主线程等待：** 主线程随后执行 `lib_context_wait_all_ready`，并调用 `pthread_cond_timedwait`。
 **陷入死锁/超时：** 主线程会一直等待，直到满足以下任一条件：
    - **超时：** 如果设置了超时时间，主线程会返回 `ETIMEDOUT` 错误。
    - **永久等待：** 如果没有超时时间，主线程将**永久阻塞**。

在这种情况下，尽管子线程已经成功初始化，但主线程**永远也感知不到**这个成功信号，导致库初始化失败（超时）或程序卡死（永久等待）。

# pthread_mutex_t与条件变量pthread_cond_t的组合来实现生产者和消费者
## 整体流程
![](attachments/Pasted%20image%2020260427182836.png)
![](attachments/Pasted%20image%2020260427182938.png)



## 生产者流程
![](attachments/Pasted%20image%2020260427183208.png)
![](attachments/Pasted%20image%2020260427183733.png)

## 伪代码实现

```c
#define MAX 4
int buffer[MAX], count = 0, in = 0, out = 0;
// MAX: 缓冲区最大元素个数
// count: 当前缓冲区实际的元素的个数
// in: 生产者指针
// out: 消费者指针

pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
// 互斥量：既可以做生产者和消费者的同步；又可以做生产者和生产者的互斥；消费者和消费者的互斥；
pthread_cond_t  not_full  = PTHREAD_COND_INITIALIZER;
// not_full: 缓冲区非满的信号
pthread_cond_t  not_empty = PTHREAD_COND_INITIALIZER;
// not_empty: 缓冲区非空的信号
```

`pthread_cond_wait` 是原子操作，它在挂起线程的同时自动释放 mutex，被唤醒后重新竞争锁，避免死锁。

**等待条件必须用 `while` 而非 `if`**，因为 `pthread_cond_signal` 可能产生虚假唤醒（spurious wakeup），用 `while` 确保被唤醒后重新验证条件。
两个条件变量分工明确：`not_full` 让生产者等待空位、消费者释放空位后发出信号；`not_empty` 让消费者等待数据、生产者写入后发出信号，两者协作形成完整的同步机制。

```c
void *producer(void *arg) {
    while (1) {
        pthread_mutex_lock(&mutex);
        while (count == MAX)                         // ← 用 while 防止虚假唤醒
            pthread_cond_wait(&not_full, &mutex);    // 自动释放锁并挂起
        buffer[in] = produce_item();
        in = (in + 1) % MAX;
        count++;
        pthread_cond_signal(&not_empty);             // 通知消费者
        pthread_mutex_unlock(&mutex);
    }
}

void *consumer(void *arg) {
    while (1) {
        pthread_mutex_lock(&mutex);
        while (count == 0)                           // ← 同样用 while
            pthread_cond_wait(&not_empty, &mutex);   // 自动释放锁并挂起
        int item = buffer[out];
        out = (out + 1) % MAX;
        count--;
        pthread_cond_signal(&not_full);              // 通知生产者
        pthread_mutex_unlock(&mutex);
        consume_item(item);
    }
}
```

# 参考
```bash

```