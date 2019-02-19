---
title: Redis 懒删除(lazy free)简史
date: 2018-12-15 13:29:12
tags: Redis
---
## Redis是单进程单线程模式吗

下图为Redis5.0启动之后的效果。LWP为线程ID，NLWP为线程数量。可以看到，5.0的redis server共有四个线程，一个主线程48684，三个bio(background IO,后台io任务)线程，三个后台线程分别执行不同的io任务，我们重点考察删除一个key时的io线程执行。

![process](/img/rl1.png)

Redis增加了异步删除命令unlink,防止删除大key时阻塞主线程。其原理为执行unlink时会将需要删除的数据挂到一个链表中，由专门的线程负责将其删除。而原来的del命令还是阻塞的。我们通过对一个有1000万条数据的集合分别执行del和unlink来观察其效果。

## 看一个大集合的删除
首先通过脚本生成一个有1000万个元素的集合testset，然后通过del命令删除，如下：

```
127.0.0.1:8888>info//首先调用info命令查看内存消耗：
 
# Memory
used_memory:857536
used_memory_human:837.44K
 
127.0.0.1:8888> eval "local i = tonumber(ARGV[1]);local res;math.randomseed(tonumber(ARGV[2]));while (i > 0) do res = redis.call('sadd',KEYS[1],math.random());i = i-1;end" 1  testset 10000000 2
(nil)
(18.51s)//创建耗时18.51s 
 
127.0.0.1:8888>info//再次查看内存消耗
# Memory
used_memory:681063080
used_memory_human:649.51M

127.0.0.1:8888> scard testset//查看集合中元素数量
(integer) 9976638 //通过math.random()生成，由于集合中不能有重复数据，可以看到，最终只有9976638条数据不重复。
127.0.0.1:8888> sscan testset 0 //查看集合中的元素内容
1) "3670016"
2)  1) "0.94438312106969913"
    2) "0.55726669754705704"
    3) "0.3246220281927949"
    4) "0.51470726752407259"
    5) "0.33469647464095453"
    6) "0.48387842554779648"
    7) "0.3680923377946449"
    8) "0.34466382877187052"
    9) "0.019202849370987551"
   10) "0.71412580307299545"
   11) "0.12846412375963484"
   12) "0.10548432828182557"

127.0.0.1:8888> del testset //调用del命令删除，耗时2.76s 
(integer) 1
(2.76s) 
 
127.0.0.1:8888>info//再次查看内存消耗
# Memory
used_memory:858568
used_memory_human:838.45K
```

重新做上边的实验,这次试用unlink来删除。

```

127.0.0.1:8888> unlink testset//unlink瞬间返回
(integer) 1
127.0.0.1:8888>info//再次查看内存消耗。可以看到，返回之后testset并没有清理干净。内存仍然占用了大约一半，再经过1-2s,会清理干净
# Memory
used_memory:326898224
used_memory_human:311.75M
```

## 尝试渐进式删除
参见:http://antirez.com/news/93

为了解决这个问题，Redis作者Antirez首先考虑的是通过渐进式删除来解决。Redis也在很多地方用到了渐进式的策略，例如 lru eviction,key 过期以及渐进式rehash.原文如下：

```
So this was the first thing I tried: create a new timer function, and perform the eviction there. Objects were just queued into a linked list, to be reclaimed slowly and incrementally each time the timer function was called. This requires some trick to work well. For example objects implemented with hash tables were also reclaimed incrementally using the same mechanism used inside Redis SCAN command: taking a cursor inside the dictionary and iterating it to free element after element. This way, in each timer call, we don’t have to free a whole hash table. The cursor will tell us where we left when we re-enter the timer function.
```

大意就是把要删除的对象放到一个链表中，起一个定期任务，每次只删除其中一部分。

这会有什么问题呢，仍然看原文中说的一种案例:

```
    WHILE 1
        SADD myset element1 element2 … many many many elements
        DEL myset
    END
```

如果删除没有增加快，上边这种案例会导致内存暴涨.(虽然不知道什么情况下会有这种案例发生)。于是作者开始设计一种自适应性的删除,即通过判断内存是增加还是减少，来动态调整删除任务执行的频率，代码示例如下：


```
 /* Compute the memory trend, biased towards thinking memory is raising
     * for a few calls every time previous and current memory raise. */
	
	//只要内存有一次显示是增加的趋势，则接下来即使内存不再增加，还是会有连续六次mem_is_raising都是1，即判断为增加。
	//注意mem_is_raising的值是根据mem_trend和0.1来比较。 即第一次0.9,第二次为0.9*0.9,第三次为0.81*0.81.第六次之后才会小于0.1  (勘误:应该为0.9^22 之后小于0.1)
	//这也就是上边注释描述的会偏向于认为只要有一次内存是增加的，就会连续几次加快执行调用删除任务的频率
    if (prev_mem < mem) mem_trend = 1; 
    mem_trend *= 0.9; /* Make it slowly forget. */
    int mem_is_raising = mem_trend > .1;

	//删除一些数据
    /* Free a few items. */
    size_t workdone = lazyfreeStep(LAZYFREE_STEP_SLOW);

	//动态调整执行频率
    /* Adjust this timer call frequency according to the current state. */
    if (workdone) {
        if (timer_period == 1000) timer_period = 20;
        if (mem_is_raising && timer_period > 3)//如果内存在增加，就加大执行频率
            timer_period--; /* Raise call frequency. */
        else if (!mem_is_raising && timer_period < 20)
            timer_period++; /* Lower call frequency. *///否则减小频率
    } else {
        timer_period = 1000;    /* 1 HZ */
    }
```

这种方法有个缺陷，因为毕竟是在一个线程中，当回收的特别频繁时，会降低redis的qps,qps只能达到正常情况下的65%.


```
when the lazy free cycle was very busy, operations per second were reduced to around 65% of the norm
```

于是redis作者antirez开始考虑异步线程回收。

## 异步线程
### 共享对象
#### 异步线程为何不能有共享数据
共享数据越多，多线程之间发生争用的可能性越大。所以为了性能，必须首先将共享数据消灭掉。

那么redis在什么地方会用到共享数据呢

#### 如何共享
如下代码示例为Redis2.8.24.

先看执行sadd时底层数据是如何保存的

```
sadd testset val1
```
底层保存如下(gdb过程如下，比较晦涩,参考下文解释)：

```
254	    set = lookupKeyWrite(c->db,c->argv[1]);
(gdb) n
255	    if (set == NULL) {
(gdb) p c->argv[1]
$1 = (robj *) 0x7f58e3ccfcc0
(gdb) p *c->argv[1]
$2 = {type = 0, encoding = 0, lru = 1367521, refcount = 1, ptr = 0x7f58e3ccfcd8}

(gdb) p (char *)c->argv[1].ptr //client中的argv是一个个robj,argv[1]的ptr中存储着key值'testset'
$4 = 0x7f58e3ccfcd8 "testset"
(gdb) n
254	    set = lookupKeyWrite(c->db,c->argv[1]);
(gdb) n
255	    if (set == NULL) {
...
(gdb) p (char *)((robj *)((dict *)set.ptr).ht[0].table[3].key).ptr
$37 = 0x7f58e3ccfcb8 "val1" //值val1保存在一个dict中，dict保存着一个个dictEntry,dictEntry的key是一个指针，指向一个robj,robj中是具体的值
```

通过下文结构体讲解，可以看下sadd testset val1,testset和val1保存在什么地方


```
typedef struct dict {
    dictType *type;
    void *privdata;
    dictht ht[2];
    long rehashidx; /* rehashing not in progress if rehashidx == -1 */
    int iterators; /* number of iterators currently running */
} dict;


 
typedef struct dictht {
    dictEntry **table;
    unsigned long size;
    unsigned long sizemask;
    unsigned long used;
} dictht;

typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
 
typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:REDIS_LRU_BITS; /* lru time (relative to server.lruclock) */
    int refcount;
    void *ptr;
} robj;

```

* 首先所有的key保存在一个dict.ht[0]的dictht结构体中。通过上边的结构体看到，dictht中的table是一个dictEntry二级指针。

* 执行sadd testset val1时，testset是其中一个dictEntry中的key,key是一个void*指针，实际存储情况为testset保存为一个char *类型

* 假设testset经过哈希之后index为3，则dict.ht[0].table[3].key为testset,dict.ht[0].table[3].v.val为一个void*指针，实际存储一个robj *类型

* 第三步中的robj中有个ptr指针，指向一个dict类型。dict中的其中一个entry的key指向另一个robj指针，该指针的ptr指向val

即获取一个值的流程为：

    key -> value_obj -> hash table -> robj -> sds_string
然后看两个共享对象的典型场景：

1.sunionstore命令

看下代码实现：

```

int setTypeAdd(robj *subject, robj *value) {
	...
    if (subject->encoding == REDIS_ENCODING_HT) {
        if (dictAdd(subject->ptr,value,NULL) == DICT_OK) {
            incrRefCount(value);//此处的value值由于是从已存在的集合中直接取出，refcount已经是1，此处并没有新建robj,而是直接将引用计数加1
            return 1;
        }
    } 
	...
}
```

执行以下命令：

sadd testset1 value2

sunionstore set testset1 testset2 //即将testset1和testset2的元素取并集并保存到set中

然后我们可以通过查看testset的元素，看看其引用计数是否变为了2

smembers testset

```

(gdb) p *(robj *)(((dict *)setobj.ptr).ht[0].table[3].key)
$88 = {type = 0, encoding = 0, lru = 1457112, refcount = 2, ptr = 0x7f58e3ccfb68} //refcount为2
 
(gdb) p (char *)(((robj *)(((dict *)setobj.ptr).ht[0].table[3].key)).ptr)
$89 = 0x7f58e3ccfb68 "val"                                  //值为val
```

2.smembers命令

返回元素的时候，重点看返回时的代码

```

/* Add a Redis Object as a bulk reply */
void addReplyBulk(redisClient *c, robj *obj) {
    addReplyBulkLen(c,obj);
    addReply(c,obj);
    addReply(c,shared.crlf);
}
```

会直接将robj对象作为返回参数

并且客户端传入参数也是一个个robj对象，会直接作为值保存到对象中


#### 共享时如何删除
那么，共享对象在单线程情况下是如何删除的呢？

看看del命令的实现

del调用dictDelete，最终调用每个数据类型自己的析构函数

```
dictFreeKey(d, he);
dictFreeVal(d, he);
```
集合类型调用如下函数

```
void dictRedisObjectDestructor(void *privdata, void *val)
{
    DICT_NOTUSED(privdata);

    if (val == NULL) return; /* Values of swapped out keys as set to NULL */
    decrRefCount(val);
}
```

可以看到，只是将值的refcount减1
如何解决共享数据
新版本如何解决了共享数据

还是通过sunionstore和smembers命令看下这两处如何解决共享：

以下代码使用redis 5.0.3版本介绍：

```
void saddCommand(client *c) {
    ...
    for (j = 2; j < c->argc; j++) {
        if (setTypeAdd(set,c->argv[j]->ptr)) added++; //sadd的时候元素也变为了c->argv[j]->ptr,一个字符串
    }
	...
}
 
int setTypeAdd(robj *subject, sds value) {//value是一个sds
    long long llval;
    if (subject->encoding == OBJ_ENCODING_HT) {
        dict *ht = subject->ptr;
        dictEntry *de = dictAddRaw(ht,value,NULL);
        if (de) {
            dictSetKey(ht,de,sdsdup(value));
            dictSetVal(ht,de,NULL);
            return 1;
        }
    }
    return 0;
}
```
增加值的时候已经变为了一个sds.

现在的保存结构为：

    key -> value_obj -> hash table -> sds_string
而返回到客户端的时候也变为了一个sds,如下：

```

addReplyBulkCBuffer(c,elesds,sdslen(elesds));

void addReplyBulkCBuffer(client *c, const void *p, size_t len) {
    addReplyLongLongWithPrefix(c,len,'$');
    addReplyString(c,p,len);
    addReply(c,shared.crlf);
}
```

#### 效果如何
效果如何呢？

首先取值的时候从robj的间接引用变为了一个sds的直接引用。

其次减少了共享会增加内存的消耗，而使用了sds之后，每个sds的内存占用会比一个robj要小。我们看下antirez如何评价这个修改：

```

The result is that Redis is now more memory efficient since there are no robj structures around in the implementation of the data structures (but they are used in the code paths where there is a lot of sharing going on, for example during the command dispatch and replication). 
...
But, the most interesting thing is, Redis is now faster in all the operations I tested so far. Less indirection was a real winner here. It is faster even in unrelated benchmarks just because the client output buffers are now simpler and faster.
```

说了两层意思，一是内存使用更加高效了

二是更少的间接引用导致redis比以前更加快，而且客户端输出更加简洁和快速。

### 异步线程
异步线程的实现以后在详细描述

问题

1.多线程之间在堆上分配内存时会有争用。但是antirez说因为redis在内存分配上使用的时间极少，可以忽略这种情况。

如何考虑这个问题？

参考：https://software.intel.com/zh-cn/articles/avoiding-heap-contention-among-threads