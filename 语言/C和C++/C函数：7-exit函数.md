```table-of-contents
```
# 多线程进程和exit函数
在Linux的多线程进程中，如果**某个线程调用了`exit()`函数**，**整个进程会立即终止**，包括所有其他线程。此时，所有线程的资源会被回收，进程退出。
## `pthread_exit` 和 `exit()`
**（1）`exit()`的作用**： 
`exit()`是C标准库函数，用于终止**当前进程**（而非单个线程）。调用后，进程的所有资源（包括所有线程、内存、文件描述符等）都会被操作系统回收。
    
**（2）`exit()`与`pthread_exit()`的区别**：
- 如果线程调用`pthread_exit()`，则**仅该线程终止**，其他线程继续运行。
- 如果调用`exit()`，则会直接终止整个进程。

### 底层机制
- Linux中，进程是资源管理的最小单位，线程是调度的最小单位。
- `exit()`会触发内核级的进程终止操作（如`exit_group`系统调用），终止所有线程；而`pthread_exit()`仅操作线程级的资源。

### 范例
```c
#include <stdio.h>
#include <pthread.h>
#include <stdlib.h>
#include <unistd.h>

void* thread_func(void* arg) {
    printf("子线程调用exit()\n");
    exit(0); // 整个进程终止
    // pthread_exit(NULL); // 仅子线程终止
}

int main() {
    pthread_t tid;
    pthread_create(&tid, NULL, thread_func, NULL);
    sleep(1); // 主线程等待子线程执行
    printf("主线程仍在运行\n"); // 若子线程调用exit()，此行不会执行
    return 0;
}
```

- 当子线程调用`exit(0)`时，输出仅为`子线程调用exit()`，主线程的`printf`不会执行。
- 若改为`pthread_exit(NULL)`，则会输出两行。



# 参考
```bash

```