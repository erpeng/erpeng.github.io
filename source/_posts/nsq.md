---
title: nsq
date: 2020-01-19 
tags:
 - nsq
 - go
---

> 版本说明:nsq v1.2.0版本.

## 功能 

### 安装
mac:
```
 $ brew install nsq
```

### 测试
默认是foreground模式,需要在不同的shell下分别执行如下命令

```
$ nsqlookupd //监听 tcp 4160,http 4161
$ nsqd --lookupd-tcp-address=127.0.0.1:4160 //监听tcp 4150,htp 4151
$ nsqadmin --lookupd-http-address=127.0.0.1:4161  //监听htp 4171,通过http://127.0.0.1:4171/ 打开UI管理面板
$ curl -d 'hello world 1' 'http://127.0.0.1:4151/pub?topic=test' //topic test下publish消息
$ nsq_to_file --topic=test --output-dir=/tmp --lookupd-http-address=127.0.0.1:4161 //可在/tmp目录下查看持久化文件内容
$ curl -d 'hello world 2' 'http://127.0.0.1:4151/pub?topic=test' //topic test下publish消息
$ curl -d 'hello world 3' 'http://127.0.0.1:4151/pub?topic=test' //topic test下publish消息

```

### Guarantees

* 默认不持久化,可配置
* delive-policy:at-least-once,客户端自己保证幂等性
* push模式
* 消息无序
* CAP中占CA,保证最终一致性

### design

![nsqd](/img/nsqd.gif)

* topic描述数据,channel描述下游consumer的功能
* topic->channel,多播,一个topic中所有channel中都有相同的全量信息
* channel->consumers,负载均衡
* nsqd:服务节点,保存topic和data
* nsqlookupd:服务发现节点,client通过nsqlookupd发现某个topic属于哪个nsqd.启动多个但不会互相传递信息,只保证最终一致性
* nsqadmin:可视化查看服务和消息 
* at-least-once:通过客户端发送FIN(finish)或者REQ(re-queue)确认是否处理成功.否则放入requeue继续重试
* --mem-queue-size通过该参数控制内存占用

![flow](/img/nsqd_flow.png)
* 通过客户端flow control,更新RDY状态控制nsqd push消息的条数


## 代码解析

**nsq中的channel写为channel,go语言中的channel写为go-chan**

### 结构体分析
关键结构体为:
* NSQD
* Topic
* Channel
* Message

NSQD管理Topic以及监听tcp和http端口,Topic管理Channel,Channel管理message.

#### NSQD
![NSQD](/img/nsqd_struct.png)
属性和方法见名思义,关键介绍一下queueScanLoop方法.
每个channel中有两个优先级队列,分别处理在途消息的超时和延迟消息的传递.queueScanLoop在一个单独
的goroutine中执行,模仿redis的主动过期算法,每100ms随机选取20个channel查看在途和延迟的优先级队列,如果需要处理的channel个数超出25%,
则继续处理其他channel的在途和延迟队列.否则等待下一个周期


#### Topic&Channel

![TopicChannel](/img/TopicAndChannel.png)
关键介绍Topic的messagePump方法
该方法将一个topic中的所有数据写入该topic下的所有channel中.


#### Message

![Message](/img/Message.png)

Message是每条消息的具体结构.


### 协议解析

context中保存一个NSQD实例指针,protocolV2具体执行协议的解析
协议解析参考链接2

#### protocolV2
![protocalV2](/img/protocolV2.png)


#### protocal

![protocal](/img/protocol.png)


### 关键go-chan



## 参考链接

* https://nsq.io/overview/design.html
* https://nsq.io/overview/internals.html
* https://nsq.io/clients/tcp_protocol_spec.html


