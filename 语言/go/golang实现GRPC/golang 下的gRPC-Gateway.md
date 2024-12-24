```table-of-contents
```
# 背景
由于公司微服务架构是采用 grpc, 但是在测试 API 方面相当麻烦。
因为测试的话，如果可以curl 请求某个服务或者下发配置的话，是比较方便的，grpc API一般是难以构造的。

# 目标

服务端同时开启gRPC和HTTP服务，网关负责接收客户端请求，然后决定直接转发给grpc服务还是转给http服务。
如果服务器接收的是客户端的HTTP请求，需要转换为grpc请求，获取Grpc响应后再转为`json`数据返回给Http的客户端。

# 问题

etcd v3 改用 gRPC 后为了兼容原来的 API，同时要提供 HTTP/JSON 方式的API。
服务端同时开启gRPC和HTTP服务。为了满足这个需求：
1》要么开发两套 API。
即一套 Http server，一套是 Grpc server。2个服务监听不同的端口（防止端口的监听的冲突）。但是实现2套Server，不是很优雅。
2》要么实现一种转换机制。
只监听一个端口。比如：同时在本机的`8091`端口对外提供gRPC API和HTTP API服务。

他们选择了后者，而我们选择跟随他们的脚步。


# 实现一：使用 gRPC-Gateway

参考：[grpc-ecosystem/grpc-gateway](https://github.com/grpc-ecosystem/grpc-gateway)

## protoc插件：protoc-gen-grpc-gateway
除了前面介绍的 基于`pb`文件生成`go`代码的的`protoc`的插件 `protoc-gen-go` 和 `protoc-gen-go-grpc` 之外；为了将`http`的请求转换为`grpc`的请求，还需要`protoc-gen-grpc-gateway`插件。



## 介绍

```text
The grpc-gateway is a plugin of the Google protocol buffers compiler [protoc](https://github.com/protocolbuffers/protobuf). It reads protobuf service definitions and generates a reverse-proxy server which 'translates a RESTful HTTP API into gRPC. This server is generated according to the `google.api.http` annotations in your service definitions.

This helps you provide your APIs in both gRPC and RESTful style at the same time.
```

`grpc-gateway`是`protoc`的一个插件。它读取 pb 文件中的 服务（service）定义，并生成一个反向代理服务器（reverse proxy），将`RESTful HTTP  API`转换为`gRPC`。此服务器是根据`gRPC`定义中的自定义选项生成的。


## 原理

![](attachments/Pasted%20image%2020241222162256.png)

### 具体流程
如下所示：
1》当 HTTP 请求到达 gRPC-Gateway 时，它将 JSON 数据解析为 Protobuf 消息。
2》然后，它使用解析的 Protobuf 消息发出正常的 Go gRPC 客户端请求。
Go gRPC 客户端将 Protobuf 结构编码为 Protobuf 二进制格式，然后将其发送到 gRPC 服务器。

3》gRPC 服务器处理请求并以 Protobuf 二进制格式返回响应。
4》Go gRPC 客户端将其解析为 Protobuf 消息，并将其返回到 gRPC-Gateway，后者将 Protobuf 消息编码为 JSON 并将其返回给原始客户端。


### 小结

`gRPC-Gateway` 相对于是 ==`Http Server` + `Grpc Client`的封装==。 
对于来自于 `Clien`t的 `Http` 请求而言，其充当 `Http Server`的角色。
再将 `HTTP`请求中的 `JSON` 数据解析为 `Protobuf` 结构，
作为 `GRPC Client` 发送 `Protobuf` 二进制格式（ Protobuf 结构编码得到的）。
`GRPC Client`的请求，被`GRPC Server`处理之后，以`Protobuf` 二进制格式返回响应。
 `gRPC Client`将其解析为 `Protobuf` 消息，并将其返回到 `gRPC-Gateway`，后者将 `Protobuf` 消息编码为 `JSON` 并将其返回给原始客户端。

## 安装
```go

1> 先安装protoc

2> 然后安装 protc-gen-go, protc-gen-go-grpc, protoc-gen-grpc-gateway 等插件：
go install \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway@v1.16.0 \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2 \
    google.golang.org/protobuf/cmd/protoc-gen-go \
    google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

安装完成之后，检查：
- 1）执行 protoc –version 能打印出版本信息；
- 2）$GOPATH/bin 目录下有 `protoc-gen-go`、`protoc-gen-go-grpc`、`protoc-gen-grpc-gateway` 这三个可执行文件。

## 实现流程
### 整体流程
**以 .proto 文件为基础，编写插件对 protoc 进行扩展，编译出go语言不同模块的源文件。**

- 1）首先定义 .proto 文件；
- 2）然后由 protoc 将 .proto 文件编译成 protobuf 格式的数据；
- 3）将 2 中编译后的数据传递到各个插件，生成对应语言、对应模块的源代码。
    - Go Plugins 用于生成 .pb.go 文件
    - gRPC Plugins 用于生成 _grpc.pb.go
    - gRPC-Gateway 则是 pb.gw.go

其中步骤2和3是一起的，只需要在 protoc 编译时传递不同参数即可。比如以下命令会同时生成 Go、gRPC 、gRPC-Gateway 需要的 3 个文件。
```go
protoc --go_out . --go-grpc_out . --grpc-gateway_out . hello_world.proto
```

### gRPC 部分
**（1）目录结构**：
```bash
proto
├── google
│   └── api
│       └── http.proto
└── helloworld
    └── hello_world.proto
```

**(2)创建.proto文件**
创建一个 `hello_world.proto` 文件，内容如下
```go
syntax = "proto3";
option go_package=".;proto";
package helloworld;

// The greeting service definition
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

```

**(3)生成 Go subs**
使用 protoc 编译生成不同模块的源文件，具体命令如下:
```go
lixd@17x:~/17x/projects/grpc-go-example/features$ protoc --proto_path=./proto \
    --go_out=./proto --go_opt=paths=source_relative \
    --go-grpc_out=./proto --go-grpc_opt=paths=source_relative \
   ./proto/helloworld/hello_world.proto

```
会生成 `*.pb.go` 和 `*_grpc.pb.go` 两个文件。


**(4)Server**
`main.go`内容如下：
```go
package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"net"

	pb "github.com/lixd/grpc-go-example/features/proto/helloworld"
	"google.golang.org/grpc"
)
var port = flag.Int("port", 50051, "the port to serve on")

type server struct {
	pb.UnimplementedGreeterServer
}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	return &pb.HelloReply{Message: "hello " + in.Name }, nil
}

func main() {
	// Create a gRPC server object
	s := grpc.NewServer()
	// Attach the Greeter service to the server
	pb.RegisterGreeterServer(s, &server{})
	// Serve gRPC Server
	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", *port))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	log.Println("Serving gRPC on 0.0.0.0" + fmt.Sprintf(":%d", *port))
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}

```

**(5) Client**
`main.go`内容如下:
```go
package main

import (
	"log"

	"golang.org/x/net/context"
	"google.golang.org/grpc"
	pb "i-go/grpc/gateway/proto/helloworld"
)

const (
	address     = "localhost:8080"
	defaultName = "world"
)

func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure(), grpc.WithBlock())
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	c := pb.NewGreeterClient(conn)
	r, err := c.SayHello(context.Background(), &pb.HelloRequest{Name: defaultName})
	if err != nil {
		log.Fatalf("could not greet: %v", err)
	}
	log.Printf("Greeting: %s", r.Message)
}

```

**(6)test**
到此分别运行 server.go、client.go ，一个简单的 gRPC demo 就跑起来了。

```text
lixd@17x:~/17x/projects/grpc-go-example/features/gateway/server$ go run main.go 
2021/01/30 10:15:53 Serving gRPC on 0.0.0.0:50051


lixd@17x:~/17x/projects/grpc-go-example/features/gateway/client$ go run main.go 
2021/01/30 10:17:23 Greeting: hello world

```

### gRPC-Gateway 部分
#### 实现方式

![](attachments/Pasted%20image%2020241223201716.png)


## 不修改proto文件方式实现 gRPC-Gateway
### 介绍
如果不存在`proto` 文件的访问权限，无法进行修改。可以借助于**额外的映射配置文件(比如：yaml文件)+ proto 文件** 来生成`grpc-gateway`程序。
比如：通过指定`grpc-gateway_opt`的参数`grpc_api_configuration=path/to/config.yaml`
来制定的 映射的 `yaml` 文件。

![](attachments/Pasted%20image%2020241223200612.png)

参考：[# Go-gRPC-Gateway](https://www.cnblogs.com/remixnameless/p/15975971.html)

### 范例一

参考：[# Go-gRPC-Gateway](https://www.cnblogs.com/remixnameless/p/15975971.html)

#### 环境搭建
```bash
$ go install \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-grpc-gateway \
    github.com/grpc-ecosystem/grpc-gateway/v2/protoc-gen-openapiv2 \
    google.golang.org/protobuf/cmd/protoc-gen-go \
    google.golang.org/grpc/cmd/protoc-gen-go-grpc
```

#### 整体目录结构
![](attachments/Pasted%20image%2020241223224430.png)

#### proto文件定义
trip.proto

```protobuf
syntax = "proto3";
package coolcar;
option go_package="coolcar/protc/gen/go;trippb";

message Location {
    double latitude = 1;
    double longitude = 2;
}

enum TripStatus {
    TS_NOT_SPECIFIED = 0;
    NOT_STARTED = 1;
    IN_PROGRESS = 2;
    FINISHED = 3;
    PAID = 4;
}

message Trip {
    string start =1;
    string end =2;
    int64 duration_sec = 3;
    int64 fee_cent = 4;
    Location start_pos = 5;
    Location end_pos = 6;
    repeated Location path_locations = 7;
    TripStatus status =8;
    bool isPromotionTrip = 9;
    bool isFromGuestUser = 10;
}

message GetTripRequest {
    string id = 1;
}

message GetTripResponse {
    string id = 1;
    Trip trip = 2;
}

service TripService {
    rpc GetTrip (GetTripRequest) returns (GetTripResponse);
}
```

#### yaml文件定义
trip.yaml

```yaml
type: google.api.Service
config_version: 3

http:
  rules:
    # selector定义：protoc定义的包名+service服务名+rpc方法名(coolcar.TripService.GetTrip)
    - selector: coolcar.TripService.GetTrip
      # 向外暴露的REST API风格接口
      # get方法； urL: /trip/{id}; {id}是参数，基于id进行查找。
      get: /trip/{id}
```

#### 代码生成脚本

gen.sh
```bash
protoc -I=. --go-grpc_out=paths=source_relative:gen/go  --go_out=paths=source_relative:gen/go trip.proto
protoc -I=. --grpc-gateway_out=paths=source_relative,grpc_api_configuration=trip.yaml:gen/go trip.proto
```

在proto目录下执行./gen.sh生成proto文件


#### Service服务端定义

```go
# cat trip.go

package trip

import (
	"context"
	trippd "coolcar/proto/gen/go"
)

// Service
type Service struct {
	trippd.UnimplementedTripServiceServer
}

func (s *Service) GetTrip(c context.Context, req *trippd.GetTripRequest) (*trippd.GetTripResponse, error) {
	return &trippd.GetTripResponse{
		Id: req.Id,
		Trip: &trippd.Trip{
			Start:       "abc",
			End:         "",
			DurationSec: 3600,
			FeeCent:     10000,
			StartPos: &trippd.Location{
				Latitude:  30,
				Longitude: 120,
			},
			EndPos: &trippd.Location{
				Latitude:  35,
				Longitude: 115,
			},
			PathLocations: []*trippd.Location{
				{
					Latitude:  31,
					Longitude: 119,
				},
				{
					Latitude:  32,
					Longitude: 118,
				},
			},
			Status: trippd.TripStatus_IN_PROGRESS,
		},
	}, nil
}
```


```go
# cat main.go

package main

import (
	"context"
	trippd "coolcar/proto/gen/go"
	trip "coolcar/tripservice"
	"log"
	"net"
	"net/http"

	"google.golang.org/protobuf/encoding/protojson"

	"github.com/grpc-ecosystem/grpc-gateway/v2/runtime"
	"google.golang.org/grpc"
)

const Addr = ":8081"

func main() {
	log.SetFlags(log.Lshortfile)
	go startGRPCGateway()
	s := grpc.NewServer()
	trippd.RegisterTripServiceServer(s, &trip.Service{})

	lis, err := net.Listen("tcp", Addr)
	if err != nil {
		log.Printf("failed to listen: %v", err)
	}

	log.Fatal(s.Serve(lis))
}

func startGRPCGateway() {
	c := context.Background()
	c, cancel := context.WithCancel(c)
	defer cancel()
	/**
	grpc-gatewayV1、V2版本不兼容
	EnumsAsInts -> MarshalOptions.UseEnumNumbers
	OrigName -> MarshalOptions.UseProtoNames
	DiscardUnknown -> UnmarshalOptions. DiscardUnknown
	*/
	mux := runtime.NewServeMux(runtime.WithMarshalerOption(runtime.MIMEWildcard, &runtime.JSONPb{
		MarshalOptions: protojson.MarshalOptions{
			UseProtoNames:  true, // 使用proto字段名代替JSON中的骆驼式名称的字段名。  
			UseEnumNumbers: true, // 使用protoc枚举定义的值作为数字发送。
		},
		UnmarshalOptions: protojson.UnmarshalOptions{
			DiscardUnknown: true, // 未知字段将被忽略
		},
	}))
	err := trippd.RegisterTripServiceHandlerFromEndpoint(
		c,
		mux,
		Addr,
		[]grpc.DialOption{grpc.WithInsecure()})
	if err != nil {
		log.Fatalf("connot start grpc gateway:%v", err)
	}
	err = http.ListenAndServe(":8080", mux)
	if err != nil {
		log.Fatalf("connot listen and server:%v", err)
	}
}
```


#### Client客户端定义
```go
# cat main.go

package main

import (
	"context"
	trippb "coolcar/proto/gen/go"
	"fmt"
	"log"

	"google.golang.org/grpc"
)

func main() {
	conn, err := grpc.Dial(":8081", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("cannot dial: %v", err)
	}
	tsClient := trippb.NewTripServiceClient(conn)
	r, err := tsClient.GetTrip(context.Background(), &trippb.GetTripRequest{Id: "trip456"})
	if err != nil {
		log.Fatalf("cannot call GetTrip: %v", err)
	}
	fmt.Println(r)
}
```

#### 测试效果
```bash
$ curl http://localhost:8080/trip/trip123

{"id":"trip123","trip":{"start":"abc","duration_sec":3600,"fee_cent":10000,"start_pos":{"latitude":30,"longitude":120},"end_pos":{"latitude":35,"longitude":115},"path_locations":[{"latitude":31,"longitude":119},{"latitude":32,"longitude":118}],"status":2}}
```

![](attachments/Pasted%20image%2020241223225406.png)

![](attachments/Pasted%20image%2020241223225412.png)

### 范例二

参考：[# 使用 gRPC-gateway 代替 HTTP 框架在Go语言中进行开发 | gRPC + gRPC-gateway 开发实践](https://juejin.cn/post/7155811498722328589)
####  test.proto 文件
```go
syntax = "proto3";
package test;
option go_package = "./test/api/gen;testpb";

message HelloRequest{
  string name = 1;
}

message HelloResponse{
  string resp = 1;
}

message SpecificRequest{
  string req = 1;
}

message SpecificResponse{
  string resp = 1;
}

service TestService {
  rpc Hello (HelloRequest) returns (HelloResponse);
  rpc Specific (SpecificRequest) returns (SpecificResponse);
}

```


#### yaml映射文件
GRPC API接口（Sevice中的函数） 和 HTTP 的 URL 的 映射关系，如下所示：
```yaml
# cat test.yaml 
type: google.api.Service
config_version: 3

http:
  rules:
    - selector: test.TestService.Hello
      post: "/test/hello"
      body: "*"
    - selector: test.TestService.Specific
      post: "/test/specific"
      body: "*"

```


#### 编译生成代码
基于 pb 文件，以及各个插件，生成对应的 go 代码。

```bash
protoc \
    --go_out=gen --go_opt=paths=source_relative \
    --go-grpc_out=gen --go-grpc_opt=paths=source_relative \
    --grpc-gateway_out=gen --grpc-gateway_opt=paths=source_relative,grpc_api_configuration=test.yaml \
    ./test.proto

```

生成成功后我们的 gen 文件中应该会产生三个新的文件，目前的项目结构如下。
```text
├── go.mod
└── test
    └── api
        ├── build.sh
        ├── gen
        │   ├── test.pb.go
        │   ├── test.pb.gw.go
        │   └── test_grpc.pb.go
        ├── test.proto
        └── test.yaml

```

## 修改proto文件方式实现 gRPC-Gateway


### 介绍
修改proto文件，来完成 GRPC 的 Server的 函数接口和 HTTP的 URL 路径的映射。

参考：[# gRPC(Go)教程(七)---利用Gateway同时提供HTTP和RPC服务](https://www.lixueduan.com/posts/grpc/07-grpc-gateway/)

详细的实现，忽略。



# 实现二：使用`json`协议的grpc

参考：[使用`json`协议的grpc](https://www.cnblogs.com/polaris1119/p/13498828.html)

## 介绍
大家经常说 gRPC 是基于 `Google 的 Protocol Buffers` 作为 payload 格式的，然而这不完全正确。`gRPC payload` 的默认格式是 `Protobuf`，但是 `gRPC-Go` 的实现中也对外暴露了 `Codec` interface ，它支持任意的 payload 编码。我们可以使用任何一种格式，包括你自己定义的二进制格式、`flatbuffers`、或者使用我们今天要讨论的 `JSON`，作为请求和响应。

# 其他

## yaml 文件讲解
### 目标
通过写`yaml`文件，将对应的`grpc`的`API`接口（某个`package`的某个`service`的某个函数接口）转换为`http`的`url`路径。


## 参数

# 参考
```bash
# GO-grpc 实践指南【整个系列都挺好的】
https://jergoo.gitbooks.io/go-grpc-practice-guide/content/chapter3/gateway.html

# Go-gRPC-Gateway
https://www.cnblogs.com/remixnameless/p/15975971.html
[整个系列都不错：https://www.cnblogs.com/remixnameless/category/1917783.html]

# gprc_api_configuration 的使用
https://www.cuiwei.net/p/1356027288

# gRPC之gRPC转换HTTP【csdn的系列都不错】
https://blog.csdn.net/qq_30614345/article/details/133954006
# gRPC之gRPC Gateway
https://blog.csdn.net/qq_30614345/article/details/133831946

# gRPC(Go)教程(七)---利用Gateway同时提供HTTP和RPC服务
https://www.lixueduan.com/posts/grpc/07-grpc-gateway/



# golang gRPC转换HTTP对外提供服务
https://juejin.cn/post/7025896168018149412


# 一款可以实现http转grpc的开源网关
https://learnku.com/articles/76088
```