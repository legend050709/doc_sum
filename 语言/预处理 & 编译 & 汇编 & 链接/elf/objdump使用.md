```table-of-contents
```

# 使用方法

# 常见使用场景
## `objdump -T xxx`
```bash
$ ll /lib64/libmlx5.so*
lrwxrwxrwx 1 root root     12 Dec  1 12:11 /lib64/libmlx5.so -> libmlx5.so.1
lrwxrwxrwx 1 root root     27 Dec  3 12:51 /lib64/libmlx5.so.1 -> /lib64/libmlx5.so.1.14.30.0
-rwxr-xr-x 1 root root 326232 Sep 14  2020 /lib64/libmlx5.so.1.14.30.0
```

```bash
$ objdump -T /lib64/libmlx5.so.1.14.30.0 | grep OFED
000000000003d560 g    DF .text	00000000000002d2  MLX5_OFED   mlx5dv_query_devx_port
000000000001ad80 g    DF .text	0000000000000094  MLX5_OFED   mlx5dv_dr_action_create_dest_ib_port
000000000001a200 g    DF .text	0000000000000044  MLX5_OFED   mlx5dv_dr_action_create_push_vlan
000000000001a0e0 g    DF .text	0000000000000044  MLX5_OFED   mlx5dv_dr_action_create_dest_devx_tir
0000000000000000 g    DO *ABS*	0000000000000000  MLX5_OFED   MLX5_OFED
000000000001af60 g    DF .text	0000000000000667  MLX5_OFED   mlx5dv_dr_action_create_flow_sampler
000000000001ad70 g    DF .text	000000000000000a  MLX5_OFED   mlx5dv_dr_action_create_pop_vlan
```

### 字段说明


### `objdump -T xxx` 等效于`readelf -Ws xxx`
```bash
$ readelf -Ws /lib64/libmlx5.so.1.14.30.0 | grep OFED
   129: 000000000003d560   722 FUNC    GLOBAL DEFAULT   12 mlx5dv_query_devx_port@@MLX5_OFED
   142: 000000000001ad80   148 FUNC    GLOBAL DEFAULT   12 mlx5dv_dr_action_create_dest_ib_port@@MLX5_OFED
   143: 000000000001a200    68 FUNC    GLOBAL DEFAULT   12 mlx5dv_dr_action_create_push_vlan@@MLX5_OFED
   148: 000000000001a0e0    68 FUNC    GLOBAL DEFAULT   12 mlx5dv_dr_action_create_dest_devx_tir@@MLX5_OFED
   153: 0000000000000000     0 OBJECT  GLOBAL DEFAULT  ABS MLX5_OFED
   166: 000000000001af60  1639 FUNC    GLOBAL DEFAULT   12 mlx5dv_dr_action_create_flow_sampler@@MLX5_OFED
   210: 000000000001ad70    10 FUNC    GLOBAL DEFAULT   12 mlx5dv_dr_action_create_pop_vlan@@MLX5_OFED
```



#### 字段说明


## `objdump -p xxx | grep -i version -A 200`

```bash
$ objdump -p /lib64/libmlx5.so.1.14.30.0 | grep -i version -A 100 -B 50

/lib64/libmlx5.so.1.14.30.0:     file format elf64-x86-64

Program Header:
    LOAD off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**21
         filesz 0x000000000004ba94 memsz 0x000000000004ba94 flags r-x
    LOAD off    0x000000000004c5c0 vaddr 0x000000000024c5c0 paddr 0x000000000024c5c0 align 2**21
         filesz 0x00000000000011f0 memsz 0x0000000000003230 flags rw-
 DYNAMIC off    0x000000000004cda8 vaddr 0x000000000024cda8 paddr 0x000000000024cda8 align 2**3
         filesz 0x0000000000000220 memsz 0x0000000000000220 flags rw-
    NOTE off    0x00000000000001c8 vaddr 0x00000000000001c8 paddr 0x00000000000001c8 align 2**2
         filesz 0x0000000000000024 memsz 0x0000000000000024 flags r--
EH_FRAME off    0x0000000000043598 vaddr 0x0000000000043598 paddr 0x0000000000043598 align 2**2
         filesz 0x0000000000001384 memsz 0x0000000000001384 flags r--
   STACK off    0x0000000000000000 vaddr 0x0000000000000000 paddr 0x0000000000000000 align 2**4
         filesz 0x0000000000000000 memsz 0x0000000000000000 flags rw-
   RELRO off    0x000000000004c5c0 vaddr 0x000000000024c5c0 paddr 0x000000000024c5c0 align 2**0
         filesz 0x0000000000000a40 memsz 0x0000000000000a40 flags r--

Dynamic Section:
  NEEDED               libibverbs.so.1
  NEEDED               libpthread.so.0
  NEEDED               libc.so.6
  SONAME               libmlx5.so.1
  INIT                 0x00000000000056c8
  FINI                 0x0000000000041340
  INIT_ARRAY           0x000000000024c5c0
  INIT_ARRAYSZ         0x0000000000000010
  FINI_ARRAY           0x000000000024c5d0
  FINI_ARRAYSZ         0x0000000000000008
  GNU_HASH             0x00000000000001f0
  STRTAB               0x0000000000001b60
  SYMTAB               0x00000000000005a0
  STRSZ                0x00000000000011f5
  SYMENT               0x0000000000000018
  PLTGOT               0x000000000024d000
  PLTRELSZ             0x0000000000000e40
  PLTREL               0x0000000000000007
  JMPREL               0x0000000000004888
  RELA                 0x0000000000003220
  RELASZ               0x0000000000001668
  RELAENT              0x0000000000000018
  VERDEF               0x0000000000002f28
  VERDEFNUM            0x0000000000000011
  VERNEED              0x0000000000003180
  VERNEEDNUM           0x0000000000000003
  VERSYM               0x0000000000002d56
  RELACOUNT            0x00000000000000e8

Version definitions:
1 0x01 0x0c6a1621 libmlx5.so.1
2 0x00 0x01bb2730 MLX5_1.0
3 0x00 0x01bb2731 MLX5_1.1
	MLX5_1.0
4 0x00 0x01bb2732 MLX5_1.2
	MLX5_1.1
5 0x00 0x01bb2733 MLX5_1.3
	MLX5_1.2
6 0x00 0x01bb2734 MLX5_1.4
	MLX5_1.3
7 0x00 0x01bb2735 MLX5_1.5
	MLX5_1.4
8 0x00 0x01bb2736 MLX5_1.6
	MLX5_1.5
9 0x00 0x01bb2737 MLX5_1.7
	MLX5_1.6
10 0x00 0x01bb2738 MLX5_1.8
	MLX5_1.7
11 0x00 0x01bb2739 MLX5_1.9
	MLX5_1.8
12 0x00 0x0bb27350 MLX5_1.10
	MLX5_1.9
13 0x00 0x0bb27351 MLX5_1.11
	MLX5_1.10
14 0x00 0x0bb27352 MLX5_1.12
	MLX5_1.11
15 0x00 0x0bb27353 MLX5_1.13
	MLX5_1.12
16 0x00 0x0bb27354 MLX5_1.14
	MLX5_1.13
17 0x00 0x0bb46884 MLX5_OFED
	MLX5_1.14

Version References:
  required from libpthread.so.0:
    0x09691a75 0x00 21 GLIBC_2.2.5
  required from libibverbs.so.1:
    0x06365ab1 0x00 22 IBVERBS_1.1
    0x090b6ed5 0x00 19 IBVERBS_PRIVATE_25
  required from libc.so.6:
    0x06969194 0x00 24 GLIBC_2.14
    0x0d696914 0x00 23 GLIBC_2.4
    0x09691974 0x00 20 GLIBC_2.3.4
    0x09691a75 0x00 18 GLIBC_2.2.5
```


# 相关问题
## `function used but not defined`问题

## 在动态链接库中使用static静态函数的问题
### 问题
我们知道static静态函数相比于全局函数，会把范围限定在自己所在文件的范围，对其他的编译单元是不可见的！
那么若在动态库中定义了static静态函数并完成库的编译，那么在主程序加载并试图调用这个static静态函数的时候会找不到它！或者 `objdump -t xx.so | grep xxx` 发现在编译生成的动态库中也是找不到这个`static` 函数的符号表。

因此需要在主程序中直接调用的函数在动态库中不要定义和声明为`static`类型！
# 参考
```bash

```