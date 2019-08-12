---
title: grpc-go源码解析4-balancer
date: 2019-08-12
tags: grpc-go
---

>grpc客户端源码分析,重点分析grpc中的负载均衡器balancer

## 概览

```
roundrobinConn, err := grpc.Dial(
		fmt.Sprintf("%s:///%s", exampleScheme, exampleServiceName),
		grpc.WithBalancerName("round_robin"), // This sets the initial balancing policy.
		grpc.WithInsecure(),
	)
```

选定balance策略只需在客户端grpc.Dial时指定BalancerName.
重点看下grpc.Dial时balancer的作用及其执行步骤

## 调用流程

书接上文,resolver调用ClientConn中的updateResolverState函数,该函数有如下关键流程
```
    if cc.dopts.balancerBuilder == nil {
		var newBalancerName string
		if cc.sc != nil && cc.sc.lbConfig != nil {
			newBalancerName = cc.sc.lbConfig.name
			balCfg = cc.sc.lbConfig.cfg
		} else {
			var isGRPCLB bool
			for _, a := range s.Addresses {
				if a.Type == resolver.GRPCLB {
					isGRPCLB = true
					break
				}
			}
			if isGRPCLB {
				newBalancerName = grpclbName
			} else if cc.sc != nil && cc.sc.LB != nil {
				newBalancerName = *cc.sc.LB
			} else {
				newBalancerName = PickFirstBalancerName
			}
		}
		cc.switchBalancer(newBalancerName)
	} else if cc.balancerWrapper == nil {
		cc.curBalancerName = cc.dopts.balancerBuilder.Name()
		cc.balancerWrapper = newCCBalancerWrapper(cc, cc.dopts.balancerBuilder, cc.balancerBuildOpts)
	}

	cc.balancerWrapper.updateClientConnState(&balancer.ClientConnState{ResolverState: s, BalancerConfig: balCfg})
```
当在grpc.Dial中指定WithBalancerName("round_robin")之后会走else语句,否则走if语句,if语句之中选取newBalancerName的逻辑见resolver章节中概述第二条描述.两段逻辑中不论是if的switchBalancer还是else语句都是需要生成一个balancerWrapper然后复制到cc.balancerWrapper.

重点看newCCBalancerWrapper函数,关键是如下两句:
```
	go ccb.watcher()
	ccb.balancer = b.Build(ccb, bopts)
```
ccb.watcher中会检测管道是否可读,可读时执行balancer的HandleResolvedAddrs函数.

那么管道何时可读呢,通过上文代码中的updateClientConnState函数,该函数关键代码如下:
```
	select {
	case <-ccb.ccUpdateCh:
	default:
	}
	ccb.ccUpdateCh <- ccs
```
向ccb.ccUpdateCh中输入内容

继续分析管道可读之后执行的HandleResolvedAddrs函数,该函数中会连接后端服务器并且返回一个可用链接.

## 小结

自己感觉此处resolver和balancer的代码包装过度,balancer中又有grpclb这种外部负载均衡器和pickfirst/roundrobin等内部负载均衡方式,整体不是太好理解.可以切分为两种模式:

resolver可以返回一个结构,类似
```
"type":"IP", //IP或者LB
"INNERLBTYPE":"ROUNDROBIN",
"IPAddresses":[]
```
如果type是"IP",则直接根据INNERLBTYPE选取一个IP;如果type是"LB",同样根据INNERLBTYPE获取一个LB的IP,然后继续从LB中获取IP地址并且连接.

INNERLBTYPE表示客户端本地选取IP的方式


