```table-of-contents
```
# 介绍
# 安装和编译
```bash
wget https://github.com/linux-rdma/perftest

yum install -y automake libtool pciutils-devel

cd perftest/

./autogen.sh

./configure

make -j 9

当前 目录下输出查看：
# ll ib_*
-rwxr-xr-x 1 root root 1042776 Mar 14 16:56 ib_atomic_bw
-rwxr-xr-x 1 root root 1036728 Mar 14 16:56 ib_atomic_lat
-rwxr-xr-x 1 root root 1042664 Mar 14 16:56 ib_read_bw
-rwxr-xr-x 1 root root 1035248 Mar 14 16:56 ib_read_lat
-rwxr-xr-x 1 root root 1088552 Mar 14 16:56 ib_send_bw
-rwxr-xr-x 1 root root 1086496 Mar 14 16:56 ib_send_lat
-rwxr-xr-x 1 root root 1044656 Mar 14 16:56 ib_write_bw
-rwxr-xr-x 1 root root 1036568 Mar 14 16:56 ib_write_lat
```

# 工具集介绍
```bash
ib_send_lat   latency test with send transactions
ib_send_bw    bandwidth test with send transactions
ib_write_lat  latency test with RDMA write transactions
ib_write_bw   bandwidth test with RDMA write transactions
ib_read_lat   latency test with RDMA read transactions
ib_read_bw    bandwidth test with RDMA read transactions
ib_atomic_lat latency test with atomic transactions
ib_atomic_bw  bandwidth test with atomic transactions
```

# 参考
```bash

```