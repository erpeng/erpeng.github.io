---
title: grpc-go源码解析3-resolver
date: 2019-08-09
tags: grpc-go
---

>grpc客户端源码分析,重点分析grpc中的名字解析器resolver

## 概览

![overview](/img/grpc31.png)

* 一个grpc客户端首先通过resolver获取多个IP地址,这些IP地址可能是服务端的地址也可能是一个load balancer的地址,随着地址返回的还有一个service config,其中会指明load balance的策略(例如round_robin或者grpclb)
* 客户端初始化load balance策略.如果resolver返回的地址只要有一个是load balance的地址,客户端就会使用grpclb策略.否则的话根据service config的配置决定.如果service config没有指定load balance粗略,则默认选用pick-first
* 创建到每一个server address的subchannel
  * grpclb策略下,client连接load balancer,并向其请求服务器地址
  * grpc server会向load balancer汇报其负载情况
  * load balancer向client返回一个server list,client向每一个server都建立一个subchannel
* 对每一个rpc请求,load balancer决定哪一个subchannel应该被使用.

整理其流程可知有如下两种方式获取服务端地址:

* resolver返回多个server地址,然后client根据round-robin或者pick-first或者random的策略选择一个server去连接.

* resolver返回load-balancer的地址,load-balancer会去做server的负载检查,探活策略,然后根据负载均衡策略返回一个地址给client.

本文重点讲解resolver的实现逻辑

## resolver原理

首先通过一个示例看看如何注册一个resolver:

```
type exampleResolverBuilder struct{}

func (*exampleResolverBuilder) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
	r := &exampleResolver{
		target: target,
		cc:     cc,
		addrsStore: map[string][]string{
			exampleServiceName: {backendAddr},
		},
	}
	r.start()
	return r, nil
}
func (*exampleResolverBuilder) Scheme() string { return exampleScheme }

// exampleResolver is a
// Resolver(https://godoc.org/google.golang.org/grpc/resolver#Resolver).
type exampleResolver struct {
	target     resolver.Target
	cc         resolver.ClientConn
	addrsStore map[string][]string
}

func (r *exampleResolver) start() {
	addrStrs := r.addrsStore[r.target.Endpoint]
	addrs := make([]resolver.Address, len(addrStrs))
	for i, s := range addrStrs {
		addrs[i] = resolver.Address{Addr: s}
	}
	r.cc.UpdateState(resolver.State{Addresses: addrs})
}
func (*exampleResolver) ResolveNow(o resolver.ResolveNowOption) {}
func (*exampleResolver) Close()                                 {}

func init() {
	// Register the example ResolverBuilder. This is usually done in a package's
	// init() function.
	resolver.Register(&exampleResolverBuilder{})
}
```
关键步骤有两步:
* init()函数中进行注册
* Build函数中生成一个resolver并且调用该resolver的start函数,start函数中会调用r.cc.UpdateState函数

继续追踪一下UpdateState函数,但在查看该函数之前,首先看下client端如何获取builder并且何时调用build函数,build函数中的UpdateState做什么事.

在grpc.Dial()中会将resolver builder赋值给如下变量
```
cc.dopts.resolverBuilder

```

然后执行:
```
	...
	rWrapper, err := newCCResolverWrapper(cc)
	...
	cc.mu.Lock()
	cc.resolverWrapper = rWrapper
	cc.mu.Unlock()

```
在newCCResolverWrapper中会调用builder的build函数并赋值给rWrapper的resolver字段.rWrapper是一个ccResolverWrapper结构,所以UpdateState实际调用的是ccResolverWrapper的方法.通过追踪代码,发现UpdateState最终调用的是ClientConn结构体的updateResolverState,其中比较关键是部分涉及到balancer相关配置.下一节再详细讲述

## 小结

resolver主要解决从域名获取地址这一步,这一步有可能直接获取到服务端地址也可能获取到balancer的地址.下一讲balancer部分详细描述.

