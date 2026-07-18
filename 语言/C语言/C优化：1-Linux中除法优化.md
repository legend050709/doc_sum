```table-of-contents
```

# 场景
比如：获取时间；
可以通过gettimeofday, clock_gettime， 但是这2个可能都不如通过 rdtsc 然后将cycles, 除以CPU的频率换算为us/ns 来的更加快捷。
但是除法是一个比较消耗性能的方法。如何对于除法进行优化。


# 方法
## 乘法+位移的倒数法
## 浮点乘法
**初始化一次浮点除法**
```c
double us_per_cycle = 1e6 / (double)freq_hz;  // 只做一次
```

避免每次调用都做 `/ freq_hz`。

**高频调用只做乘法**
```c
double elapsed_us = cycles * us_per_cycle; 
```
CPU 浮点乘法流水线优化好，几条指令就完成。

**精度**
- double 有 52 位有效尾数，TSC 在秒级以内换算微秒精度足够。
- 如果时间间隔特别大（几小时以上），微小精度误差可能累积。


## 直接整数相除

## 性能对比
```c
#define _GNU_SOURCE
#include <stdio.h>
#include <stdint.h>
#include <time.h>
#include <x86intrin.h> // __rdtsc, __rdtscp

typedef struct {
    uint64_t mult;
    uint8_t shift;
} tsc_scale_t;

// ----------------- rdtsc -----------------
static inline uint64_t rdtsc(void) {
    unsigned hi, lo;
    __asm__ __volatile__("rdtsc" : "=a"(lo), "=d"(hi));
    return ((uint64_t)hi << 32) | lo;
}

static inline uint64_t rdtsc_start(void) {
    unsigned dummy;
    return __rdtscp(&dummy);
}

static inline uint64_t rdtsc_end(void) {
    unsigned dummy;
    uint64_t t = __rdtscp(&dummy);
    _mm_lfence();
    return t;
}

// ----------------- 整数倒数乘法 -----------------
static inline uint64_t tsc_to_us(uint64_t tsc, const tsc_scale_t *s) {
    __uint128_t x = (__uint128_t)tsc * s->mult;
    return (uint64_t)(x >> s->shift);
}

void init_scale_us(tsc_scale_t *s, uint64_t freq_hz) {
    s->shift = 24;
    // 目标单位是 us，所以乘 1e6
    s->mult  = ((1000000ULL << s->shift) + freq_hz/2) / freq_hz;
}

int main(void) {
    const int N = 1000000;
    uint64_t freq_hz = 3000000000ULL; // 假设 3GHz CPU
    tsc_scale_t scale;
    init_scale_us(&scale, freq_hz);

    double us_per_cycle = 1e6 / (double)freq_hz; // 浮点乘法

    struct timespec ts;
    uint64_t start, end;
    uint64_t sum_mul_shift = 0;
    uint64_t sum_fp = 0;
    uint64_t sum_div = 0;
    uint64_t sum_vdso = 0;

    int i;
    // 1️⃣ 整数乘法+右移 (us)
    for (i = 0; i < N; i++) {
        start = rdtsc_start();
        uint64_t c = rdtsc();
        uint64_t us = tsc_to_us(c, &scale);
        (void)us;
        end = rdtsc_end();
        sum_mul_shift += end - start;
    }

    // 2️⃣ 浮点乘法 (us)
    for (i = 0; i < N; i++) {
        start = rdtsc_start();
        uint64_t c = rdtsc();
        double us = c * us_per_cycle;
        (void)us;
        end = rdtsc_end();
        sum_fp += end - start;
    }

    // 3️⃣ 整数直接除法 (us)
    for (i = 0; i < N; i++) {
        start = rdtsc_start();
        uint64_t c = rdtsc();
        uint64_t us = c * 1000000ULL / freq_hz;
        (void)us;
        end = rdtsc_end();
        sum_div += end - start;
    }

    // 4️⃣ vDSO clock_gettime
    for (i = 0; i < N; i++) {
        start = rdtsc_start();
        clock_gettime(CLOCK_REALTIME, &ts);
        end = rdtsc_end();
        sum_vdso += end - start;
    }

    printf("Avg cycles per call:\n");
    printf("  Integer multiply+shift (us): %.2f cycles\n", (double)sum_mul_shift / N);
    printf("  Floating point multiply (us): %.2f cycles\n", (double)sum_fp / N);
    printf("  Integer direct division (us): %.2f cycles\n", (double)sum_div / N);
    printf("  vDSO clock_gettime: %.2f cycles\n", (double)sum_vdso / N);

    return 0;
}
```

测试结果如下所示：
```bash
# gcc -std=gnu99 test.c -o test

# ./test
Avg cycles per call:
  Integer multiply+shift (us): 63.87 cycles
  Floating point multiply (us): 57.21 cycles
  Integer direct division (us): 76.63 cycles
  vDSO clock_gettime: 72.57 cycles
```

# linux内核中vDSO 的 `clock_gettime` / `gettimeofday`的实现


# 参考
```bash

```