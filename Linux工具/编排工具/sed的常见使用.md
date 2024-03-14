```table-of-contents
```
# 概述
# 场景
# 使用方法
# 操作
## 提取
### 从指定的字符串中提取多个内容
将目标用()括起来，然后输出的时候使用\n来代替进行输出。

```c
# ss -tman | grep skmem
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o136,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb787968,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb4194304,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb332800,f4096,w0,o136,bl0)
	 skmem:(r0,rb369280,t0,tb374272,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb46080,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t1408,tb87040,f3968,w4224,o0,bl0)
	 skmem:(r0,rb369280,t0,tb678912,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb2491109,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb217600,f0,w0,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o136,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f4096,w0,o0,bl0)
	 skmem:(r0,rb366080,t0,tb46080,f4096,w0,o136,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb978736,t0,tb46080,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f0,w0,o0,bl0)
	 skmem:(r0,rb1061488,t0,tb2626560,f4096,w0,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
	 skmem:(r0,rb369280,t0,tb87040,f0,w0,o0,bl0)
	 skmem:(r0,rb87380,t0,tb16384,f2816,w1280,o0,bl0)
```
比如想要提取每一个数字。
则如下所示：
```c
# ss -tman | grep skmem | sed -r "s/skmem:\(r([0-9]+),rb([0-9]+),t([0-9]+),tb([0-9]+),f([0-9]+),w([0-9]+),o([0-9]+),bl([0-9]+)\)/\1  \2  \3  \4  \5  \6   \7  \8/g"
```
![](attachments/Pasted%20image%2020231116195238.png)

```c
# echo "libgcc-4.8.5-4.h5.x86_64.rpm" | sed -r "s/libgcc-([0-9]+\.[0-9]+.*)\.rpm/\1/g"
4.8.5-4.h5.x86_64

如果是使用grep，则如下所示：
# echo "libgcc-4.8.5-4.h5.x86_64.rpm" | grep -Eo "[0-9]+\.[0-9]+.*x86_64"
4.8.5-4.h5.x86_64
```
```c
grep参数说明：

   -E, --extended-regexp
          Interpret PATTERN as an extended regular expression (ERE, see below).  (-E is specified by POSIX.)

   -o, --only-matching
          Print only the matched (non-empty) parts of a matching line, with each such part on a separate output line.

   -e PATTERN, --regexp=PATTERN
          Use PATTERN as the pattern.  This can be used to specify multiple search patterns, or to protect a pattern beginning with a hyphen (-).  (-e is specified by POSIX.)
```
## 删除
### 删除指定范围的行
```bash
for a in `seq 25 39`; do sed -i '18,145d' 192.22.2.${a}_tcp_8080.conf; done
```
### 删除匹配指定字符串的行
```c
sed -i '/hello/d' aaaa
sed -i '/^hello/d' aaaa
```
## 添加
### 指定字符串之后行添加
- 在指定字符串的下一行插入一行
```c
sed -i '/snat_log_enable/a\        snat_log_stop 1' /etc/dpvs.conf
或者
sed -i "/snat_log_enable/a\\        snat_log_stop 1" /etc/dpvs.conf
```

- 在指定字符串的下面插入多行
```bash
比如，要在keepalived.conf中的 “cluster_tag test-fnat“ 字符串后面插入：
        vxlan {
            vni 1
            local_vtep_ip 192.22.2.100
            local_vtep_mac 0C:42:A1:90:F4:43
            vtep_ip 192.20.144.100
            vtep_port 4789
            vtep_mac 0c:42:a1:78:4f:a1
        }


# sed -i "/cluster_tag test-fnat/a\\        vxlan {\n            vni 1\n            local_vtep_ip 192.22.2.100\n            local_vtep_mac 0C:42:A1:90:F4:43\n            vtep_ip 192.20.144.100\n            vtep_port 4789\n            vtep_mac 0c:42:a1:78:4f:a1\n        }\n" keepalived.conf
```

### 注释指定行
```c
文件中，不是以#开头的行，在开头添加#
sed -i "s/^[^#].*$/#&/g" ${outFile}
```
![](attachments/Pasted%20image%2020230720123333.png)
## 替换
### 替换为常量
```c
ls *tcp_8080.conf | xargs sed -i 's/192.20.1/192.20.45/g'
```
### 替换为变量
如下所示，变量用'${xx}' 。
```c
for a in `seq 25 39`; do sed -i 's/192.22.2.'${a}'_tcp_80/192.22.2.'${a}'_tcp_8080/g' 192.22.2.${a}_tcp_8080.conf; done
```
# 参考
