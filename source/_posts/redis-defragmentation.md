---
title: Redis 内存碎片
date: 2019-07-22
tags: Redis
---


## 内存碎片
 
举个例子,假设一个Redis实例中共有5GB的数据,然后删除了2GB数据,RSS仍旧可能是5G.也就是说并不是在Redis中将key删除后就会将占用的内存返回给OS.这个特性并不是Redis独有的,因为该特性是由服务端代码使用的内存分配器决定的(libc的malloc,jemalloc,tcmalloc等等,在Linux下,Redis默认使用jemalloc).为什么分配器这么做呢？这是一种必然但同时也是一种优化,必然是因为内存分配都是以page为单位的,如果删除的key所在的page仍然有其他key,很明显这个page是不能被回收的;优化是因为如果此时增加新的key,例如重新增加了2GB的数据,RSS可能仍然为5GB,即可以复用之前标记为删除的内存

## 碎片整理

从Redis4.0开始,引入了一个新的功能,叫做active defragmentation.基本配置如下:
```
//是否开启activedefrag
activedefrag yes

//碎片字节数最低要求,达到该字节之后才开启碎片整理
active-defrag-ignore-bytes 100mb

//碎片化比率,超过该比率才开启碎片整理
active-defrag-threshold-lower 10

//碎片化比率上限,超过该值后会加大碎片整理力度
active-defrag-threshold-upper 100

//碎片整理力度,最低为CPU的5%
active-defrag-cycle-min 5

//碎片整理力度,最高为CPU的75%
active-defrag-cycle-max 75


//最大的set/hash/zset/list中的fields个数,超过该值时不会实时清理

active-defrag-max-scan-fields 1000
```

两个疑问:
* 原理是什么,怎么进行碎片整理?
* 为什么是active defragmentation,active代表什么？上述配置项如何理解?
我们在如下两个章节依次回答这两个问题

### 碎片整理的原理

在4.0之前如果遇到碎片化特别严重该如何处理呢？
其实很简单,Windows系统如何整理硬盘碎片呢？拷贝磁盘中的数据到另一张磁盘然后再拷贝回来即可.同样,在Mysql中如果一个表经过大量的增删改查造成大量的页空洞而导致表空间占用超过实际数据大小的时候,可以使用如下命令收缩表空间.该命令同样是一个新建临时表->拷贝数据->rename临时表为t的过程.
```
alter talbe t engine=InnoDB;
```

以上两个例子都是硬盘碎片整理,Redis需要的是内存碎片整理,可以通过重启Redis实例来达到该目的.重启之后Redis需要加载AOF或者RDB文件来重新构造内存,从而达到碎片整理的目的.

但线上服务重启可不是一个常规操作,有没有不重启就能整理内存碎片的方法呢?参考下节

### active defragmentation

active碎片整理,不需要重启Redis实例,渐进式的逐步整理内存(注意该功能默认关闭,并且只适用于使用jemalloc分配器的情况).

如何实现呢？如果我们能够判断哪块内存碎片化比较严重,那么将该内存中的数据移动到其他内存中,这样是不是就能回收碎片化严重的内存了？我们看下关键代码片段:

```
void* activeDefragAlloc(void *ptr) {
    int bin_util, run_util;
    size_t size;
    void *newptr;
    if(!je_get_defrag_hint(ptr, &bin_util, &run_util)) {
        server.stat_active_defrag_misses++;
        return NULL;
    }
    //run和bin都是jemalloc中的术语,此处判断是否需要进行回收操作
    if (run_util > bin_util || run_util == 1<<16) {
        server.stat_active_defrag_misses++;
        return NULL;
    }

    //重新分配一块空间,并且进行赋值操作
    size = zmalloc_size(ptr);
    newptr = zmalloc_no_tcache(size);
    memcpy(newptr, ptr, size);
    zfree_no_tcache(ptr);
    return newptr;
}
```
首先通过je_get_defrag_hint函数获取值判断是否需要进行回收操作,如果需要,则重新分配空间并赋值后返回新的空间地址

该功能的函数调用链为serverCron()->databasesCron()->activeDefragCycle(),在周期函数serverCron中调用.在activeDefragCycle()中会调用computeDefragCycles(),配置参数基本都在该函数中体现.该函数会控制是否需要进行碎片整理工作,如果需要,最多能够使用多长的时间进行碎片整理.

```
void computeDefragCycles() {
    size_t frag_bytes;
    //碎片率和碎片字节数
    float frag_pct = getAllocatorFragmentation(&frag_bytes);
    //碎片率和碎片字节数是否超过了active-defrag-ignore-bytes和active-defrag-threshold-lower,如果没有超出则不进行整理
    if (!server.active_defrag_running) {
        if(frag_pct < server.active_defrag_threshold_lower || frag_bytes < server.active_defrag_ignore_bytes)
            return;
    }

    /* Calculate the adaptive aggressiveness of the defrag */
    //根据碎片化率计算需要的cpu百分比.
    int cpu_pct = INTERPOLATE(frag_pct,
            server.active_defrag_threshold_lower,
            server.active_defrag_threshold_upper,
            server.active_defrag_cycle_min,
            server.active_defrag_cycle_max);
    cpu_pct = LIMIT(cpu_pct,
            server.active_defrag_cycle_min,
            server.active_defrag_cycle_max);
    if (!server.active_defrag_running ||
        cpu_pct > server.active_defrag_running)
    {
        server.active_defrag_running = cpu_pct;
        serverLog(LL_VERBOSE,
            "Starting active defrag, frag=%.0f%%, frag_bytes=%zu, cpu=%d%%",
            frag_pct, frag_bytes, cpu_pct);
    }
}

```
* 首先获取当前的碎片字节和碎片化率,并判断碎片字节数和碎片率是否超过了active-defrag-ignore-bytes和active-defrag-threshold-lower,如果没有超出则不进行整理
* 根据碎片化率计算需要占用的cpu百分比,计算公式如下,假设frag_pct为40
```
5+(40-10)*(75-5)/(100-10) = 28(按默认配置计算)
公式为:
#define INTERPOLATE(x, x1, x2, y1, y2) ( (y1) + ((x)-(x1)) * ((y2)-(y1)) / ((x2)-(x1)) )
#define LIMIT(y, min, max) ((y)<(min)? min: ((y)>(max)? max: (y)))
```
通过碎片化率计算出碎片整理需要的cpu_pct之后如何计算超时时间呢？
代码如下:
```
    start = ustime();
    timelimit = 1000000*server.active_defrag_running/server.hz/100;
    if (timelimit <= 0) timelimit = 1;
    endtime = start + timelimit;
```
server.active_defrag_runnin即为上文计算的cpu_pct,1000000单位为us,即1s,server.hz决定每秒执行多少次serverCron,1000000/server.hz即为每次执行serverCron的最大时间,乘以cpu_pct/100即为该次碎片整理允许占用的最大时间.

## jemalloc
通过上文所述,获取碎片字节、碎片化率以及判断一个指针所在空间是否需要进行碎片整理都是通过jemalloc相关的函数得知.如有需要,需对jemalloc有一个整体认知.

## 参考链接
* http://download.redis.io/redis-stable/redis.conf
* https://redis.io/topics/memory-optimization





















