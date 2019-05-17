---
title: 如何设计一个高效的监控系统
date: 2019-05-17
tags: architecture
---

>本文主要关注两点,一为分布式追踪系统的监控设计,一为分布式调用指标的采集.主要思路来源于google dapper与statsd.

## dapper:大规模分布式追踪系统基础架构
* traceId:一次请求的唯一标识,通过traceid可以将该请求所有的调用串联起来
* spanId:我们需要知道调用方与被调用方,因此需要spanId来区分.每个调用方生成自己的spanId,传递给被调用方,被调用方将该spanId作为自己的parentId.
* 通过traceId与spanId(parentId)可以将调用关系生成一颗树,其中根节点的parentId为null.

实际生产中需要考虑如下问题:
* traceId与spanId的生成.需要高效并且唯一
* 对应用透明
* 开销要小

采集和汇总工具:
dapper会将traceId,spanId(parentId)打印到日志中,通过一个采集器采集上报,放到一个bigtable中汇总分析.

## statsd:简单但强大的指标聚合工具

在<<如何设计一个高并发的日志系统>>一文中,我们提到可以通过udp将日志输出.这种方式还有一个好处是,udp server可以实现一些逻辑,做一些汇总统计的工作.

statsd就是通过这种方式进行数据采集.statsd主要是进行应用层面的数据采集,应用层每次进行rpc调用时可以通过一个简单的协议将调用结果或者调用延时通过udp发送到statsd,statsd统计汇总之后上报展示.

statsd在一个周期范围内(10-60s,可配置)统计两类指标,每次周期结束后将统计清零然后重新开始下一个周期

* 计数:例如20s内该rpc调用共执行了多少次,失败多少次
* 计时:例如20s内该rpc调用每次耗时的99分位,50分位,最低最高值


## 参考链接
* http://code.flickr.net/2008/10/27/counting-timing/
* https://static.googleusercontent.com/media/research.google.com/zh-CN//archive/papers/dapper-2010-1.pdf