---
title: prometheus
date: 2020-03-02
tags:
 - prometheus
---

## 概览

* 时间序列维度的数据,PromQL语言用来查询,一般通过本地监听http端口,展示面板例如grafana可以来拉取数据
* Go语言开发,生态活跃健康
* 适用于进行指标监控


## 查询展示

语法如下:
```
promhttp_metric_handler_requests_total{code="200"}
```
promhttp_metric_handler_requests_total为一个指标名称(name),大括号中key/value对为标签(label),一个指标名称下可以通过标签区分不同的统计,例如本例中'code'标签对应的值为'200'表示成功,值也可以为499/502/504等.

通过这个计数我们也可以计算每秒钟请求量,例如:

```
rate(promhttp_metric_handler_requests_total{code="200"}[1m])
```
'[1m]'表示最近1分钟内,rate函数求每秒的200个数

## metric类型

* Counter:单调递增的计数,只增不减,重启后重置为0.可以统计例如请求数量
* Gauge:可增可减,例如温度,内存数量,同时并发的请求等
* Histogram:适用于统计请求延迟或者响应包大小这类数据(request durations or response sizes)
            一个直方图metric实际会保存如下几条时序数据:
            ```
            <basename>_bucket{le="<upper inclusive bound>"}
            <basename>_sum
            <basename>_count 等价于 <basename>_bucket{le="+Inf"}
            ```

* Summary:适用于统计请求延迟或者响应包大小这类数据(request durations or response sizes)
          一个Summary实际保存如下几条时序数据:
          ```
          <basename>{quantile="<φ>"}
          <basename>_sum
          <basename>_count
          ```
## job&instances
job表示同一类型的服务,instance表示具体的某个实例
job和instance会作为label由采集服务自动添加



## PromQL
表达式有四种类型:
* Instant vector
    ```
    http_requests_total{environment=~"staging|testing|development",method!="GET"}
    ```
    lable可以用"=","!=","=~","!~".
* Range vector
    ```
    http_requests_total{job="prometheus"}[5m]
    ```
* Scalar 
* String

## Functions


## 参考链接

* https://prometheus.io/docs/prometheus/latest/querying/functions
* https://prometheus.io/docs/concepts/metric_types/
