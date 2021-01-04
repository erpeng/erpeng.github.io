---
title: MQTT学习笔记
date: 2021-01-04 
tags:
 - Message Queue
---


## Introduction
 
关键特性:bandwidth-efficient,lightweight,high latency networks
因为这些特性,所以MQTT很适合物联网的环境,从技术角度来说,消息队列有很多,为何MQTT会有这些特性呢？进一步,如何设计一个协议能达到这些特性.
* 加密端口8883,非加密端口1883
* pub/sub模式,包括客户端和Message broker,默认使用TCP传输信息
* 如果没有订阅者,broker丢弃消息,除非publisher指定消息为retained.但broker只会保存最近一条消息(消息可丢失,适用于类型重复并且大量的消息)
* 有14种消息类型,每个消息由control message和数据组成,control message最小可以只有2个字节,data可以有256M字节(报文control message确实比较轻量)
* MQTT-SN,变种,可以通过UDP或者蓝牙传输信息

MQTT v5.0
* ACK信息支持返回codes,能够提供失败原因
* 订阅者能够进行负载均衡
* 消息过期时间,过期之后被删除
* Topic别名:Topic能够被一个数字代表(减少流量)

QOS:
* At most once:不用ack,fire and forgot
* At least once:一直重试直到收到ack
* Exactly once:只有一份数据会被收到 (ssured delivery)

## 协议分析

### 结构

Structure |结构 |
--------------- |
Fixed header, present in all MQTT Control Packets | 固定长度头部
Variable header, present in some MQTT Control Packets | 可变长度头部
Payload, present in some MQTT Control Packets | Payload


### 固定头部

第一个字节:
前4bit表明包类型,后4bit根据不同的包类型可以置不同的flag
0,15两种类型保留,因此共有2^4-2=14种包的类型,参考链接2 Table2.1和Table2.2

2-5字节(Remaining Length):
2-5字节表示可变头部+Payload的大小,注意该部分会编码为一个可变长度模式,最多4字节,0x00-0xFFFFFF7F(最大表示256MB),最少1字节,0x00-0x7F(最大表示127).可变长度模式低7位编码数据,最高位表示是否有后续字节.比较通用的一种方式,详情参考链接2 2.2.3

### 可变头部

某些MQTT包会包括可变头部

#### Packet Identifier
两字节,当subscribe,unsubscribe,publish(Qos>0)必须包括一个16bit的非空标识符,如果一个客户端重发一个包,需要使用相同的标识符
Publish Qos为0时不需要包括标识符

puback,pubrec(receive),pubrel(pubrelease)包必须包含和publish相同的标识符.suback和unsuback也必须和subscribe,unsubscribe相同的标识符

### Payload

某些MQTT包会包括Payload

## 包分析

### CONNECT
客户端进行连接时必须发送的第一个包

可变头部:

* 可变头部为10字节,头6字节是UTF-8编码,包括两字节的长度字段(长度为4)+ "MQTT"这个协议名称.通过该值可以检测一个流是否为MQTT协议
* 第7个字节为Protocol Level,version 3.1.1的该字段为 (0x04).CONNACK返回码为0x01时说明该协议版本不被server支持
* 第8个字节为Connect Flags,可以用来控制server的行为或者指示Payloads内容,由低位到高位分别解释如下:
	* 0bit:reserved
	* 1bit:clean session 会话管理的生命周期.设置为0时会收到所有QOS为1和2的消息,即使客户端在该消息发布时已经断开.clean  session为0时服务端还会保存客户端的订阅情况.因此如果想保证客户端断开后不丢消息,可以将QOS设置为1或者2,并且clean session设置为0
	* 2bit:will flag  遗愿标志,设置为1表明一个连接建立后,和连接相关的一个will message(遗愿信息)必须保存到服务端.之后如果客户单没有发送disconnect就断开,那么遗愿信息需要发送到指定的topic中.will topic和will message必须在payload中出现.会发送遗愿信息的情况包括:服务端发现了网络错误或者I/O错误,客户端在keep alive 时间内没有发送心跳,客户端未发送disconnect就关闭了连接以及服务端因为协议错误关闭了连接.will message发送之后就会删除或者是客户端发送disconnect后删除
	* 3-4bit:will qos will flag为0时该值为0x00,will flag为1时可以为0x00,0x01,0x02
	* 5bit: will retain 决定一个will message publish之后是否继续retained
	* 6bit: password flag 为0表明payload中没有密码,为1表明有密码.如果User Name Flag为0,则password flag必须为0
	* 7bit: user name flag 参考password flag
* 9-10字节:keep alive interval,如果该时间内仍未有包传输,客户端应该传输一个PINGREQ包并且收到相应的PINGRESP包,以此来进行两端的探活.最大值为18hours12minutes15seconds.可以不设置

Payload:

length-prefixed fields,出现顺序为Client Identifier,Will Topic,Will Message,User Name,Password

* Client Identifier:1-23字节长的UTF-8字符串,包括数字和大小写字母.如果服务端拒绝了一个ClientId,CONNACK返回0x02(Identifier rejected)
* Will Topic:UTF-8编码的字符串,根据Will Flag是否设置为1决定是否有该字段
* Will Message:Will Flag设置为1后有该字段.决定发送给Will Topic的消息.最长65535字节
* User Name: User Name Flag设置为1时有该字段
* Password:同 User Name

### CONNACK

可变头部
* 1byte(connect acknowledge flags):7-1都是0,reserved,0表示session present flag,即session是否存在
* 2byte(connect return code):0x00 connection accepted,0x01 connect refused,unacceptable protocol version,0x02 connection refused,identifier rejected,0x03 server unavailable,0x04 username or password malformed,0x05 not authorized

Payload:
no payload

（未完待续)

## 参考链接
* https://en.wikipedia.org/wiki/MQTT
* http://docs.oasis-open.org/mqtt/mqtt/v3.1.1/os/mqtt-v3.1.1-os.html






