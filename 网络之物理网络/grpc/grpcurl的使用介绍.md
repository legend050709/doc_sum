```table-of-contents
```
# 背景
一般情况下测试 gRPC 服务，都是通过客户端来直接请求服务端。
如果客户端还没准备好的话，也可以使用 BloomRPC 这样的 GUI 客户端。 如果环境不支持安装这种 GUI 客户端的话，那么有没有一种工具，类似于 curl 这样的，直接通过终端，在命令行发起请求呢？ 答案肯定是有的，就是本文要介绍的 grpcurl。

# 介绍
`grpcurl` 是一个能够直接与 gRPC 服务交互的命令行工具。基本算是 `curl` 的 gRPC 版本。
由于 gRPC 服务之间的通信使用的是 protocol buffers（下文 PB 指代） 格式的二进制编码，所以无法使用传统的 curl，更何况旧版本的 curl 还不支持 HTTP/2。

## 原理
为了更好上手，该工具和服务器交互时，我们只需要提供 **JSON 数据作为请求数据**即可，该工具底层会**自动将其编码为 PB 格式的二进制与服务端进行交互**。


该工具支持通过以下几种情况查看 gPRC service 的定义格式（schema）：

- 通过 反射服务进行查询(需要服务器开启反射服务)
- 通过 proto 源文件（服务器没有开启反射服务时可以考虑）
- 通过编译完成的 protoset文件

只有通过使用上述方式查询得到的 schema，该工具才能能够将 JSON 请求数据准确的转换成 PB 格式的二进制数据。

# 特点
1. grpcurl 支持所有 gRPC 的方法，包括 stream 方法。通过 grpcurl 你甚至可以与服务端进行双向的 stram 交流。
2. grpcurl 支持 plain-text(HTTP/2) 及 TLS, 对于 TLS 有大量的可选项配置，同时支持双向 TLS 即当客户端被要求提交证书也是支持的。
3. grpcurl 支持通过反射服务无缝连接，又或者使用 proto 或则 protoset 文件。

Protobuf 本身具有反射功能，可以在运行时获取对象的 Proto 文件。gRPC 同样也提供了一个名为 reflection 的反射包，用于为 gRPC 服务提供查询。

# 安装

## 方式一：下载go包编译
https://github.com/fullstorydev/grpcurl/issues/154

```go
yum install -y golang 
export GOPROXY=https://mirrors.aliyun.com/goproxy/

go get github.com/fullstorydev/grpcurl

go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest
```

结果：在 $GOPATH/bin 目录下，生成一个 grpcurl 可执行文件。我们可以复制到 /usr/local/bin/ 下。

## 方式二：直接下载可执行文件
```bash
wget https://github.com/fullstorydev/grpcurl/releases/download/v1.7.0/grpcurl_1.7.0_linux_x86_64.tar.gz

tar -xvf grpcurl_1.7.0_linux_x86_64.tar.gz

chmod +x grpcurl

./grpcurl -help
```

# 使用
```bash
% grpcurl --help
```
-`plaintex`t ：在使用grpcurl时，需要通过`-cert`和`-key`参数设置公钥和私钥文件，连接启用了`tls`协议的服务。
对于没用tls协议的grpc服务，通过`-plaintext`参数忽略`tls`证书的验证过程。
如果是`Unix Socket`协议，则需要指定`-unix`参数。

 `-d` 参数设置的是方法参数，默认是`json`格式。
 如果 `-d` 参数是 `@` 则表示从标准输入读取 json 输入参数
 
 `-proto value` :
```c
The name of a proto source file. Source files given will be used to determine the RPC schema instead of querying for it from the remote server via the gRPC reflection API. When set: the ‘list’ action lists the services found in the given files and their imports (vs. those exposed by the remote server), and the ‘describe’ action describes symbols found in the given files. May specify more than one via multiple -proto flags. Imports will be resolved using the given -import-path flags. Multiple proto files can be specified by specifying multiple -proto flags. It is an error to use both -protoset and -proto flags.
```

`-import-path value `:
```c
The path to a directory from which proto sources can be imported, for use with -proto flags. Multiple import paths can be configured by specifying multiple -import-path flags. Paths will be searched in the order given. If no import paths are given, all files (including all imports) must be provided as -proto flags, and grpcurl will attempt to resolve all import statements from the set of file names given.
```
# 范例
## 查看指定项目提供的所有服务(service)
假设服务端支持如下服务：
```proto
syntax = "proto3";

package grpc.examples.echo;

option go_package = "google.golang.org/grpc/examples/features/proto/echo";

// EchoRequest is the request for echo.
message EchoRequest {
  string message = 1;
}

// EchoResponse is the response for echo.
message EchoResponse {
  string message = 1;
}

// Echo is the echo service.
service Echo {
  // UnaryEcho is unary echo.
  rpc UnaryEcho(EchoRequest) returns (EchoResponse) {}
  // ServerStreamingEcho is server side streaming.
  rpc ServerStreamingEcho(EchoRequest) returns (stream EchoResponse) {}
  // ClientStreamingEcho is client side streaming.
  rpc ClientStreamingEcho(stream EchoRequest) returns (EchoResponse) {}
  // BidirectionalStreamingEcho is bidi streaming.
  rpc BidirectionalStreamingEcho(stream EchoRequest) returns (stream EchoResponse) {}
}
```
### 反射方式
成功运行 服务端代码 后（默认运行在了 50051 端口），通过下述命令可以查看其支持的所有服务：
```bash
% grpcurl -plaintext 127.0.0.1:50051 list                                 
grpc.examples.echo.Echo
grpc.reflection.v1alpha.ServerReflection

注：此中为 -plaintext，而不是 --plaintext，因为go程序的参数使用一个`-`即可。
```

即上面的命令我们通过 HTTP/2 的方式与 gRPC 服务端进行了一次交互，由于服务端使用了反射服务，可以查询到其支持的所有的 service，其中就包括前面定义的 proto 内容 `grpc.examples.echo.Echo` 。

### proto文件方式
亦或者我们可以直接加载 proto 文件的方式查看所支持的服务：
```bash
% grpcurl -import-path echo -proto echo.proto list
grpc.examples.echo.Echo
```
-import-path + -proto 指定具体的proto文件的位置。

## 查看指定服务(service)支持的方法(method)
在前面使用 list 的基础上，我们还可以进一步查询指定服务所支持的方法：
### 反射方式
```bash
% grpcurl -plaintext 127.0.0.1:50051 list grpc.examples.echo.Echo
# 查看具体的某个 servier的 方法列表

grpc.examples.echo.Echo.BidirectionalStreamingEcho
grpc.examples.echo.Echo.ClientStreamingEcho
grpc.examples.echo.Echo.ServerStreamingEcho
grpc.examples.echo.Echo.UnaryEcho
```
#### proto文件方式
同理，使用 proto 文件方式也可以得到一样的输出:
```bash
% grpcurl -import-path ./echo -proto echo.proto list grpc.examples.echo.Echo

grpc.examples.echo.Echo.BidirectionalStreamingEcho
grpc.examples.echo.Echo.ClientStreamingEcho
grpc.examples.echo.Echo.ServerStreamingEcho
grpc.examples.echo.Echo.UnaryEcho
```

## 获取指定服务(service)的描述信息
前面两个案例我们使用 list 只是查看到了相应的服务名，如果需要查看更具体的描述则需要通过 describe。
### 反射方式
```bash
% grpcurl -plaintext 127.0.0.1:50051 describe grpc.examples.echo.Echo     
grpc.examples.echo.Echo is a service:
service Echo {
  rpc BidirectionalStreamingEcho ( 
  	stream .grpc.examples.echo.EchoRequest ) returns ( 
  	stream .grpc.examples.echo.EchoResponse );

  rpc ClientStreamingEcho ( 
  	stream .grpc.examples.echo.EchoRequest ) returns ( 
  	.grpc.examples.echo.EchoResponse );

  rpc ServerStreamingEcho ( 
  	.grpc.examples.echo.EchoRequest ) returns ( 
  	stream .grpc.examples.echo.EchoResponse );

  rpc UnaryEcho ( 
  	.grpc.examples.echo.EchoRequest ) returns ( 
  	.grpc.examples.echo.EchoResponse );
}
```
注：grpc.examples.echo 为 package, Echo 为 Service。如果需要查看具体的method， 则可以考虑在后面继续加上method的名称即可。

### proto文件方式
同理，使用加载 proto 文件方式也可以得到一样的输出:
```bash
% grpcurl -import-path ./echo -proto echo.proto describe grpc.examples.echo.Echo
grpc.examples.echo.Echo is a service:
// Echo is the echo service.
service Echo {
  // BidirectionalStreamingEcho is bidi streaming.
  rpc BidirectionalStreamingEcho ( 
  	stream .grpc.examples.echo.EchoRequest ) returns ( 
  	stream .grpc.examples.echo.EchoResponse );

  // ClientStreamingEcho is client side streaming.
  rpc ClientStreamingEcho ( 
  	stream .grpc.examples.echo.EchoRequest ) returns ( 
  	.grpc.examples.echo.EchoResponse );

  // ServerStreamingEcho is server side streaming.
  rpc ServerStreamingEcho ( 
  	.grpc.examples.echo.EchoRequest ) returns ( 
  	stream .grpc.examples.echo.EchoResponse );

  // UnaryEcho is unary echo.
  rpc UnaryEcho ( 
  	.grpc.examples.echo.EchoRequest ) returns ( 
  	.grpc.examples.echo.EchoResponse );
}
```
## 获取指定message 信息
当然除了输出服务（service）的描述信息外，也可以输出 message 描述信息：
```bash
% grpcurl -plaintext 127.0.0.1:50051 describe grpc.examples.echo.EchoRequest
grpc.examples.echo.EchoRequest is a message:
message EchoRequest {
  string message = 1;
}
```
```bash
% grpcurl -plaintext 127.0.0.1:50051 describe grpc.examples.echo.EchoResponse
grpc.examples.echo.EchoResponse is a message:
message EchoResponse {
  string message = 1;
}
```

## 与 gRPC 服务端进行交互
服务端部分交互代码如下：
```go
type ecServer struct { // Echo 服务的实现
	ecpb.UnimplementedEchoServer
}

func (s *ecServer) UnaryEcho(
	ctx context.Context, req *ecpb.EchoRequest) (*ecpb.EchoResponse, 
	error) {
	return &ecpb.EchoResponse{Message: req.Message}, nil
}
```
通过 `-d` 参数发送 JSON 数据
```bash
 % grpcurl -plaintext -d '{"message":"hi"}' 127.0.0.1:50051 grpc.examples.echo.Echo.UnaryEcho
{
  "message": "hi"
}
```
# 问题
## 服务没有注册反射服务
如果 grpc 服务正常，但是服务没有启动 reflection 反射服务，将会遇到以下错误：如下所示：
```c
Failed to list services: server does not support the reflection API
```

### 服务器注射反射服务
go 代码范例如下所示：
```go
/**
 * @Author: zhangsan
 * @Description:
 * @File:  main
 * @Version: 1.0.0
 * @Date: 2021/5/10 下午5:04
 */

package main

import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/reflection"
    "log"
    "net"
    servers "test/grpcurl/server"
    server "test/grpcurl/src"
)

func main(){
    grpcServe := grpc.NewServer()

    //注册grpcurl反射服务，否则报错
    reflection.Register(grpcServe)

    server.RegisterServerServer(grpcServe,servers.NewHelloServices())

    listen,e := net.Listen("tcp","127.0.0.1:8080")
    if e != nil{
        log.Fatal(e)
    }

    grpcServe.Serve(listen)
}
```

### 本地通过proto文件访问
在本地将服务器的Proto文件(echo.proto)搞过来。然后通过下面的方法进行方法：
```c
grpcurl -proto echo.proto  -plaintext -d '{"message":"hi"}' 127.0.0.1:50051 grpc.examples.echo.Echo/UnaryEcho

or 


grpcurl -proto echo.proto  -plaintext -d '{"message":"hi"}' 127.0.0.1:50051 grpc.examples.echo.Echo.UnaryEcho

注：grpc.examples.echo 为 package, Echo 为 service， UnaryEcho 为 Method。
```
## 参数格式错误
报错信息：
```c
Error invoking method "greet.Greeter/SayHello": error getting request data: invalid character 'n' looking for beginning of object key string
```
**解决：**
`-d` 后面参数为 json 格式，并且需要使用 `''` 包裹起来。
## gRPC Server 未启用 TLS
如果没有配置好公钥和私钥文件，也没有忽略证书的验证过程，那么将会遇到类似以下的错误：
```c
Failed to dial target host "127.0.0.1:50051": tls: first record does not look like a TLS handshake
```
**解决：**
请求时增加参数：`-plaintext`，参考上面的命令。

# 参考
```c
https://tangzixiang.github.io/posts/2020/02/grpcurl-%E4%BD%BF%E7%94%A8%E6%8C%87%E5%8D%97/

```