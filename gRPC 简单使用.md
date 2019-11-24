<!-- TOC -->

- [简介](#简介)
- [定义服务](#定义服务)
- [Golang 下使用](#golang-下使用)
- [参考](#参考)

<!-- /TOC -->

## 简介

RPC 的全称是 Remote Procedure Call(远程过程调用), 即可以在客户端应用程序中直接调用其他计算机(服务端)上定义的方法.

gRPC 是一个 RPC 框架, 使用 protobuf 作为数据交换协议.

## 定义服务

既然是 RPC 系统, 主要的目的在于定义方法, 或者说服务 service.

```protobuf
service HelloService {
  rpc SayHello (HelloRequest) returns (HelloResponse);
}

message HelloRequest {
  string greeting = 1;
}

message HelloResponse {
  string reply = 1;
}
```

在 gRPC 中可以定义四种类型的服务:

- 一元 RPC: 客户端向服务器发送单个请求并获取单个响应, 类似普通函数调用
- 服务器流式 RPC: 客户端发送单个请求, 服务端返回流式响应
- 客户端流式 RPC: 客户端发送流式请求, 服务端返回单个响应
- 双向流式 RPC: 使用两个独立的流, 客户端发送流式请求, 服务端返回流式响应

```protobuf
rpc SayHello(HelloRequest) returns (HelloResponse){
}
rpc LotsOfReplies(HelloRequest) returns (stream HelloResponse){
}
rpc LotsOfGreetings(stream HelloRequest) returns (HelloResponse) {
}
rpc BidiHello(stream HelloRequest) returns (stream HelloResponse){
}
```

## Golang 下使用

定义 protobuf 文件 `hello.proto` 的内容为:

```protobuf
syntax = "proto3";

import "google/protobuf/any.proto";

package hello;
option go_package = "hello";

message HelloReq {
  string name = 1;
}

message HelloResp {
  int32 code = 1;
  string greet = 2;
  google.protobuf.Any details = 3;
}

service HelloService {
  rpc Greet(HelloReq) returns (HelloResp) {};
  rpc GreetWithServerStream(HelloReq) returns (stream HelloResp) {};
  rpc GreetWithClientStream(stream HelloReq) returns (HelloResp) {};
  rpc GreetWithBidirectionalStream(stream HelloReq) returns (stream HelloResp) {};
}
```

初始化项目, 安装必要的依赖:

```powershell
go mod init tzh.com/app
go get -u github.com/golang/protobuf/protoc-gen-go
go get -u google.golang.org/grpc
mkdir hello
# 假设 protoc3 已经解压好了
.\protoc3\bin\protoc.exe --proto_path=. hello.proto --go_out=plugins=grpc:./hello
```

生成代码的时候, 注意指定插件 `plugins=grpc`.

main.go 如下:

```go
package main

import (
	"context"
	"flag"
	"fmt"
	"io"
	"log"
	"net"
	"strings"

	"google.golang.org/grpc"
	pb "tzh.com/app/hello"
)

const port = ":5000"

type server struct {
	pb.UnimplementedHelloServiceServer
}

func (s *server) Greet(ctx context.Context, in *pb.HelloReq) (*pb.HelloResp, error) {
	return &pb.HelloResp{Code: 0, Greet: "hello " + in.GetName()}, nil
}

// 对于空格分隔的 name, 使用 stream 发送多次数据
func (s *server) GreetWithServerStream(in *pb.HelloReq, stream pb.HelloService_GreetWithServerStreamServer) error {
	names := strings.Split(in.GetName(), " ")
	for i, name := range names {
		err := stream.Send(&pb.HelloResp{
			Code:  int32(i),
			Greet: fmt.Sprintf("part %d: hello %s", i, name),
		})
		if err != nil {
			return err
		}
	}
	return nil
}

// 对于客户端发送的多个 name, 合并后发送单条响应
func (s *server) GreetWithClientStream(stream pb.HelloService_GreetWithClientStreamServer) error {
	names := make([]string, 0)
	for {
		msg, err := stream.Recv()
		if err != nil {
			break
		}
		names = append(names, msg.GetName())
	}
	stream.SendAndClose(&pb.HelloResp{
		Code:  0,
		Greet: fmt.Sprintf("hello %s count: %d", strings.Join(names, " "), len(names)),
	})
	return nil
}

// 双向流, 对于每个请求, 一一响应
func (s *server) GreetWithBidirectionalStream(stream pb.HelloService_GreetWithBidirectionalStreamServer) error {
	for {
		msg, err := stream.Recv()
		if err == io.EOF {
			return nil
		}
		if err != nil {
			return err
		}
		if err := stream.Send(&pb.HelloResp{
			Code:  0,
			Greet: "hello " + msg.GetName(),
		}); err != nil {
			return err
		}
	}
}

func runServer() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen on %s: %v", port, err)
	}
  s := grpc.NewServer()
  // 注册服务
	pb.RegisterHelloServiceServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to server: %v", err)
	}
}

func run1(client pb.HelloServiceClient) {
	log.Println("############ Greet ")
	r, err := client.Greet(context.Background(), &pb.HelloReq{
		Name: "tt",
	})
	if err != nil {
		log.Fatalf("failed to get greet resp: %v", err)
	}
	log.Printf("get code : %d, get greet: %s \n", r.GetCode(), r.GetGreet())
}

func run2(client pb.HelloServiceClient) {
	log.Println("############ GreetWithServerStream")
	serverStream, err := client.GreetWithServerStream(context.Background(), &pb.HelloReq{
		Name: "tt aa xx ff",
	})
	if err != nil {
		log.Fatalf("failed with GreetWithServerStream: %v", err)
	}

	for {
		r, err := serverStream.Recv()
		if err != nil {
			break
		}
		log.Printf("get code : %d, get greet: %s \n", r.GetCode(), r.GetGreet())
	}
}

func run3(client pb.HelloServiceClient) {
	log.Println("############ GreetWithClientStream ")
	clientStream, err := client.GreetWithClientStream(context.Background())
	if err != nil {
		log.Fatalf("failed with GreetWithClientStream: %v", err)
	}

	for _, name := range []string{"tt", "qq", "aa", "yy"} {
		err := clientStream.Send(&pb.HelloReq{
			Name: name,
		})
		if err != nil {
			log.Fatalf("failed with GreetWithClientStream during send request: %v", err)
			break
		}
	}
	r, err := clientStream.CloseAndRecv()
	if err != nil {
		log.Fatalf("failed with GreetWithClientStream when get response: %v", err)
	}
	log.Printf("get code : %d, get greet: %s \n", r.GetCode(), r.GetGreet())
}

func run4(client pb.HelloServiceClient) {
	log.Println("############ GreetWithBidirectionalStream ")
	stream, err := client.GreetWithBidirectionalStream(context.Background())
	if err != nil {
		log.Fatalf("failed with GreetWithBidirectionalStream: %v", err)
	}

	for _, name := range []string{"tt", "qq", "aa", "yy"} {
		err := stream.Send(&pb.HelloReq{
			Name: name,
		})
		if err != nil {
			log.Fatalf("failed with GreetWithClientStream during send request: %v", err)
			break
		}
	}
	stream.CloseSend()

	for {
		r, err := stream.Recv()
		if err != nil {
			break
		}
		log.Printf("get code : %d, get greet: %s \n", r.GetCode(), r.GetGreet())
	}
}

func runClient() {
	address := "localhost" + port
	conn, err := grpc.Dial(address, grpc.WithInsecure())
	if err != nil {
		log.Fatalf("failed to connect %s: %v", address, err)
	}
	defer conn.Close()
	client := pb.NewHelloServiceClient(conn)

	run1(client)
	run2(client)
	run3(client)
	run4(client)
}

func main() {
	isClient := flag.Bool("client", false, "run client")
	flag.Parse()
	if *isClient {
		runClient()
	} else {
		runServer()
	}
}
```

代码有点长, 因为将服务端代码和客户端代码都混合在同一个文件中, 使用下面的命令分别启动服务端和运行客户端.

```bash
# 运行服务端
go run main.go
# 运行客户端
go run main.go --client
```

## 参考

- [grpc](https://grpc.io/docs/tutorials/basic/go/)
- [grpc-go](https://github.com/grpc/grpc-go)
