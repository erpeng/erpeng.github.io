---
title: OpenTracing and OpenCensus-1
date: 2019-10-18
tags:
 - go
 - trace
 - metric
---
## Metrics/Tracing/Logging

Metrics:聚合指标,例如http请求数是一个counter,队列长度是一个gauge,请求时间是一个histogram
Logging:离散的events
Tracing:request-scoped.一个请求范围内的调用关系

OpenTracing通过一个spc定义了实体和实体能力,解耦应用和后端tracing系统.

## OpenTracing Specification

### 模型

#### span定义
链路通过span定义,每个span包括如下状态:
* 名称
* 开始时间
* 结束时间
* 0或多个Span Tags
* 0或多个Span Logs.key/value pairs附加一个时间戳
* 一个SpanContext.
* **0或多个关联的Span**
SpanContext包括如下状态:
* trace 和 span ids
* **Baggage items:key/value paires**

标准的Span Tags如下:

基础tag:
* component: "grpc" or "django" or "JDBI"
* error: "true" or "false"
* span.kind :"client" or "server" or "producer" or "consumer"
* sampling.priority : 0 or 1 or 2
* peer.ipv4 : "127.0.0.1"
* peer.hostname : "ke.com"

db相关:
* db.instance: "db1" or "db2"
* db.type : "redis" or "mysql"
* db.statement : "set mykey myvalue" or "select * from tb1 where id =1 "
* db.user : "readonly" or "writer_user1"

http相关:
* http.method :"get" or "post"
* http.status_code : 200 or 404 or 503
* http.url : "https://ke.com"

mq相关:
* message_bus.destination : "tpoic name"

标准的:Log Fields

Log除了一个时间戳外有如下一些标准fields:

* error.kind : "Exception" or "OSError"
* error.object : "java.lang.UnsupportedOperationException" or "exceptions.NameError"
* event: "initialized" or "timed out"
* message: "Could not connect to backend" or "Cache invalidation succeeded"
* stack:  调用栈

#### span间的关系

ChildOf:parent span依赖于child span时是child of 关系.例如:
* server 端rpc的span与客户端的一个rpc调用是child of
* sql insert的一个span与orm中的save方法关系是child of
* 一个父span并发执行许多子span,然后合并子请求的结果,关系也是child of

followsfrom:父span不依赖于子span的结果.
例如生产者消费者关系

### API

有三个关键概念:Tracer,Span,SpanContext

Tracer有如下能力:

* 开启一个Span
* 将一个SpanContext注入一个载体
* 将一个SpanContext从一个载体中解析出

载体可以有如下几种类型:

* Text Map:key/value
* HTTP Headers
* Binary

Span:
* 覆盖span名字
* finish span
* set span tag
* set log fields
* **set/get baggage item**

SpanContext:

* 遍历baggage items


## 参考连接:

* https://opentracing.io/specification/
* https://opentracing.io/specification/conventions/
* https://github.com/yurishkuro/opentracing-tutorial/tree/master/go
* https://zhuanlan.zhihu.com/p/34318538
* http://peter.bourgon.org/blog/2017/02/21/metrics-tracing-and-logging.html
