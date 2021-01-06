---
title: MQTT GO客户端实现
date: 2021-01-06 
tags:
 - Message Queue
 - GO
---


## Introduction
上一篇文章通过spec对mqtt有了基本了解,接着通过一个mqtt go客户端的代码,看看具体的工程实现细节.具体代码参考链接

## 代码概览
首先看看代码结构:
```
ZhangShihua:paho.mqtt.golang zhangshihua$ find . -name "*.go"|grep -v 'test'|grep -v 'cmd' | xargs wc -l |sort -nr
    4864 total
    1055 ./client.go
     464 ./net.go
     383 ./options.go
     346 ./packets/packets.go
     257 ./filestore.go
     200 ./token.go
     182 ./router.go
     176 ./messageids.go
     167 ./options_reader.go
     151 ./packets/connect.go
     138 ./memstore.go
     136 ./store.go
     127 ./message.go
     109 ./websocket.go
      91 ./netconn.go
      86 ./topic.go
      83 ./packets/publish.go
      74 ./ping.go
      69 ./packets/subscribe.go
      57 ./packets/suback.go
      56 ./packets/unsubscribe.go
      52 ./packets/connack.go
      42 ./packets/unsuback.go
      42 ./packets/pubrel.go
      42 ./packets/pubrec.go
      42 ./packets/pubcomp.go
      42 ./packets/puback.go
      40 ./trace.go
      34 ./packets/pingresp.go
      34 ./packets/pingreq.go
      34 ./packets/disconnect.go
      32 ./components.go
      21 ./oops.go
```
直观上思考,首先必须有网络连接层,处理TCP/WS/WSS/TLS之类的连接细节,接着需要具体的mqtt 14种报文解析,QoS1和2级别需要存储消息,因此也需要一个存储层.
* 网络连接层:netconn.go,websocket.go,net.go
* 报文解析:packet目录下是具体的报文编解码代码
* 存储层:store.go,memstore.go,filestore.go
其他代码数最多的client.go对外提供接口,token.go处理返回值,router.go通过topic匹配去寻找对应的handler,用来处理接收到的publish消息

## 分层解析
下边逐层解析相应的代码
### 网络层
websocket.go暴露一个新建websocket(NewWebsocket)函数,之后可以通过返回的连接进行读写
netconn.go是提供一个工厂方法,通过url schema判断返回何种类型的连接,schema可以为:
* ws/wss
* mqtt/tcp
* unix
* ssl/tls/mqtts/mqtt+ssl/tcps

net.go是在连接上层具体处理MQTT进出报文的逻辑
* ConnectMQTT函数:在连接层之上开始发送CONNECT报文并且接收CONNACK报文.CONNECT可变头部（3.1.1版本)前6字节为:0x0004(2字节长度),MQTT(4字节协议名称),第7字节为protocol level,该客户端库中会处理如下协议:
	* 3.1 protocol level为3,protocol name 为 MQIsdp
	* 3.1b protocol level为0x83,protocol name 为 MQIsdp
	* 3.1.1b protocol level为0x84,protocol name 为 MQTT
	* 3.1.1 protocol level为4,protocol name 为 MQTT
	CONNACK第一字节最低位返回session是否存在,第二字节返回连接的return code
* startIncomingComms函数:处理接收到的包和存储中获取的包,起单独的goroutine进行处理,返回一个管道,管道中包括需要进一步处理的包.处理逻辑为:
	* 通过读取报文的4-7bit决定包类型,并且解析为相应的包
	* 判断是否需要持久化包并且更新最后收到包的时间
	* 根据包类型决定下一步的处理逻辑,例如如果收到了publish包需要传递给应用层,收到了pubrec包则需要回复pubrel包,收到了pubrel包则需要回复pubcomp包.
* startOutgoingComms函数:处理需要发送的包
	* 发送的包有两个来源,一是本身发出的,一种是收到包后需要回复的包,例如收到一个publish并且qos为1,则需要回复一个puback包
	* 发送包之后更新最后发送包的时间戳

### 报文解析
package.go定义了报文的接口,报文类型以及编解码,接口定义如下:
```
type ControlPacket interface {
	Write(io.Writer) error
	Unpack(io.Reader) error
	String() string
	Details() Details
}
```
每种类型的报文自定义自己的Write函数-组装报文并且发送,Unpack函数-解析收到的报文
例如puback。go中对puback报文的解析.固定头部第二字节即RemainLength是2,表明可变头部加payload共两字节,可变头部两字节为client Identifier,payload为空,代码如下:
```
func (pa *PubackPacket) Write(w io.Writer) error {
	var err error
	pa.FixedHeader.RemainingLength = 2
	packet := pa.FixedHeader.pack()
	packet.Write(encodeUint16(pa.MessageID))
	_, err = packet.WriteTo(w)

	return err
}
```
代码逻辑为将固定头部打包然后将client Idenfier编码为2字节,写入即可
固定头部pack方法如下:
```
func (fh *FixedHeader) pack() bytes.Buffer {
	var header bytes.Buffer
	header.WriteByte(fh.MessageType<<4 | boolToByte(fh.Dup)<<3 | fh.Qos<<1 | boolToByte(fh.Retain))
	header.Write(encodeLength(fh.RemainingLength))
	return header
}
```
并不是所有报文的固定头部都有dup以及qos和retain字段,此处代码会根据实际报文情况,选择性赋值给这三个字段(除了publish报文其他的赋值不代表这三个字段的真实含义)

puback的Unpack代码如下:
```
func (pa *PubackPacket) Unpack(b io.Reader) error {
	var err error
	pa.MessageID, err = decodeUint16(b)

	return err
}
```
解析出client identifier即可

## 存储层
store.go定义存储接口,如下:
```
type Store interface {
	Open()
	Put(key string, message packets.ControlPacket)
	Get(key string) packets.ControlPacket
	All() []string
	Del(key string)
	Close()
	Reset()
}
```
filestore.go以及memstore.go分别实现文件存储以及内存存储格式.
store.go中比较关键的是QoS为1和2时需要将相应的报文进行保存,方法参考persistOutbound以及persistInbound,这两个方法中会进行消息的保存和删除操作

## 处理响应
token.go用来处理响应包,客户端调用publish,subcribe之后都会返回一个token.因为publish并且QoS为1或者2时broker会回复puback或者pubrec包,subscribe之后会回复suback包.网络层有响应包之后会修改token状态,从而达到通知应用层的目的
```
type Token interface {
	Wait() bool

	WaitTimeout(time.Duration) bool
	Done() <-chan struct{}

	Error() error
}

type TokenErrorSetter interface {
	setError(error)
}

type tokenCompletor interface {
	Token
	TokenErrorSetter
	flowComplete()
}
```
接口和实现都比较简单,不再赘述

## 接收publish包
订阅之后需要根据topic调用相应的应用层处理,该代码位于router.go中.处理订阅消息的方法为matchAndDispatch,该文件中还包括了注册topic以及相应处理器,删除topic等一系列操作.底层数据结构为一个双向链表,每次处理都会全部遍历,因此topic不宜过多


## 后记
整体代码结构比较清晰,每个文件中的类型基本都是interface,可测试性好并且易于扩展.有两处实现需要注意:
* client idenfier同时只能有65535个,会释放和重复使用
* topic为双向链表,订阅之后每次都需要全部遍历
* 报文编码默认固定头部都会有dup,qos,retain字段,这样写起来比较统一,但实际报文并不是全部都有此类字段,有些易混淆


## 参考链接
* https://github.com/eclipse/paho.mqtt.golang






