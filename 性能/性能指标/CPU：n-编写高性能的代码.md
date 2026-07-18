```table-of-contents
```
# 推测执行(Speculative Execution)和链表
现代 CPU 中用于提升执行性能的两种并行 执行特性：乱序执行（Out-of-Order Execution）和推测执行（Speculative Execution）。
## 背景
动态数据结构（如链表和搜索树）具有静态数据结构（如数组）无法比拟的优点：它们在内存消耗方面经济实惠，提供快速的插入和删除机制，并且可以高效地调整大小。然而，某些动态数据结构的一个主要缺点是它们妨碍了处理器预测未来可能执行的内存加载指令的能力。
在非常大的动态数据结构中，大多数指针访问导致高延迟的外部内存访问时，这种缺乏并发性尤其令人担忧。

一般而言，主机内存访问延迟约为L1缓存的50倍，因此，有可能看到，尽管进程显示了100％的CPU利用率，但大部分时间都在“等待”。

## 优化前
```c
unsigned long sequentialSum(size_t arr_size, list **la) {
    list *lp;
    unsigned long  res = 0; 

    for (int i = 0; i < arr_size; i++) { 
        lp = la[i]; 
        while (lp) { 
            res += lp->val;
            lp = lp->next;
        }
    }

    return res; 
}
```


## 优化
![](../../语言/C语言/attachments/Pasted%20image%2020250527181651.png)
### 优化思想
通过增加并行内存访问的数量，从而减少外部内存访问的延迟对性能的影响。
通过交错执行随机内存位置访问的操作，可以实现比顺序执行它们显著更好的性能。

### 优化1
```c
unsigned long interleavedSum(size_t arr_size, list **la) {
    list **lthreads = malloc(arr_size * sizeof(list *)); 
    unsigned long res = 0; 
    int n = arr_size; 

    for (int i = 0; i < arr_size; i++) {
        lthreads[i] = la[i]; 
        if (lthreads[i] == NULL) 
            n--; 
    } 

    while(n) {
        for (int i = 0; i < arr_size; i++) { 
            if (lthreads[i] == NULL) 
                continue; 
            res += lthreads[i]->val;
            lthreads[i] = lthreads[i]->next; 
            if (lthreads[i] == NULL) 
                n--;
        }  
    }

    free(lthreads);
    return res; 
}
```
### 优化2
在优化1的基础上，`malloc` 都可以不用，直接使用一个动态数组（变量作为数组的参数）。
```c
	 if (lthreads[i] == NULL) 
		n--;
```

优化上面的代码为下面的代码。
```c
	if (lthreads[i]) 
		__builtin_prefetch(lthreads[i]);
	else 
		n--;
```


# 参考
```bash
https://valkey.io/blog/unlock-one-million-rps-part2/

# 帮助编译器
https://www.dataorienteddesign.com/cn-dod-main/node10.html#SECTION001080000000000000000
```