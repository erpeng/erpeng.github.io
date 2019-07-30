---
title: grpc-go源码解析2-client
date: 2019-07-30
tags: grpc-go
---

>grpc客户端源码分析

## grpc.Dial()

### 函数原型分析
func Dial(target string, opts ...DialOption) (*ClientConn, error) {
  return DialContext(context.Background(), target, opts...)
}

首先看一下Dial函数的原型,两个参数,一个为target代表服务端的地址,一个为可变长度参数opts,为DialOption.
DialOption定义如下:
```
type DialOption interface {
  apply(*dialOptions)
}
```
可以看到是一个接口,定义了一个apply函数.我们可以猜测到Dial()函数中最后如何应用这些参数呢,如下:
```
for opt := range opts {
  opt.apply(*dialOptions)
}
```

看看grpc中如何实现参数的生成:
```
type funcDialOption struct {
  f func(*dialOptions)
}

func (fdo *funcDialOption) apply(do *dialOptions) {
  fdo.f(do)
}

func newFuncDialOption(f func(*dialOptions)) *funcDialOption {
  return &funcDialOption{
    f: f,
  }
}
```
funcDialOption实现了apply函数,那么如何生成一个funcDialOption结构呢,使用newFuncDialOption函数生成,该函数需要定义一个函数f.例如:grpc.WithInsecure函数定义如下(所有参数相关的配置都在dialoptions.go文件中):
```
func WithInsecure() DialOption {
  return newFuncDialOption(func(o *dialOptions) {
    o.insecure = true
  })
}
```
### 关键结构体
clientConn结构体
```
type ClientConn struct {
  ctx    context.Context
  cancel context.CancelFunc

  target       string  //目标服务器
  parsedTarget resolver.Target //将target解析为resolver.Target结构
  authority    string
  dopts        dialOptions //tcp连接相关参数设置
  csMgr        *connectivityStateManager //连接状态管理

  balancerBuildOpts balancer.BuildOptions
  blockingpicker    *pickerWrapper

  mu              sync.RWMutex
  resolverWrapper *ccResolverWrapper
  sc              *ServiceConfig 
  conns           map[*addrConn]struct{}
  // Keepalive parameter can be updated if a GoAway is received.
  mkp             keepalive.ClientParameters //keepalive相关
  curBalancerName string
  balancerWrapper *ccBalancerWrapper
  retryThrottler  atomic.Value

  firstResolveEvent *grpcsync.Event

  channelzID int64 // 统计相关
  czData     *channelzData
}
```

一个关键字段为dopts,代表dialOptions,其结构如下:
```
type dialOptions struct {
  unaryInt  UnaryClientInterceptor //rpc拦截器
  streamInt StreamClientInterceptor 

  chainUnaryInts  []UnaryClientInterceptor
  chainStreamInts []StreamClientInterceptor

  cp          Compressor
  dc          Decompressor
  bs          backoff.Strategy //重试策略
  block       bool //连接类型,阻塞和非阻塞
  insecure    bool //是否需要验证证书
  timeout     time.Duration //连接超时
  scChan      <-chan ServiceConfig 
  authority   string
  copts       transport.ConnectOptions
  callOptions []CallOption
  
  balancerBuilder balancer.Builder

  resolverBuilder             resolver.Builder
  channelzParentID            int64
  disableServiceConfig        bool
  disableRetry                bool //是否禁止重试
  disableHealthCheck          bool
  healthCheckFunc             internal.HealthChecker
  minConnectTimeout           func() time.Duration
  defaultServiceConfig        *ServiceConfig
  defaultServiceConfigRawJSON *string
}
```
### 调用流程
实际调用函数为DialContext,该函数其实就是进行clientConn结构体和dialOptions(cc.dopts)的各字段初始化

调用流程如下:
* 应用配置参数
```
  for _, opt := range opts {
    opt.apply(&cc.dopts)
  }
```
* 设置interceptor相关
```
  chainUnaryClientInterceptors(cc)
  chainStreamClientInterceptors(cc)
```

拦截器串行执行后最终需要调用客户端实际调用rpc的函数,chainUnaryClientInterceptors的作用就是将各个拦截器串联之后放置到cc.unaryInt.最终只需调用cc.unaryInt就会将所有拦截器依次执行



