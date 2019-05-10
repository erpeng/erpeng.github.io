---
title: Redis连接处理
date: 2019-05-10
tags: Redis
---
本文从网络层视角看一下客户端连接,超时,输入输出缓冲区相关的一些知识

## 连接
客户端连接之后做如下处理:
* 设置socket为non-blocking
* 设置socket为TCP_NODELAY
* 注册读取事件准备读取客户端请求

但如果已经超出maxclients配置的最大连接数,则会发送错误后断开连接
**注意是在连接已经建立之后才检测maxclients,为何不在accept的时候直接比较?**
因为直接断开对客户端不太友好,客户端没法知道到底是什么原因导致的连接不了.该比较在连接之后并且是将client的fd设置为non-blocking之后,这样可以无阻塞的写入客户端错误原因

代码参考:
```
static void acceptCommonHandler(int fd, int flags, char *ip) {
    client *c;
    if ((c = createClient(fd)) == NULL) {
        ...
    }
    ...
    if (listLength(server.clients) > server.maxclients) {
        ...
    }
    ...
}
```
## 处理顺序

redis处理时有如下两点:
* 当客户端socket有新数据时,只执行一次read()的系统调用(不会循环读取直到读取完).防止请求量小的客户端被饿死
* 待确认?

## 最大连接数
不设置maxclients时默认是10000的并发连接数限制
Redis内部需要使用32个文件描述符,如果32+maxclients超出操作系统设置的单进程最大能够打开的文件描述符(检查soft limit),则会在日志中打出

系统层面(Linux)可以按如下两方法设置:
* ulimit -Sn 100000 # This will only work if hard limit is big enough.
* sysctl -w fs.file-max=100000

## 输出缓冲区
Redis中可以设置输出缓冲区的大小:
* hard limit:达到该值直接断开客户端连接
* soft limit:连续时间t内一直大于某个数值m,则断开连接

不同类型客户端的默认值:
* normal clients:不限制
* pub/sub clients: hard limit 32M,soft limit 8M 60s
* slaves: hard limit 256M,soft limit 64M 60s

代码详见:
```
int checkClientOutputBufferLimits(client *c) {
   ...
    if (server.client_obuf_limits[class].hard_limit_bytes &&
        used_mem >= server.client_obuf_limits[class].hard_limit_bytes)
        hard = 1;
    if (server.client_obuf_limits[class].soft_limit_bytes &&
        used_mem >= server.client_obuf_limits[class].soft_limit_bytes)
        soft = 1;

    ...
    if (soft) {
        if (c->obuf_soft_limit_reached_time == 0) {
            c->obuf_soft_limit_reached_time = server.unixtime;
            soft = 0; /* First time we see the soft limit reached */
        } else {
            time_t elapsed = server.unixtime - c->obuf_soft_limit_reached_time;

            if (elapsed <=
                server.client_obuf_limits[class].soft_limit_seconds) {
                soft = 0; /* The client still did not reached the max number of
                             seconds for the soft limit to be considered
                             reached. */
            }
        }
    } else {
        c->obuf_soft_limit_reached_time = 0;
    }
    return soft || hard;
}
```

## 输入缓冲区
硬性限制为1GB,超出之后会断开连接

## 超时
默认没有空闲超时时间,但可以在redis.conf或者config set timeout value
注意超时只针对 normal clients,pub/sub模式下不计算该超时
Redis不会设置定时器去检查并且不会每次都顺序检查,而是增量式的检查.所以超时如果设置为10s,12s之后才断开也是正常的

调用代码详见:
```
int clientsCronHandleTimeout(client *c, mstime_t now_ms) {
    time_t now = now_ms/1000;

    if (server.maxidletime &&
        !(c->flags & CLIENT_SLAVE) &&    /* no timeout for slaves */
        !(c->flags & CLIENT_MASTER) &&   /* no timeout for masters */
        !(c->flags & CLIENT_BLOCKED) &&  /* no timeout for BLPOP */
        !(c->flags & CLIENT_PUBSUB) &&   /* no timeout for Pub/Sub clients */
        (now - c->lastinteraction > server.maxidletime))
    {
        ...
    }
```
此处注意如何实现的增量式检查,代码如下:
```
void clientsCron(void) {
    //每次只遍历iterations次,不会检查所有clients
    while(listLength(server.clients) && iterations--) {
        //rotate clients的链表,将最后一个node放到头部
        listRotate(server.clients);
        head = listFirst(server.clients);
        c = listNodeValue(head);
        //检查是否超时
        if (clientsCronHandleTimeout(c,now)) continue;
        ...
    }
}
```
可以看到每次都不会进行遍历,而是增量式取固定个数去执行检测

## client命令
```
显示客户端状态
client lists
```
* addr 客户端地址
* fd  句柄号
* name 客户端名称-通过client setname设置
* age  连接存在时间
* idle 连接空闲时间
* flags 客户端类型 n代表normal clients
* omem  输出缓冲区内存使用大小
* cmd 客户端最后执行的命令

```
kill一个客户端 
client kill
```


## tcp keepalive
3.2以上版本会设置tcp keepalive ,时间为300s
该机制能够检测出dead peers,并且如果客户端和服务端的中间代理有超时设置,可以避免被断开

## 参考链接

* https://redis.io/topics/clients