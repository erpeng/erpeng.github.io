---
title: Redis单机版本框架
date: 2018-06-07 13:07:29
tags: Redis
---
## Redis主流程伪代码
```
def main():
 init_server()
 
 while server_is_not_shutdown():
       time = aeSearchNearestTimer()
       beforeSleep()
       aeApiPoll(time)
       processFileEvents()
       processTimeEvents()
 
 clean_server()
 ```

## Redis main函数调用流程图及关键节点
![call](/img/rs1.jpg)

## 一条简单的set命令的执行流程

![flow](/img/rs2.jpg)

## serverCron函数的功能

![flow](/img/rs3.jpg)

## Q&A
1.bgsave执行时再次执行bgsave如何处理？

直接返回,返回信息会通知正在执行.

如果在aof rewrite时执行bgsave,会直接返回不能执行.

看代码此处应该有bgsave schedule命令,如果此时在执行aof rewrite,则会在aof结束后在serverCron中执行。

代码如下 

![flow](/img/rs4.png)

2.aof rewrite正在执行时再次发送bgrewriteaof会如何处理?

直接返回,返回信息通知正在执行

如果此时在执行rdbsave,则会在serverCron中在rdbsave结束之后执行aof rewrite.

代码如下:

![flow](/img/rs5.png)

3.bgsave时如果master还在执行写入,由于linux COW机制,此时会给子进程拷贝一份数据,导致双倍内存。

 待在测试环境验证是否会出现

4.client端发送的命令能否在server端保证顺序?

5.为什么redis本身支持分布式生产环境还在使用codis？