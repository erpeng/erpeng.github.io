---
title: Redis Replication,fullsync以及psync
date: 2019-11-13 
tags: Redis
---
>具体参考链接见下文,本文对redis作者思考replication的过程做一个复盘

## psync

主从之间同步时,主相当于一个从的client,从只需要接收命令并执行.并且重传不会导致数据不一致,简单而可信赖(Simple and reliable).


在内存很昂贵的情况下,即使因为网络抖动导致重连从而执行full sync,重传RAM-sized大小的数据是可接受的.但随着内存变的便宜,大量用户开始存储大量数据到内存的情况下,full sync开始变得昂贵

重传还会涉及到一些I/O操作,例如RDB文件生成和加载.因此考虑PSYNC

但显然,如果保存所有命令到一个磁盘文件,每次都可以重传也是不可接受的.这涉及到大量的I/O操作,毕竟redis是一个内存型的数据库.

因此psync需要满足两个条件:
* 在一定时间内重连
* 主没有重启

psync在redis2.8版本内将会引入

## 探索同步复制

Redis 2.6之前主会向从定时发送ping,防止从检测到超时.这种情况下有些网络链接异常的错误检测不到,导致从的output buffer中堆积大量的命令消耗内存.因此需要从也定时ping 主.但作者认为只发送ping没有意义,因此引入了 replconf ack <replication-offset>这个命令

进一步的,如果主能感知从的同步状态的话,是不是很容易就能实现一个同步复制的命令了？作者之前思考的是
```
MINRELICAS <count> <min-idle>
```
类似blpop这种阻塞命令,在min-idle的超时时间内等待至少cout个replica复制成功才返回.是不是感觉很熟悉,这就是后来的wait命令.


## 再谈psync

从Redis4.0开始,当一个实例因为failover切换为主之后,仍旧能够向旧主的replicas执行部分重同步.因为replica中会记录旧主的replication ID和offset,**并且每个replica默认也会开启backlog**

而且,replicas重启时会将replication ID和offset记录到rdb文件中.因此升级时可以先执行shutdown执行一个save&quit的流程,然后启动,仍然可以执行psync

但是注意,如果只开启了AOF或者是从AOF恢复的数据,就不能够执行psync,因为AOF中并没有记录replication ID 和 offset,因此重启时可以先把持久化切换为RDB才可以.


##  参考链接
* http://antirez.com/news/47
* http://www.antirez.com/news/58
* https://redis.io/topics/replication
