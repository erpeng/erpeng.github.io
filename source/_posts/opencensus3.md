---
title: OpenTracing and OpenCensus-3
date: 2019-10-29
tags:
 - go
 - trace
 - metric
---

>本文关注metrics/trace/log的收集方式.通过三个常用的开源工具了解其实现原理

## prometheus

![prometheus](/img/op1.png)

脚本类工作(Short-lived jobs)可以通过Pushgateway收集,daemon可以直接通过引入client端收集并对外提供接口.Prometheus server可以通过定期调用接口获取metrics并且存储到时序数据库(TSDB)中

Prometheus web UI或者Grafana或者API通过PromQL调用并且可视化数据

AlertManager可以通过push的方式触发告警

## jaeger

![jaeger](/img/op2.jpg)
jaeger的各种语言client实现了OpenTracing定义的接口.首先发送到本地的agent,agent需要部署到所有的宿主机上.agent发送到collector,collector做存储,ui做展示


## logstash

logstash grok filter可以将非结构化的数据解析为结构化的数据
具体原理为通过正则匹配捕获之后放置到不同的字段中 

```
2016-07-11T23:56:42.000+00:00 INFO [MySecretApp.com.Transaction.Manager]:Starting transaction for session -464410bf-37bf-475a-afc0-498e0199f008

```
例如如上日志需要结构化为timestamp, log level, class, and then the rest of the message

grok规则如下:

```
grok {
   match => { "message" => "%{TIMESTAMP_ISO8601:timestamp} %{LOGLEVEL:log-level} \[%{DATA:class}\]:%{GREEDYDATA:message}" }
 }
```

常见模式定义查看该链接:https://github.com/logstash-plugins/logstash-patterns-core/blob/master/patterns/grok-patterns

grok也可以增加其他配置去操纵数据,参考:https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html#_synopsis_125




## 链接
* https://logz.io/blog/logstash-grok/
