---
title: Nginx499原因探查
date: 2019-09-07
tags: 
 - NGINX
 - PHP-FPM
 - WRK
---
>499为客户端请求超时主动断开后,NGINX日志中记录的状态码.笔者线上遇到一个问题,NGINX日志记录499,request_time记录为1s左右(即客户端的超时时间),php-fpm层记录的实际处理时间为10-20ms,产生了不一致,通过文章中的实验验证一下问题出自哪里.

## WRK

### 安装
环境如下:
```
[root@rh etc]# cat /etc/redhat-release
CentOS Linux release 7.5.1804 (Core)
```
* 安装wrkYUM源:yum install https://extras.getpagespeed.com/release-el7-latest.rpm
* 安装wrk :yum install wrk

### 使用

```
[root@rh etc]# wrk
Usage: wrk <options> <url>
  Options:
    -c, --connections <N>  Connections to keep open //连接数量
    -d, --duration    <T>  Duration of test         //测试持续时间
    -t, --threads     <N>  Number of threads to use //并发线程数量

    -s, --script      <S>  Load Lua script file        
    -H, --header      <H>  Add header to request
        --latency          Print latency statistics   
        --timeout     <T>  Socket/request timeout  //请求超时时间,可以模拟客户端请求超时断开,从而产生499    
    -v, --version          Print version details
```


## php-fpm的配置

```
pm = dynamic
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 4
```
模拟配置,生产上比该值要大.
首先验证一个问题,我们在php代码逻辑中sleep 50s,并发10个请求,根据如上配置,php-fpm是有4个worker还是10个worker

```
[root@rh matrix]# wrk -c 10 -d 30 -t 10 http://localhost/alloc
...
[root@rh etc]# ps -ef |grep php-fpm|wc -l
10
```

可以看到是10个worker,请求结束后会变为4个.

看起来似乎跟php-fpm的配置没有关系.php-fpm是能够动态增加个数的.

## 模拟线上 

线上情况是客户端1s的超时时间,那么我们可以如下设计实验,在php接口中sleep 900ms,然后设置请求持续时间为1s,看看实际情况:

```
[root@rh fpm.d]# wrk -c 20  -t 20 -d 1   --latency http://localhost/alloc
Running 1s test @ http://localhost/alloc
  20 threads and 20 connections
  Thread Stats   Avg      Stdev     Max   +/- Stdev
    Latency   908.22ms    1.84ms 910.55ms   75.00%
    Req/Sec     0.50      0.58     1.00    100.00%
  Latency Distribution
     50%  908.84ms
     75%  910.55ms
     90%  910.55ms
     99%  910.55ms
  4 requests in 1.10s, 856.00B read
Requests/sec:      3.64
Transfer/sec:     778.77B
```

可以看到只执行成功4个请求,并不是如预料中所想的10个请求,看看nginx和应用的日志

```
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 200 5 0.903 0.903 22 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 200 5 0.907 0.907 22 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 200 5 0.906 0.906 22 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 200 5 0.908 0.908 22 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 0.194 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 0.196 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 0.194 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 0.190 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.100 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.100 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.100 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.097 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.097 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.097 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.097 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.096 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.096 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.096 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.096 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.096 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.094 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.094 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.094 - 0 "-" "-" 127.0.0.1 "-" "-"
- - localhost [08/Sep/2019:08:37:33 +0800] "GET /alloc HTTP/1.1" 49 499 0 1.094 - 0 "-" "-" 127.0.0.1 "-" "-"
```
大量的499 1s钟超时

应用层日志处理时间基本维持在900ms左右.并且应用层与nginx层日志数量都是24条

当然,调大php-fpm启动数量后可以请求的个数相应增加.读者可自行实验.

## 小结


* 动态配置看起来很美好,实际总是有时间开销(master统计,fork进程,监控并发).大并发场景建议还是使用static静态配好.

* 即使php-fpm处理不了所有请求,nginx还是会发送所有建立的连接并发送到php-fpm,占用资源

