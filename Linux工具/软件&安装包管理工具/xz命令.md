# 简介
**xz命令** XZ Utils 是为 POSIX 平台开发具有高压缩率的工具。它使用 LZMA2 压缩算法，生成的压缩文件比 POSIX 平台传统使用的 gzip、bzip2 生成的压缩文件更小，而且解压缩速度也很快。

以.xz格式压缩或解压缩文件。
![](attachments/Pasted%20image%2020230615123315.png)

xz命令可以压缩、解压缩单个文件，也可以对tar的文件进行压缩、解压缩。
压缩之后，一般文件后会有一个.xz的后缀，比如常见的 tar.xz。

# 使用
![](attachments/Pasted%20image%2020230615123926.png)

```css
长选项的强制参数对短选项也是强制的。

  -z, --compress      强制压缩
  -d, --decompress    强制解压
  -t, --test          测试压缩文件完整性
  -l, --list          列出有关文件的信息
  -k, --keep          保留（不删除）输入文件
  -f, --force         强制覆盖输出文件和（取消）压缩链接
  -c, --stdout        写入标准输出，不删除输入文件
  -0 .. -9            压缩预设；0-2快速压缩，3-5良好
                      压缩，6-9极好的压缩；默认值为6
  -e, --extreme       编码时使用更多的CPU时间来增加压缩
                      不增加解码器内存使用率的比率
  -q, --quiet         取消警告；指定两次也可以取消错误
  -v, --verbose       详细；为更详细的内容指定两次
  -h, --help          显示此简短帮助
  -H, --long-help     显示长帮助（同时列出高级选项）
  -V, --version       显示版本号
```
# 范例

- 压缩一个文件 test.txt
![](attachments/Pasted%20image%2020230615123822.png)

注：没有添加-k标记，压缩成功后生成 test.txt.xz, 原文件会被删除。

- 展示xz文件内容
使用参数 -l 显示 .xz 文件的基本信息。基本信息包括压缩率、数据完整性验证方式等。也可以和参数 -v 或 -vv 配合显示更详尽的信息。

```shell
xz -l index.txt.xz
# Strms  Blocks   Compressed Uncompressed  Ratio  Check   Filename
#    1       1        768 B      1,240 B  0.619  CRC64   index.txt.
```