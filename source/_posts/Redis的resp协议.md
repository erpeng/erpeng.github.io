---
title: Redis的resp协议
date: 2019-01-31 14:03:45
tags: Redis
---

## resp协议
redis客户端和服务端交互使用的是redis作者制定的一个协议，叫resp(REdis Serialization Protocol)。

具体分如下几个层次

* 基于tcp
* 请求响应模式,但在两种情况下不再是简单的请求和响应模式(下文介绍)
* 支持五种类型的数据，分别是简单字符串，错误，整型，bulk strings ,数组

客户端发给服务端的命令都会序列化为array,而服务端返回给客户端的可以为如上任意一种类型，各简单举例如下

* 简单字符串,第一个byte为 '+',例如  "+OK\r\n"
* 错误，第一个byte为'-',例如 "-Error message\r\n"
* 整型, 第一个byte为':",例如 ":10\r\n"

* bulk strings,第一个byte为"\$",例如 "\$6\r\nfoobar\r\n"(此处为美元符号，没有前边的反斜杠)
* 数组,第一个byte为'*',例如"*2\r\n$3\r\nfoo\r\n$3\r\nbar\r\n"
具体介绍参考:https://redis.io/topics/protocol

## sub pattern
请求响应模式有两种特殊情况

* pipeline模式，客户端同时发送多条命令到服务端

* sub pattern:当客户端执行subscribe命令后,不再要求客户端发送命令，当有其他客户端在订阅渠道上publish消息后,服务端会主动push信息到客户端

我们拿redis-cli客户端试一下执行subscribe
```
127.0.0.1:6379> subscribe foo
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "foo"
3) (integer) 1
```

可以看到,redis-cli会阻塞，当另起一个客户端,publish消息后，会收到该消息并打印
```
xiaoju@zsh_test ~$redis-cli
127.0.0.1:6379> publish foo bar
(integer) 1
127.0.0.1:6379>


xiaoju@zsh_test ~$redis-cli
127.0.0.1:6379> subscribe foo
Reading messages... (press Ctrl-C to quit)
1) "subscribe"
2) "foo"
3) (integer) 1
1) "message"
2) "foo"
3) "bar"
```

直观感觉是服务端阻塞了，没有返回数据给客户端。但看redis源码，实际服务端并没有阻塞，并且可以在连接上继续接收并处理命令
```
void subscribeCommand(client *c) {
    int j;
	//将订阅的渠道加入相应结构体并直接返回
    for (j = 1; j < c->argc; j++)
        pubsubSubscribeChannel(c,c->argv[j]);
    //将客户端置CLINET_PUBSUB标记
    c->flags |= CLIENT_PUBSUB;
}
```
当客户端置了CLINET_PUBSUB标记后,命令处理会做如下特殊逻辑
```
int processCommand(client *c) {
	...
	//当置CLIENT_PUBSUB标记后,只有ping/subscribe/unsubscribe/psubscribe/punsubscribe命令能够执行
	if (c->flags & CLIENT_PUBSUB &&
    	c->cmd->proc != pingCommand &&
    	c->cmd->proc != subscribeCommand &&
    	c->cmd->proc != unsubscribeCommand &&
    	c->cmd->proc != psubscribeCommand &&
    	c->cmd->proc != punsubscribeCommand) {
   	 	addReplyError(c,"only (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT allowed in this context");
   	 	return C_OK;
	}
 	...
}
```

godis-cli
如上文所示,当客户端执行subscribe命令后,实际上是可以继续订阅或者取消订阅渠道，并且可以执行ping命令。redis-cli客户端实际上是自己阻塞了，如下代码：
```
if (config.pubsub_mode) {
    if (config.output != OUTPUT_RAW)
        printf("Reading messages... (press Ctrl-C to quit)\n");
	//进入死循环，一直等待服务端发送消息
    while (1) {
        if (cliReadReply(output_raw) != REDIS_OK) exit(1);
    }
}
```
那么，我们可以拿go写一个不阻塞的版本，并且可以测试redis的subscribe 模式。效果如下
```
>bogon:godis-cli didi$ go run godis-cli.go
> set k1 v1 
OK
> get k1
v1
> subscribe foo
subscribe foo 1
[sub]>subscribe foo1//sub模式下可以继续订阅其他渠道
subscribe foo1 2
[sub]> unsubscribe foo1//取消订阅
unsubscribe foo1 1
[sub]> ping//sub模式也可以执行ping
pong
[sub]> get k1 //sub模式下不能执行get命令
Redis Error: only (P)SUBSCRIBE / (P)UNSUBSCRIBE / PING / QUIT allowed in this context
[sub]> exit
exit sub pattern....
>get k1//退出sub模式后可以继续正常执行get命令
v1
> exit

```
由于godis-cli直接实现了resp协议，虽然只是为了观察subscribe pattern的效果，但实际上可以支持所有redis命令的执行

具体代码地址见:https://github.com/erpeng/godis-cli