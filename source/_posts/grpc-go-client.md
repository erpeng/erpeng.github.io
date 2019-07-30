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
### 

