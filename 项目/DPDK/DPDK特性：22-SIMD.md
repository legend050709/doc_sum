```table-of-contents
```

# SIMD

SIMD，单指令多数据（Single Instruction Multiple Data），允许在一个指令周期内同时对多个数据元素进行操作。
常见的SIMD指令集包括SSE（Streaming SIMD Extensions）、AVX（Advanced Vector Extensions）和AVX-512等。

例如，一个SIMD指令可以在一个操作中对多个数据点进行加法、乘法等操作，而不是逐个处理每个数据点。
SIMD 指令通过使用更宽的寄存器（如 YMM 寄存器）来处理多个数据单元，能够充分利用处理器缓存的带宽。

## AVX 和 SSE
**AVX** 的全称是 **Advanced Vector Extensions**（高级向量扩展集），是 Intel 在 2011 年推出的 x86 架构 SIMD 指令集，用于替代并扩展 SSE（Streaming SIMD Extensions：流式单指令多数据流扩展集）。

|指令集|寄存器宽度|单指令处理 float 数（4B）|主要改进|
|---|---|---|---|
|**SSE**|128 位|4 个|引入 XMM 寄存器|
|**AVX**|**256 位**|**8 个**|寄存器翻倍，使用 VEX 编码增加指令灵活性|
|**AVX-512**|512 位|16 个|宽度再翻倍，新增掩码寄存器|

## 在 DPDK 中的典型应用

- **`rte_memcpy` 的 AVX 版本**：利用 256 位 YMM 寄存器一次复制 32 字节，比 SSE 版本吞吐量更高。
- **查表（LPM/FIB）**：AVX2 指令加速 256 位宽的多键比较。
- **网卡驱动**：`i40e`、`ice` 等 PMD 驱动使用 AVX2/AVX-512 实现向量化收发包处理。
- **数据包校验和计算**：对 256 位块并行执行计算。

### DPDK中的memcpy函数

```c
static __rte_always_inline void *
rte_memcpy_generic(void *dst, const void *src, size_t n)
{
    __m128i xmm0, xmm1, xmm2, xmm3, xmm4, xmm5, xmm6, xmm7, xmm8;
    void *ret = dst;
    size_t dstofss;
    size_t srcofs;

    /**
     * Copy less than 16 bytes
     */
    if (n < 16) {
        return rte_mov15_or_less(dst, src, n);
    }

    /**
     * Fast way when copy size doesn't exceed 512 bytes
     */
    if (n <= 32) {
        rte_mov16((uint8_t *)dst, (const uint8_t *)src);
        if (__rte_constant(n) && n == 16)
            return ret; /* avoid (harmless) duplicate copy */
        rte_mov16((uint8_t *)dst - 16 + n, (const uint8_t *)src - 16 + n);
        return ret;
    }
    if (n <= 64) {
        rte_mov32((uint8_t *)dst, (const uint8_t *)src);
        if (n > 48)
            rte_mov16((uint8_t *)dst + 32, (const uint8_t *)src + 32);
        rte_mov16((uint8_t *)dst - 16 + n, (const uint8_t *)src - 16 + n);
        return ret;
    }
    if (n <= 128) {
        goto COPY_BLOCK_128_BACK15;
    }
    if (n <= 512) {
        if (n >= 256) {
            n -= 256;
            rte_mov128((uint8_t *)dst, (const uint8_t *)src);
            rte_mov128((uint8_t *)dst + 128, (const uint8_t *)src + 128);
            src = (const uint8_t *)src + 256;
            dst = (uint8_t *)dst + 256;
        }
COPY_BLOCK_255_BACK15:
        if (n >= 128) {
            n -= 128;
            rte_mov128((uint8_t *)dst, (const uint8_t *)src);
            src = (const uint8_t *)src + 128;
            dst = (uint8_t *)dst + 128;
        }
COPY_BLOCK_128_BACK15:
        if (n >= 64) {
            n -= 64;
            rte_mov64((uint8_t *)dst, (const uint8_t *)src);
            src = (const uint8_t *)src + 64;
            dst = (uint8_t *)dst + 64;
        }
COPY_BLOCK_64_BACK15:
        if (n >= 32) {
            n -= 32;
            rte_mov32((uint8_t *)dst, (const uint8_t *)src);
            src = (const uint8_t *)src + 32;
            dst = (uint8_t *)dst + 32;
        }
        if (n > 16) {
            rte_mov16((uint8_t *)dst, (const uint8_t *)src);
            rte_mov16((uint8_t *)dst - 16 + n, (const uint8_t *)src - 16 + n);
            return ret;
        }
        if (n > 0) {
            rte_mov16((uint8_t *)dst - 16 + n, (const uint8_t *)src - 16 + n);
        }
        return ret;
    }

    /**
     * Make store aligned when copy size exceeds 512 bytes,
     * and make sure the first 15 bytes are copied, because
     * unaligned copy functions require up to 15 bytes
     * backwards access.
     */
    dstofss = (uintptr_t)dst & 0x0F;
    if (dstofss > 0) {
        dstofss = 16 - dstofss + 16;
        n -= dstofss;
        rte_mov32((uint8_t *)dst, (const uint8_t *)src);
        src = (const uint8_t *)src + dstofss;
        dst = (uint8_t *)dst + dstofss;
    }
    srcofs = ((uintptr_t)src & 0x0F);

    /**
     * For aligned copy
     */
    if (srcofs == 0) {
        /**
         * Copy 256-byte blocks
         */
        for (; n >= 256; n -= 256) {
            rte_mov256((uint8_t *)dst, (const uint8_t *)src);
            dst = (uint8_t *)dst + 256;
            src = (const uint8_t *)src + 256;
        }

        /**
         * Copy whatever left
         */
        goto COPY_BLOCK_255_BACK15;
    }

    /**
     * For copy with unaligned load
     */
    MOVEUNALIGNED_LEFT47(dst, src, n, srcofs);

    /**
     * Copy whatever left
     */
    goto COPY_BLOCK_64_BACK15;
}
```

对于大于512字节的拷贝，首先对齐目标内存，然后使用256字节块的拷贝方式，如果拷贝量不足256字节，再使用128、64、32、16字节的块进行处理，利用了SSE和AVX指令集的优势。

```c
static __rte_always_inline void
rte_mov64(uint8_t *dst, const uint8_t *src)
{
#if defined __AVX512F__ && defined RTE_MEMCPY_AVX512
    __m512i zmm0;

    zmm0 = _mm512_loadu_si512((const void *)src);
    _mm512_storeu_si512((void *)dst, zmm0);
#else /* AVX2, AVX & SSE implementation */
    rte_mov32((uint8_t *)dst + 0 * 32, (const uint8_t *)src + 0 * 32);
    rte_mov32((uint8_t *)dst + 1 * 32, (const uint8_t *)src + 1 * 32);
#endif
}
```

当编译器检测到支持AVX512指令集时，函数使用 `_mm512_loadu_si512` 和 `_mm512_storeu_si512` 指令进行内存拷贝。这些指令可以同时处理512位的数据（即64字节），显著提高了拷贝速度。

在不支持`AVX512`的情况下，函数退回到使用 `rte_mov32`，进行32字节的数据拷贝。这个实现同样利用了`AVX2/AVX/SSE`指令集的并行处理能力，但处理的块大小较小，需要多次调用来完成64字节的拷贝。


# 参考
```bash
# 高性能网络框架：DPDK 最佳入门指南
https://zhuanlan.zhihu.com/p/4988737495

```