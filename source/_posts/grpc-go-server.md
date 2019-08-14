---
title: grpc-go源码解析5-method
date: 2019-08-14
tags: grpc-go
tags: HTTP2
---

>简略分析服务端代码,关键是如何获取调用的服务及方法

## 概览

grpc使用protobuf序列化数据并传输,那么服务端如何知道client调用的是哪个服务的哪个方法呢?
我们通过追踪服务端代码观察一下该过程

## 服务端调用

### 代码路径
服务端关键代码如下:
```
	...
	s := grpc.NewServer()
	pb.RegisterGreeterServer(s, &server{})
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
	...
```
第一步通过NewServer()生成一个grpc.Server结构,第二步RegisterGreeterServer将用户定义的服务注册到之前生成的grpc.Server.注册内容我们先通过grpc.Server结构体来看一下:

```
type Server struct {
	...
	m      map[string]*service // service name -> service info
	...
}
```
注册及将一个服务名和一个具体的服务在Server的m map中关联起来.接着看下service结构:
```
type service struct {
	server interface{} // the server for service methods
	md     map[string]*MethodDesc
	sd     map[string]*StreamDesc
	mdata  interface{}
}
type MethodDesc struct {
	MethodName string
	Handler    methodHandler
}
type StreamDesc struct {
	StreamName string
	Handler    StreamHandler

	// At least one of these is true.
	ServerStreams bool
	ClientStreams bool
}
```
service结构中也是通过md和sd将method的name和具体的handler关联起来,不赘述
接着第三部调用grpc.Server的Serve方法开始进入正式执行流程

### 服务处理

* func (s *Server) handleRawConn(rawConn net.Conn)
* func (s *Server) serveStreams(st transport.ServerTransport)
* func (t *http2Server) HandleStreams(handle func(*Stream), traceCtx func(context.Context, string) context.Context) {
* func (s *Server) handleStream(t transport.ServerTransport, stream *transport.Stream, trInfo *traceInfo)

第三部是一个关键的步骤,其中会生成一个Stream结构,Stream中会包含调用的方法
具体调用链路为:
func (t *http2Server) operateHeaders(frame *http2.MetaHeadersFrame, handle func(*Stream), traceCtx func(context.Context, string) context.Context) (fatal bool) {
func (d *decodeState) decodeHeader(frame *http2.MetaHeadersFrame) error {
func (d *decodeState) processHeaderField(f hpack.HeaderField) {

其中关键代码为:
```
...
	case ":path":
		d.data.method = f.Value
...
```
即根据http2 meta header的:path这个key,找到rpc调用的方法

## 结论

grpc使用protobuf序列化传输数据,然后根据:path这个http2头获取需要调用的方法
