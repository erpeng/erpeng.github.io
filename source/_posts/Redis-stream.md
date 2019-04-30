---
title: Redis stream简介
date: 2019-04-30 
tags: Redis
---
## stream简介
append-only mode
数据结构为一个前缀树加listpack,listpack的介绍详见 [Redis的一个历史bug及其后续改进]一文.
前缀树中保存的为ID,ID由两部分组成,毫秒级时间戳+该ms内的递增计数.stream结构可以理解为如下三种模式:
1. 一个sorted set,score是时间,member是一个hash,时间序列存储,可以按时间范围遍历
2. 阻塞模式下,类似一个日志系统,可以按tail -f 查看日志
3. 一个pub/sub模式的消息队列,但是可以将消息分区之后pub给消费组中的不同消费者(负载均衡);会保存数据,不像pub/sub,推送之后就删除了.


## stream实现

### xadd
```
增加条目,并且指定最大条目数.至少保存1000条,当listpack可以回收时才会释放多余的条目
XADD mystream MAXLEN ~ 1000 * ... entry fields here ...
```

### xdel
```
删除指定的stream item.也会等待listpack可以回收时真实释放
XDEL mystream 1526654999635-0
```
### xrange
```
xrange key  start end  count N
start:-,时间戳或者entry id
end:+,时间戳或者entry id
遍历 O(log(N))的时间复杂度,不需要XSCAN
```
### xrevrange
```
查看最后一条数据

XREVRANGE mystream + - COUNT 1
```


### xread
```
读取mystream/otherstream中id大于max-id1/max-id2的entry
XREAD COUNT 2 STREAMS mystream  otherstream max-id1 max-id2

读取mystream,$表明读取从现在开始新产生的entry;阻塞读取,超时时间设置为0(即不超时)
XREAD BLOCK 0 STREAMS mystream $ 

```

### xgroup
```
group的增删改查
在key为mystream的stream上创建一个消费组,名称为mygroup,从当前时间开始最新的entry开始消费
$表示只消费最新消息,0表示消费所有历史消息,或者指定一个entryid,表明从该id开始消费
XGROUP CREATE mystream mygroup $
```

### xreadgroup
```
消费信息
group参数指定消费组,接着是消费者的唯一标识 >表明没有提供给其他消费者消费的信息
如果不是>而是指定某一个entry id,表明要消费的是Alice没有xack的消息,即pending的消息
XREADGROUP GROUP mygroup Alice COUNT 1 STREAMS mystream >

```

### xack

```
确认已经消费了某条信息
XACK mystream mygroup 1526569495631-0
```

### xpending
```
显示出mystream中mygroup消费组pending的消息
XPENDING mystream mygroup
```

### xclaim
```
将大于idle-time并且指定id的entry重新分配给consumer.分配完后idle time会重新计算
XCLAIM <key> <group> <consumer> <min-idle-time> <ID-1> <ID-2> ... <ID-N>

```

### xinfo

```
查看mystream相关的信息
XINFO STREAM mystream

查看mystream中消费组相关的信息
XINFO GROUPS mystream

查看mystream中的消费组mygroup的消费者相关的信息
XINFO CONSUMERS mystream mygroup
```


### xtrim
```
修改mystream的最大保存节点个数为10
XTRIM mystream MAXLEN 10

```
## stream其他
 
* dead letter:根据投递次数,如果投递次数大于某个值可以认为该消息不需要继续处理,可以放入其他的stream中
* xadd 中可以指定maxlen,指明stream中保存的最大条目个数



## 参考文章
* http://antirez.com/news/128
* https://redis.io/topics/streams-intro
