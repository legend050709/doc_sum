# 查看是否支持大页
系统能否支持大页，支持大页的大小为多少是由其使用的**处理器**决定的。以Intel 的处理器为例，如果处理器的功能列表有PSE，那么它就支持2MB大小的大页；如果处理器的功能列表有PDPE1GB，那么就支持1GB大小的大页。当然，不同体系架构支持的大页的大小都不尽相同，比如x86处理器架构的2MB和1GB大页，而在IBM Power架构中，大页的大小则为16MB和16GB。
![](attachments/Pasted%20image%2020230904161132.png)
# 大页的配置流程

## 激活大页
Linux操作系统采用了基于hugetlbfs的特殊文件系统来加入对2MB或者1GB的大页面支持。这种采用特殊文件系统形式支持大页面的方式，使得应用程序可以根据需要灵活地选择虚存页面大小，而不会被强制使用2MB大页面。

为了使用大页，必须在编译内核的时候激活hugetlbfs。  
```c
# cat /boot/config-`uname -r` | grep -i huge
CONFIG_ARCH_WANT_HUGE_PMD_SHARE=y
CONFIG_ARCH_WANT_GENERAL_HUGETLB=y
CONFIG_CGROUP_HUGETLB=y
CONFIG_HAVE_ARCH_TRANSPARENT_HUGEPAGE=y
CONFIG_HAVE_ARCH_TRANSPARENT_HUGEPAGE_PUD=y
CONFIG_HAVE_ARCH_HUGE_VMAP=y
CONFIG_ARCH_ENABLE_HUGEPAGE_MIGRATION=y
CONFIG_TRANSPARENT_HUGEPAGE=y
CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS=y
# CONFIG_TRANSPARENT_HUGEPAGE_MADVISE is not set
CONFIG_TRANSPARENT_HUGE_PAGECACHE=y
CONFIG_HUGETLBFS=y
CONFIG_HUGETLB_PAGE=y
```

在激活hugetlbfs之后，还必须在Linux启动之后保留一定数量的内存作为大页来使用。现在有两种方式来预留内存。
第一种是在Linux命令行指定，这样Linux启动之后内存就已经预留；第二种方式是在Linux启动之后，可以动态地预留内存作为大页使用。
注：**对于大小为1GB的大页，则必须在Linux命令行的时候就指定，不能动态预留。**

### 启动设置
注意：**HugePages内存页是不会被系统交换出去（swapped out）的。  **
由于`HugePages`需要更大的连续物理内存，所以在系统启动时更容易获得更多的`HugePages`内存，并且还能尽量保证这些`HugePages`内存页连续。
可以通过添加对应的内核启动参数来实现:

例子：可以通过添加对应的内核启动参数来实现:

|项目|描述|
|---|---|
|hugepages|`HugePages`个数|
|hugepagesz|单个`HugePages`字节大小|
|default_hugepagesz|默认`HugePages`字节大小|

>注意： 如果系统不支持设置的默认`HugePages`内存页大小，实际的`HugePages`内存页大小会保持2M。

以下是2MB大页命令行的参数。  
Huagepage=1024  
对于其他大小的大页，比如1GB，其大小必须显示地在命令行指定，并且命令行还可以指定默认的大页大小。比如，我们想预留4GB内存作为大页使用，大页的大小为1GB，那么可以用以下的命令行：  
```c
default_hugepagesz=1G hugepagesz=1G hugepages=4
```

**具体流程**:
- 更改 /etc/default/grub
- 生成新的grub文件：grub2-mkconfig -o /boot/grub2/grub.cfg
- 查看grub2文件配置：cat  /boot/grub2/grub.cfg
- reboot
- 查看验证：cat /proc/cmdline
注：其实 /etc/grub2.cfg 就是软链接到 /boot/grub2/grub.cfg。

### 动态设置
存在2中方法，来进行动态设置，一个是通过/proc接口，一个是通过/sys接口。目前更倾向于使用/sys接口。
![](attachments/Pasted%20image%2020230905115552.png)

在我们之后会讲到的NUMA系统中，因为存在本地内存的问题，系统会均分地预留大页。假设在有两个处理器的NUMA系统中，以上例预留4GB内存为例，在NODE0和NODE1上会各预留2GB内存。

在Linux启动之后，如果想预留大页，则可以使用以下的方法来预留内存。
- 非NUMA系统
可以使用以下方法预留2MB大小的大页
```c
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages
```
该命令预留1024个大小为2MB的大页，也就是预留了2GB内存。

- NUMA系统
假设有两个NODE的系统中，则可以用以下的命令：
```c
echo 1024 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
echo 1024 > /sys/devices/system/node/node1/hugepages/hugepages-2048kB/nr_hugepages
```
该命令在NODE0和NODE1上各预留1024个大小为2MB的大页，总共预留了4GB大小。

### 多种大页设置
```c
default_hugepagesz=1G hugepagesz=1G hugepages=16 hugepagesz=2M hugepages=2048 iommu=pt intel_iommu=on isolcpus=1-13,15-27
```
如果系统支持多种大小的`HugePages`，就会有多个`/sys/kernel/mm/hugepages/hugepages-${huge_page_size}`目录。  
`${huge_page_size}`为`HugePages`的字节数。默认是`2048kB`。

每一个目录下存在相同的的一些HugePages系统文件。可用于设置或者查看系统信息。

|项目	|描述	|权限|
| --- | --- | --- | --- | 
|nr_hugepages	|写入常驻或读取实际的HugePages个数	|读写|
|nr_hugepages_mempolicy|	NUMA node的HugePages个数	|读写|
|nr_overcommit_hugepages	|最大容许超发HugePages个数	|读写|
|free_hugepages|	空闲HugePages个数|	只读|
|resv_hugepages	|保留的HugePages个数	|只读|
|surplus_hugepages|	实际使用的超发HugePages个数|	只读|

如果系统支持多种大小的`HugePages`，每个`NUMA node`就会有多个`/sys/devices/system/node/node${numa_node_id}/hugepages/hugepages-${huge_page_size}`目录。


## 挂载设置
在大页预留之后，接下来则涉及使用的问题。我们以DPDK为例来说明如何使用大页。  
DPDK也是使用HUGETLBFS来使用大页。
### 动态设置
首先，它需要把大页mount到某个路径，比如/mnt/huge，以下是命令：  
```c
mkdir /mnt/huge  
mount -t hugetlbfs nodev /mnt/huge  
or 
mount -t hugetlbfs hugetlbfs /mnt/huge

注：没有-o参数，挂载系统默认hugepage大小。默认大小指的是 cat  /proc/cmdline  | grep -i default_huge 中的大小。


mkdir -p /mnt/huge_2mb
mount -t hugetlbfs nodev /mnt/huge_2mb -o pagesize=2MB（-o参数指定挂载2M的hugepage大小）
```
注：需要指出的是，在mount之前，要确保之前已经成功预留内存，否则之上命令会失败。
如下所示，是挂载到了 /dev/hugepages 目录下。
![](attachments/Pasted%20image%2020230904161424.png)

### 启动设置
mount命令只是临时的mount了文件系统，如果想每次开机时省略该步骤，可以修改/etc/fstab文件，加上一行：
```c
nodev /mnt/huge hugetlbfs defaults 0 0  

对于1GB大小的大页，则必须用如下的命令：  
nodev /mnt/huge_1GB hugetlbfs pagesize=1GB 0 0
```
## 查看
### **meminfo**
![](attachments/Pasted%20image%2020230905111725.png)


```c
# cat /proc/meminfo | grep -i huge
AnonHugePages:     20480 kB
ShmemHugePages:        0 kB
HugePages_Total:     100
HugePages_Free:       70
HugePages_Rsvd:        0
HugePages_Surp:        0
Hugepagesize:    1048576 kB
Hugetlb:        104857600 kB
```
其中HugePages内存信息的解释如下：

|项目|描述|
|---|---|
|HugePages_Total|系统当前总共拥有的HugePages个数。|
|HugePages_Free	|系统当前总共拥有的空闲HugePages个数。|
|HugePages_Rsvd	|(reserved)系统当前总共保留的HugePages个数(备注1)|
|HugePages_Surp	|(surplus)实际使用的超发HugePages个数。(备注2)|
|Hugepagesize|	每一页HugePages的字节大小。|

**备注1**：指程序已经向系统申请，但是由于程序还没有对`HugePages`实质的读写操作，系统尚未实际分配给程序的`HugePages`个数。

**备注2**：可以通过`/proc/sys/vm/nr_overcommit_hugepages`来控制最大超发的`HugePages`个数。

HugePages_Total 对应内核参数 vm.nr_hugepages，也可以在运行中的系统上直接修改 /proc/sys/vm/nr_hugepages，修改的结果会立即影响空闲内存 MemFree的大小，因为HugePages在内核中独立管理，只要一经定义，无论是否被使用，都不再属于free memory。

```c
### 内存池数据结构

// include\linux\hugetlb.h
struct hstate {
    int next_nid_to_alloc;
    int next_nid_to_free;
    // 页面大小对应伙伴系统的order
    unsigned int order;
    unsigned long mask;
    // 最大可使用的大页数
    unsigned long max_huge_pages;
    // 内存池静态大页数
    unsigned long nr_huge_pages;
    // 内存池空闲大页数
    unsigned long free_huge_pages;
    // 内存池承诺分配但未最终分配的大页数
    unsigned long resv_huge_pages;
    // 内存池超额分配的大页数
    unsigned long surplus_huge_pages;
    // 内存池允许最大超额数
    unsigned long nr_overcommit_huge_pages;
    struct list_head hugepage_activelist;
    struct list_head hugepage_freelists[MAX_NUMNODES];
    unsigned int nr_huge_pages_node[MAX_NUMNODES];
    unsigned int free_huge_pages_node[MAX_NUMNODES];
    unsigned int surplus_huge_pages_node[MAX_NUMNODES];
    // 内存池名称
    char name[HSTATE_NAME_LEN];
};

// mm\hugetlb.c
struct hstate hstates[HUGE_MAX_HSTATE];

简单说以下数据结构中的几个字段：

@ max_huge_pages：内存池最大能使用的大页，包括静态预留以及最大允许超额的数量。

@ nr_huge_pages：内存池当前总共大页数量，包括静态预留以及超额的数量。

@ free_huge_pages：内存池当前空闲大页数量，包括承诺分配但未最终分配的部分。

@ resv_huge_pages：内存池承诺分配但未最终分配的数量，所以内存池真正空闲的数量应该是free_huge_pages - resv_huge_pages。

@ surplus_huge_pages：内存池超额分配的大页数量，当进程使用完毕后，内存池会归还给伙伴系统。

@ nr_overcommit_huge_pages：内存池允许最大超额的大页数量。

有如下的公式：max_huge_pages = nr_huge_pages - surplus_huge_pages + nr_overcommit_huge_pages。
```

####  Hugetlbfs Reservation
参考：[Hugetlbfs Reservation](https://www.kernel.org/doc/html/v4.18/vm/hugetlbfs_reserv.html)
![](attachments/Pasted%20image%2020230904201425.png)
### 其他
```c
# ll /sys/devices/system/node/node0/hugepages
drwxr-xr-x 2 root root 0 Aug 31 18:31 hugepages-1048576kB

# ll /sys/devices/system/node/node0/hugepages/hugepages-1048576kB/
-r--r--r-- 1 root root 4096 Aug 31 19:41 free_hugepages
-rw-r--r-- 1 root root 4096 Sep  4 16:23 nr_hugepages
-r--r--r-- 1 root root 4096 Sep  4 16:23 surplus_hugepages
```

```c
# ll /sys/kernel/mm/hugepages/
drwxr-xr-x 2 root root 0 Aug 31 18:34 hugepages-1048576kB

# ll /sys/kernel/mm/hugepages/hugepages-1048576kB/
-r--r--r-- 1 root root 4096 Sep  4 16:24 free_hugepages
-rw-r--r-- 1 root root 4096 Sep  4 16:24 nr_hugepages
-rw-r--r-- 1 root root 4096 Sep  4 16:24 nr_hugepages_mempolicy
-rw-r--r-- 1 root root 4096 Sep  4 16:24 nr_overcommit_hugepages
-r--r--r-- 1 root root 4096 Sep  4 16:24 resv_hugepages
-r--r--r-- 1 root root 4096 Sep  4 16:24 surplus_hugepages
```
```c
# ls /dev/hugepages/
rtemap_0  rtemap_10  rtemap_12  rtemap_14  rtemap_16384  rtemap_16386  rtemap_16388  rtemap_16390  rtemap_16392  rtemap_16394  rtemap_16396  rtemap_2  rtemap_4  rtemap_6  rtemap_8
rtemap_1  rtemap_11  rtemap_13  rtemap_15  rtemap_16385  rtemap_16387  rtemap_16389  rtemap_16391  rtemap_16393  rtemap_16395  rtemap_16397  rtemap_3  rtemap_5  rtemap_7  rtemap_9

# ls /dev/hugepages/ |wc
     30      30     332
说明，已经使用了30个大页。和meminfo中的 total - free 是对应的。
```
## 程序中申请大页
```c
Users can use the huge page support in Linux kernel by either using the mmap
system call or standard SYSV shared memory system calls (shmget, shmat).
```
![](attachments/Pasted%20image%2020230905121408.png)
- 范例
```c
#include <sys/mman.h>
#include <stdio.h>
#include <memory.h>
 
int main(int argc, char *argv[]) {
  char *m;
  size_t s = (8UL * 1024 * 1024);
 
  m = mmap(NULL, s, PROT_READ | PROT_WRITE,
                    MAP_PRIVATE | MAP_ANONYMOUS | 0x40000 /*MAP_HUGETLB*/, -1, 0);
  if (m == MAP_FAILED) {
    perror("map mem");
    m = NULL;
    return 1;
  }
 
  memset(m, 0, s);
 
  printf("map_hugetlb ok, press ENTER to quit!\n");
  getchar();
 
  munmap(m, s);
  return 0;
}
```

- tools/testing/selftests/vm/map_hugetlb.c
```c
// SPDX-License-Identifier: GPL-2.0

/*

* Example of using hugepage memory in a user application using the mmap

* system call with MAP_HUGETLB flag. Before running this program make

* sure the administrator has allocated enough default sized huge pages

* to cover the 256 MB allocation.

*

* For ia64 architecture, Linux kernel reserves Region number 4 for hugepages.

* That means the addresses starting with 0x800000... will need to be

* specified. Specifying a fixed address is not required on ppc64, i386

* or x86_64.

*/

#include <stdlib.h>

#include <stdio.h>

#include <unistd.h>

#include <sys/mman.h>

#include <fcntl.h>

  

#define LENGTH (256UL*1024*1024)

#define PROTECTION (PROT_READ | PROT_WRITE)

  

#ifndef MAP_HUGETLB

#define MAP_HUGETLB 0x40000 /* arch specific */

#endif

  

/* Only ia64 requires this */

#ifdef __ia64__

#define ADDR (void *)(0x8000000000000000UL)

#define FLAGS (MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB | MAP_FIXED)

#else

#define ADDR (void *)(0x0UL)

#define FLAGS (MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB)

#endif

  

static void check_bytes(char *addr)

{

printf("First hex is %x\n", *((unsigned int *)addr));

}

  

static void write_bytes(char *addr)

{

unsigned long i;

  

for (i = 0; i < LENGTH; i++)

*(addr + i) = (char)i;

}

  

static int read_bytes(char *addr)

{

unsigned long i;

  

check_bytes(addr);

for (i = 0; i < LENGTH; i++)

if (*(addr + i) != (char)i) {

printf("Mismatch at %lu\n", i);

return 1;

}

return 0;

}

  

int main(void)

{

void *addr;

int ret;

  

addr = mmap(ADDR, LENGTH, PROTECTION, FLAGS, -1, 0);

if (addr == MAP_FAILED) {

perror("mmap");

exit(1);

}

  

printf("Returned address is %p\n", addr);

check_bytes(addr);

write_bytes(addr);

ret = read_bytes(addr);

  

/* munmap() length of MAP_HUGETLB memory must be hugepage aligned */

if (munmap(addr, LENGTH)) {

perror("munmap");

exit(1);

}

  

return ret;

}
```

- tools/testing/selftests/vm/hugepage-mmap.c
```c
// SPDX-License-Identifier: GPL-2.0

/*

* hugepage-mmap:

*

* Example of using huge page memory in a user application using the mmap

* system call. Before running this application, make sure that the

* administrator has mounted the hugetlbfs filesystem (on some directory

* like /mnt) using the command mount -t hugetlbfs nodev /mnt. In this

* example, the app is requesting memory of size 256MB that is backed by

* huge pages.

*

* For the ia64 architecture, the Linux kernel reserves Region number 4 for

* huge pages. That means that if one requires a fixed address, a huge page

* aligned address starting with 0x800000... will be required. If a fixed

* address is not required, the kernel will select an address in the proper

* range.

* Other architectures, such as ppc64, i386 or x86_64 are not so constrained.

*/

  

#include <stdlib.h>

#include <stdio.h>

#include <unistd.h>

#include <sys/mman.h>

#include <fcntl.h>

  

#define FILE_NAME "huge/hugepagefile"

#define LENGTH (256UL*1024*1024)

#define PROTECTION (PROT_READ | PROT_WRITE)

  

/* Only ia64 requires this */

#ifdef __ia64__

#define ADDR (void *)(0x8000000000000000UL)

#define FLAGS (MAP_SHARED | MAP_FIXED)

#else

#define ADDR (void *)(0x0UL)

#define FLAGS (MAP_SHARED)

#endif

  

static void check_bytes(char *addr)

{

printf("First hex is %x\n", *((unsigned int *)addr));

}

  

static void write_bytes(char *addr)

{

unsigned long i;

  

for (i = 0; i < LENGTH; i++)

*(addr + i) = (char)i;

}

  

static int read_bytes(char *addr)

{

unsigned long i;

  

check_bytes(addr);

for (i = 0; i < LENGTH; i++)

if (*(addr + i) != (char)i) {

printf("Mismatch at %lu\n", i);

return 1;

}

return 0;

}

  

int main(void)

{

void *addr;

int fd, ret;

  

fd = open(FILE_NAME, O_CREAT | O_RDWR, 0755);

if (fd < 0) {

perror("Open failed");

exit(1);

}

  

addr = mmap(ADDR, LENGTH, PROTECTION, FLAGS, fd, 0);

if (addr == MAP_FAILED) {

perror("mmap");

unlink(FILE_NAME);

exit(1);

}

  

printf("Returned address is %p\n", addr);

check_bytes(addr);

write_bytes(addr);

ret = read_bytes(addr);

  

munmap(addr, LENGTH);

close(fd);

unlink(FILE_NAME);

  

return ret;

}
```


# 参考
```c
https://blog.csdn.net/shaoyunzhe/article/details/54614077
https://blog.csdn.net/yk_wing4/article/details/88080442
https://toutiao.io/posts/n4hzg1/preview
https://cloud.tencent.com/developer/article/1816836
https://zhuanlan.zhihu.com/p/351300228
https://www.cnblogs.com/studywithallofyou/p/17435497.html
https://www.kernel.org/doc/Documentation/vm/hugetlbpage.txt

大页为什么要挂载
https://developer.aliyun.com/article/90383


大页内存的使用
https://blog.csdn.net/Rong_Toa/article/details/108227075


dpdk中大页的原理
https://blog.csdn.net/ApeLife/article/details/99700882

```