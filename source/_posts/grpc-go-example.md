---
title: grpc-go源码解析1-example
date: 2019-07-30
tags: grpc-go
---

>grpc使用protobuf编解码,使用http2传输数据.我们通过一个示例来看下如何编写grpc客户端和服务端代码.注意代码分为两部分,一部分是protobuf生成的stub,一部分是需要用户实际编写的代码.以下分别用protobuf客户端,protobuf服务端以及grpc客户端和grpc服务端指代.

## protobuf

具体安装和生成protobuf代码细节可以参考链接
protobuf定义文件内容如下:
```
syntax = "proto3";

option java_multiple_files = true;
option java_package = "io.grpc.examples.helloworld";
option java_outer_classname = "HelloWorldProto";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```
比较关键的语法是package-定义包名,service-定义服务名,包+服务的路径下可以定义各种可供rpc调用的函数.上述proto文件的SayHello函数在grpc服务端注册路径为/helloworld.Greeter/SayHello.接着定义函数原型及入参和出参类型结构体,而message定义一个具体的类型结构体.

## 客户端
我们看下protobuf客户端代码:
```
type GreeterClient interface {
	// Sends a greeting
	SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error)
}

type greeterClient struct {
	cc *grpc.ClientConn
}

func NewGreeterClient(cc *grpc.ClientConn) GreeterClient {
	return &greeterClient{cc}
}

func (c *greeterClient) SayHello(ctx context.Context, in *HelloRequest, opts ...grpc.CallOption) (*HelloReply, error) {
	out := new(HelloReply)
	err := c.cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
	if err != nil {
		return nil, err
	}
	return out, nil
}
```
通过protobuf客户端代码可以猜测grpc客户端代码只需首先生成一个grpc.ClientConn结构,然后调用NewGreeeterClient生成一个greeterClient结构,即可调用SayHello方法.SayHello方法最终调用的是grpc.ClientConn结构的invoke方法,调用路径为/helloworld.Greeter/Sayhello,可以猜想到,grpc服务端肯定在这个路径会注册一个钩子函数去执行,并且该钩子函数需要在grpc服务端自己定义.

protobuf客户端生成的只是一个wrapper,我们完全可以直接按如下方法调用:
```
cc.Invoke(ctx, "/helloworld.Greeter/SayHello", in, out, opts...)
```

看下grpc客户端实际的代码:
```
package main
import (
    ...
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)
const (
	address     = "localhost:50051"
	defaultName = "world"
)

func main() {
	conn, err := grpc.Dial(address, grpc.WithInsecure())
    ...
	c := pb.NewGreeterClient(conn)
    ...
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
    ...
	r, err := c.SayHello(ctx, &pb.HelloRequest{Name: name})
    ...
	log.Printf("Greeting: %s", r.Message)
}
```
只保留关键路径,可以看到,grpc.Dial生成一个grpc.ClientConn,其他步骤与上文想法一致

## 服务端

我们看下protobuf服务端代码:
```
type GreeterServer interface {
	SayHello(context.Context, *HelloRequest) (*HelloReply, error)
}

func RegisterGreeterServer(s *grpc.Server, srv GreeterServer) {
	s.RegisterService(&_Greeter_serviceDesc, srv)
}

func _Greeter_SayHello_Handler(srv interface{}, ctx context.Context, dec func(interface{}) error, interceptor grpc.UnaryServerInterceptor) (interface{}, error) {
	in := new(HelloRequest)
	if err := dec(in); err != nil {
		return nil, err
	}
	if interceptor == nil {
		return srv.(GreeterServer).SayHello(ctx, in)
	}
	info := &grpc.UnaryServerInfo{
		Server:     srv,
		FullMethod: "/helloworld.Greeter/SayHello",
	}
	handler := func(ctx context.Context, req interface{}) (interface{}, error) {
		return srv.(GreeterServer).SayHello(ctx, req.(*HelloRequest))
	}
	return interceptor(ctx, in, info, handler)
}

var _Greeter_serviceDesc = grpc.ServiceDesc{
	ServiceName: "helloworld.Greeter",
	HandlerType: (*GreeterServer)(nil),
	Methods: []grpc.MethodDesc{
		{
			MethodName: "SayHello",
			Handler:    _Greeter_SayHello_Handler,
		},
	},
	Streams:  []grpc.StreamDesc{},
	Metadata: "helloworld.proto",
}
```
通过protobuf服务端可以猜想grpc服务端关键结构体有grpc.ServiceDesc,grpc.MethodDesc,grpcStreamDesc.
首先需要grpc服务端生成一个grpc.Server指针,并且实现GreeterServer接口(即实现SayHello方法),然后调用RegisterGreeterServer即可将其注册到grpc服务端.
我们看到grpc.ServiceDesc中的Methods字段保存的即为方法SayHello和具体的钩子函数,回调钩子函数为_Greeter_SayHello_Handler,该函数也是protobuf服务端代码,其关键步骤如下:
* 使用dec解析输入
* 如果interceptor为空,即不存在拦截器,则直接调用grpc服务端GreeterServer的SayHello方法并且返回
* 否则返回interceptor(ctx, in, info, handler),interceptor为函数,handler为grpc服务端GreeterServer的SayHello方法,可以猜测到interceptor函数先将输入处理完后最后调用handler处理

grpc服务端实际代码如下:
```
package main
import (
    ...
	pb "google.golang.org/grpc/examples/helloworld/helloworld"
)
const (
	port = ":50051"
)
type server struct{}

func (s *server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.Name)
	return &pb.HelloReply{Message: "Hello " + in.Name}, nil
}

func main() {
	lis, err := net.Listen("tcp", port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	s := grpc.NewServer()//生成一个grpc.Server指针
	pb.RegisterGreeterServer(s, &server{})//注册一个GreeterServer,server结构体实现了GreeterServer接口
	if err := s.Serve(lis); err != nil { //调用grpc.Server的Serve函数
		log.Fatalf("failed to serve: %v", err)
	}
}
```
同理,grpc服务端生成一个grpc.Server并且注册之后,调用Serve函数,并且传入一个Listener,即完成了该代码逻辑.

## 后记

接下来会首先分析grpc源码中设计客户端的grpc.ClientConn结构体和grpc.Dial()函数生成该结构体的过程以及关键的CientConn的invoke()函数如何发起一个请求
```
grpc.Dial()
ClientConn:
    invoke()
```
服务端grpc.Server结构体,grpc.ServiceDesc,grpc.MethodDesc,grpc.StreamDesc以及grpc.NewServer()生成一个Server结构体,Server结构体的RegisterService()函数如何注册一个服务,以及Serve()函数如何提供服务(接收请求,解析请求,通过注册服务的回调函数处理请求,返回相应).
```
grpc.NewServer()
Server:
    RegisterService()
    Serve()
```

## 参考链接

* https://github.com/grpc/grpc-go/blob/master/examples/README.md