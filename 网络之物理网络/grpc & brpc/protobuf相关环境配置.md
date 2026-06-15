```table-of-contents
```
# protobuf 和 protobuf-c
## 简介
### protobuf 简介
ProtoBuf （Google Protocol Buffer）是由google公司用于数据交换的序列结构化数据格式，具有跨平台、跨语言、可扩展特性，类型于常用的XML及JSON，但具有更小的传输体积、更高的编码、解码能力，特别适合于数据存储、网络数据传输等对存储体积、实时性要求高的领域。以 .proto为后缀，有自己独立的编译器。

优点：空间效率和时间效率高，对于数据大小敏感，传输效率高的
缺点：消息结构可读性不高，序列化后的字节序列为二进制序列不能简单的分析有效性；

###  protobuf-c 简介
官方的 `Protocol Buffer`提供了C++、C#、Dart、Go、Java、Kotlin和Python语言的支持。但是不包括C语言。
(1)`protoc`命令通过 `.proto`文件生成支持语言的代码。
(2) `protobuf`不同语言的库用于代码最终调用时使用。

**`对于 C语言版本的protobuf-c，只针对C语言做了实现。`**
（1）`protoc-c`命令通过 `*.proto`文件生成对应 C语言的代码（`.pb-c.h`和`.pb-c.c`文件），以便在C语言中使用。
（2）`libprotobuf-c`库用于编译时连接使用。


## 安装
### 范例一
#### 安装 protobuf

protobuf源码包下载路径：[https://github.com/protocolbuffers/protobuf/releases](https://github.com/protocolbuffers/protobuf/releases "https://github.com/protocolbuffers/protobuf/releases")

![](attachments/Pasted%20image%2020250819141618.png)

当前最新的版本为3.19.4，这里我就选择下载最新版本的包，选择.tar.gz包下载，其他方式亦可。

```bash

# 下载，此中下载的是3.20.3.tar.gz
wget https://github.com/protocolbuffers/protobuf/releases/download/v3.20.3/protobuf-all-3.20.3.tar.gz

#1.解压tar.gz包
tar zxvf protobuf-3.19.4.tar.gz
 
#2.进入解压后的目录完成编译和安装
cd protobuf-3.19.4
./configure --prefix=/usr/local/protobuf-3.19.4
make
sudo make install
 
#3.配置环境变量———即指定protobuf编译工具的路径
cd ~
vim .bashrc
#在.bashrc脚本中添加下面两行
export PATH="$PATH:/usr/local/protobuf-3.19.4/bin"
export PKG_CONFIG_PATH=/usr/local/protobuf-3.19.4/lib/pkgconfig
#保存并退出vim
 
#4.使对.bashrc文件的修改立即生效
source .bashrc
 
#5.指定protobuf动态库的路径
sudo vim /etc/ld.so.conf
#添加下面一行
/usr/local/protobuf/protobuf-3.19.4/lib
 
#6.使上面新指定的链接库立即生效
sudo ldconfig
```
#### 安装protobuf-c

protobuf-c源码包下载路径：[https://github.com/protobuf-c/protobuf-c](https://github.com/protobuf-c/protobuf-c "https://github.com/protobuf-c/protobuf-c")

![](attachments/Pasted%20image%2020250819142308.png)

当前最新版本为v1.4.0，我之前已经下好了v1.3.3版本，所以这直接就用v1.3.3版本好了

```bash
(1) 查看移除旧的
# yum list installed |grep protobuf
protobuf.x86_64                  2.5.0-8.el7              @base
protobuf-compiler.x86_64         2.5.0-8.el7              @base

# rpm -ql protobuf
/usr/lib64/libprotobuf.so.8
/usr/lib64/libprotobuf.so.8.0.0
/usr/share/doc/protobuf-2.5.0
/usr/share/doc/protobuf-2.5.0/CHANGES.txt
/usr/share/doc/protobuf-2.5.0/CONTRIBUTORS.txt
/usr/share/doc/protobuf-2.5.0/COPYING.txt
/usr/share/doc/protobuf-2.5.0/README.txt

# rpm -ql protobuf-compiler
/usr/bin/protoc
/usr/lib64/libprotoc.so.8
/usr/lib64/libprotoc.so.8.0.0
/usr/share/doc/protobuf-compiler-2.5.0
/usr/share/doc/protobuf-compiler-2.5.0/COPYING.txt
/usr/share/doc/protobuf-compiler-2.5.0/README.txt

# yum remove -y protobuf-compiler protobuf protobuf-c-devel protobuf-c

#（2）下载新的
# 下载，此中下载的是1.5.0
wget https://github.com/protobuf-c/protobuf-c/releases/download/v1.5.0/protobuf-c-1.5.0.tar.gz

#1.解压.tar.gz源码包
tar zxf protobuf-c-1.3.3.tar.gz


#2.编译并安装
cd protobuf-c-1.3.3/
./configure --prefix=/usr/local/protobuf-c-1.3.3
make
sudo make install
 
#3.配置环境变量———即指定protobuf-c编译工具的路径
cd ~
vim .bashrc
 
#添加下面一行
export PATH="$PATH:/usr/protobuf-c-1.3.3/bin"
 
 
#4.使对.bashrc文件的修改立即生效
source .bashrc
 
#5.指定protobuf动态库的路径
sudo vim /etc/ld.so.conf
#添加下面一行
/usr/local/protobuf/protobuf-c-1.3.3/lib
 
#6.使上面新指定的链接库立即生效
sudo ldconfig
```

#### 安装完成确认

可以通过查看编译工具的版本来确认是否安装成功，如下便确认已经安装成功

![](attachments/Pasted%20image%2020250819142344.png)


### 范例二
#### 安装 protobuf
```bash
# 1. 编译安装protobuf
[root@centos7 protobuf]# cd /usr/local/protobuf/protobuf-3.19.6
[root@centos7 protobuf-3.19.6]# ./configure --prefix=/usr/local/protobuf/protobuf-3.19.6/
[root@centos7 protobuf-3.19.6]# make
[root@centos7 protobuf-3.19.6]# make install

# 2. 添加环境变量
[root@centos7 protobuf-3.19.6]# cd ~
[root@centos7 ~]# vim .bashrc
# 添加这两行
export PATH="$PATH:/usr/local/protobuf/protobuf-3.19.6/bin"
export PKG_CONFIG_PATH=/usr/local/protobuf/protobuf-3.19.6/lib/pkgconfig
# 使之生效
[root@centos7 ~]# source .bashrc

# 3. 检查是否安装成功，查看版本信息。
[root@centos7 ~]# protoc --version
libprotoc 3.19.6

```

`make install`之后，查看输出：
```bash
# tree /usr/local/protobuf/protobuf-3.20.3/
/usr/local/protobuf/protobuf-3.20.3/
├── bin
│   └── protoc
├── include
│   └── google
│       └── protobuf
│           ├── any.h
│           ├── any.pb.h
│           ├── any.proto
│           ├── api.pb.h
│           ├── api.proto
│           ├── arena.h
│           ├── arena_impl.h
│           ├── arenastring.h
│           ├── arenaz_sampler.h
│           ├── compiler
│           │   ├── code_generator.h
│           │   ├── command_line_interface.h
│           │   ├── cpp
│           │   │   ├── cpp_file.h
│           │   │   ├── cpp_generator.h
│           │   │   ├── cpp_helpers.h
│           │   │   └── cpp_names.h
│           │   ├── csharp
│           │   │   ├── csharp_doc_comment.h
│           │   │   ├── csharp_generator.h
│           │   │   ├── csharp_names.h
│           │   │   └── csharp_options.h
│           │   ├── importer.h
│           │   ├── java
│           │   │   ├── java_generator.h
│           │   │   ├── java_kotlin_generator.h
│           │   │   └── java_names.h
│           │   ├── js
│           │   │   └── js_generator.h
│           │   ├── objectivec
│           │   │   ├── objectivec_generator.h
│           │   │   └── objectivec_helpers.h
│           │   ├── parser.h
│           │   ├── php
│           │   │   └── php_generator.h
│           │   ├── plugin.h
│           │   ├── plugin.pb.h
│           │   ├── plugin.proto
│           │   ├── python
│           │   │   ├── python_generator.h
│           │   │   └── python_pyi_generator.h
│           │   └── ruby
│           │       └── ruby_generator.h
│           ├── descriptor_database.h
│           ├── descriptor.h
│           ├── descriptor.pb.h
│           ├── descriptor.proto
│           ├── duration.pb.h
│           ├── duration.proto
│           ├── dynamic_message.h
│           ├── empty.pb.h
│           ├── empty.proto
│           ├── explicitly_constructed.h
│           ├── extension_set.h
│           ├── extension_set_inl.h
│           ├── field_access_listener.h
│           ├── field_mask.pb.h
│           ├── field_mask.proto
│           ├── generated_enum_reflection.h
│           ├── generated_enum_util.h
│           ├── generated_message_bases.h
│           ├── generated_message_reflection.h
│           ├── generated_message_tctable_decl.h
│           ├── generated_message_tctable_impl.h
│           ├── generated_message_util.h
│           ├── has_bits.h
│           ├── implicit_weak_message.h
│           ├── inlined_string_field.h
│           ├── io
│           │   ├── coded_stream.h
│           │   ├── gzip_stream.h
│           │   ├── io_win32.h
│           │   ├── printer.h
│           │   ├── strtod.h
│           │   ├── tokenizer.h
│           │   ├── zero_copy_stream.h
│           │   ├── zero_copy_stream_impl.h
│           │   └── zero_copy_stream_impl_lite.h
│           ├── map_entry.h
│           ├── map_entry_lite.h
│           ├── map_field.h
│           ├── map_field_inl.h
│           ├── map_field_lite.h
│           ├── map.h
│           ├── map_type_handler.h
│           ├── message.h
│           ├── message_lite.h
│           ├── metadata.h
│           ├── metadata_lite.h
│           ├── parse_context.h
│           ├── port_def.inc
│           ├── port.h
│           ├── port_undef.inc
│           ├── reflection.h
│           ├── reflection_ops.h
│           ├── repeated_field.h
│           ├── repeated_ptr_field.h
│           ├── service.h
│           ├── source_context.pb.h
│           ├── source_context.proto
│           ├── struct.pb.h
│           ├── struct.proto
│           ├── stubs
│           │   ├── bytestream.h
│           │   ├── callback.h
│           │   ├── casts.h
│           │   ├── common.h
│           │   ├── hash.h
│           │   ├── logging.h
│           │   ├── macros.h
│           │   ├── map_util.h
│           │   ├── mutex.h
│           │   ├── once.h
│           │   ├── platform_macros.h
│           │   ├── port.h
│           │   ├── status.h
│           │   ├── stl_util.h
│           │   ├── stringpiece.h
│           │   ├── strutil.h
│           │   └── template_util.h
│           ├── text_format.h
│           ├── timestamp.pb.h
│           ├── timestamp.proto
│           ├── type.pb.h
│           ├── type.proto
│           ├── unknown_field_set.h
│           ├── util
│           │   ├── delimited_message_util.h
│           │   ├── field_comparator.h
│           │   ├── field_mask_util.h
│           │   ├── json_util.h
│           │   ├── message_differencer.h
│           │   ├── time_util.h
│           │   ├── type_resolver.h
│           │   └── type_resolver_util.h
│           ├── wire_format.h
│           ├── wire_format_lite.h
│           ├── wrappers.pb.h
│           └── wrappers.proto
└── lib
    ├── libprotobuf.a
    ├── libprotobuf.la
    ├── libprotobuf-lite.a
    ├── libprotobuf-lite.la
    ├── libprotobuf-lite.so -> libprotobuf-lite.so.31.0.3
    ├── libprotobuf-lite.so.31 -> libprotobuf-lite.so.31.0.3
    ├── libprotobuf-lite.so.31.0.3
    ├── libprotobuf.so -> libprotobuf.so.31.0.3
    ├── libprotobuf.so.31 -> libprotobuf.so.31.0.3
    ├── libprotobuf.so.31.0.3
    ├── libprotoc.a
    ├── libprotoc.la
    ├── libprotoc.so -> libprotoc.so.31.0.3
    ├── libprotoc.so.31 -> libprotoc.so.31.0.3
    ├── libprotoc.so.31.0.3
    └── pkgconfig
        ├── protobuf-lite.pc
        └── protobuf.pc
```



#### 安装 protobuf-c
```bash
# 1. 编译安装protobuf-c
[root@centos7 ~]# cd /usr/local/protobuf/protobuf-c-1.4.1
[root@centos7 protobuf-c-1.4.1]# ./configure --prefix=/usr/local/protobuf/protobuf-c-1.4.1
[root@centos7 protobuf-c-1.4.1]# make
[root@centos7 protobuf-c-1.4.1]# make install

# 2. 添加环境变量
[root@centos7 protobuf-c-1.4.1]# cd ~
[root@centos7 ~]# vim .bashrc
# 添加这一行
export PATH="$PATH:/usr/local/protobuf/protobuf-c-1.4.1/bin"
# 使之生效
[root@centos7 ~]# source .bashrc

# 3. 检查是否安装成功，查看版本信息。
[root@centos7 ~]# protoc-c --version
protobuf-c 1.4.1
libprotoc 3.19.6

```

`make install`之后，查看输出：
```bash
# tree /usr/local/protobuf/protobuf-c-1.5.0
/usr/local/protobuf/protobuf-c-1.5.0
├── bin
│   ├── protoc-c -> protoc-gen-c
│   └── protoc-gen-c
├── include
│   ├── google
│   │   └── protobuf-c
│   │       └── protobuf-c.h -> ../../protobuf-c/protobuf-c.h
│   └── protobuf-c
│       ├── protobuf-c.h
│       └── protobuf-c.proto
└── lib
    ├── libprotobuf-c.a
    ├── libprotobuf-c.la
    ├── libprotobuf-c.so -> libprotobuf-c.so.1.0.0
    ├── libprotobuf-c.so.1 -> libprotobuf-c.so.1.0.0
    ├── libprotobuf-c.so.1.0.0
    └── pkgconfig
        └── libprotobuf-c.pc
```

### 安装问题
有可能安装过程中，需要

# 参考
```bash
# Linux下protobuf和 protobuf-c安装使用
https://blog.csdn.net/qq_42402854/article/details/134085587
```