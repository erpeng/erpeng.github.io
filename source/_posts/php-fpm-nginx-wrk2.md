---
title: Nginx499原因探查2
date: 2019-09-011
tags: 
 - NGINX
 - PHP-FPM
 - WRK
---

## nginx配置

增加如下nginx配置:

```
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent $request_time $upstream_connect_time  $upstream_header_time   $upstream_response_time  "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
```

request_time:nginx从接收到客户端第一个字节到发送到客户端最后一个字节的时间
upstream_connect_time:nginx建立到上游连接的时间
upstream_header_time:nginx从建立到上游连接到收到上游响应头的时间
upstream_response_time:nginx从建立到上游连接到接受完响应的时间

看一下nginx中对这三个值的计算逻辑:

```
函数:ngx_http_upstream_connect
		
	    u->start_time = ngx_current_msec;//记录开始时间

函数:ngx_http_upstream_finalize_request

      u->state->response_time = ngx_current_msec - u->start_time;//计算response time

函数:ngx_http_upstream_send_request 		 
      if (u->state->connect_time == (ngx_msec_t) -1) {
          u->state->connect_time = ngx_current_msec - u->start_time; //计算连接时间
      }
函数: ngx_http_upstream_process_header
      u->state->header_time = ngx_current_msec - u->start_time;

```
流程为从开始建立连接记录一个start_time,然后可以发送请求的时候用当前时间减去start_time计算upstream_connect_time,收到响应头后通过当前时间减去start_time计算upstream_header_time,结束上游请求的时候用当前时间减去start_time计算upstream_response_time(**注意如上响应头有相应的nginx版本要求**)

upstream_header_time-upstream_connect_time可以理解为从发送请求开始到响应的时间
upstream_response_time-upstream_header_time如果很大,可以理解数据量响应耗时比较多

继续实验,将php脚本sleep 1s,然后用wrk压测

```
wrk -c 20  -t 20    --latency http://localhost:8800/index.php
```
php-fpm配置如下:

```
pm.max_children = 8
pm.start_servers = 2
pm.min_spare_servers = 2
pm.max_spare_servers = 4
```

观察nginx日志如下:

```
127.0.0.1 - - [11/Sep/2019:11:53:03 +0800] "GET /index.php HTTP/1.1" 200 14 1.000 0.001  1.001   1.001  "-" "-" "-"
 87 127.0.0.1 - - [11/Sep/2019:11:53:03 +0800] "GET /index.php HTTP/1.1" 200 14 1.001 0.000  1.000   1.000  "-" "-" "-"
 88 127.0.0.1 - - [11/Sep/2019:11:53:03 +0800] "GET /index.php HTTP/1.1" 200 14 1.001 0.000  1.000   1.000  "-" "-" "-"
 89 127.0.0.1 - - [11/Sep/2019:11:53:03 +0800] "GET /index.php HTTP/1.1" 200 14 1.000 0.000  1.001   1.001  "-" "-" "-"
 90 127.0.0.1 - - [11/Sep/2019:11:53:03 +0800] "GET /index.php HTTP/1.1" 200 14 1.316 0.000  1.316   1.316  "-" "-" "-"
 91 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.000 0.000  1.999   1.999  "-" "-" "-"
 92 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.000 0.000  2.000   2.000  "-" "-" "-"
 93 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.000 0.000  2.000   2.000  "-" "-" "-"
 94 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.000 0.000  2.000   2.000  "-" "-" "-"
 95 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.316 0.001  2.316   2.316  "-" "-" "-"
 96 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.317 0.000  2.316   2.316  "-" "-" "-"
 97 127.0.0.1 - - [11/Sep/2019:11:53:04 +0800] "GET /index.php HTTP/1.1" 200 14 2.320 0.000  2.320   2.320  "-" "-" "-"
 98 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 2.999 0.000  2.999   2.999  "-" "-" "-"
 99 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 2.998 0.000  2.999   2.999  "-" "-" "-"
100 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 2.998 0.000  2.998   2.998  "-" "-" "-"
101 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 2.999 0.000  2.999   2.999  "-" "-" "-"
102 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 3.315 0.000  3.315   3.315  "-" "-" "-"
103 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 3.315 0.000  3.315   3.315  "-" "-" "-"
104 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 3.315 0.000  3.316   3.316  "-" "-" "-"
105 127.0.0.1 - - [11/Sep/2019:11:53:05 +0800] "GET /index.php HTTP/1.1" 200 14 3.319 0.000  3.319   3.319  "-" "-" "-"
106 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 3.001 0.000  3.001   3.001  "-" "-" "-"
107 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 3.001 0.001  3.002   3.002  "-" "-" "-"
108 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 3.001 0.001  3.002   3.002  "-" "-" "-"
109 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 3.001 0.000  3.001   3.001  "-" "-" "-"
110 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 3.001 0.000  3.001   3.001  "-" "-" "-"
111 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 2.317 0.001  2.318   2.318  "-" "-" "-"
112 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
113 127.0.0.1 - - [11/Sep/2019:11:53:06 +0800] "GET /index.php HTTP/1.1" 200 14 2.322 0.000  2.321   2.321  "-" "-" "-"
114 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 3.000 0.000  3.001   3.001  "-" "-" "-"
115 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.685   2.685  "-" "-" "-"
116 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.683 0.000  2.684   2.684  "-" "-" "-"
117 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.681 0.000  2.680   2.680  "-" "-" "-"
118 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
119 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
120 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
121 127.0.0.1 - - [11/Sep/2019:11:53:07 +0800] "GET /index.php HTTP/1.1" 200 14 2.321 0.000  2.321   2.321  "-" "-" "-"
122 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.685 0.000  2.684   2.684  "-" "-" "-"
123 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.685 0.000  2.684   2.684  "-" "-" "-"
124 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
125 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.680 0.000  2.681   2.681  "-" "-" "-"
126 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
127 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.317   2.317  "-" "-" "-"
128 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
129 127.0.0.1 - - [11/Sep/2019:11:53:08 +0800] "GET /index.php HTTP/1.1" 200 14 2.321 0.000  2.321   2.321  "-" "-" "-"
130 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
131 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.685   2.685  "-" "-" "-"
132 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
133 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.680 0.001  2.681   2.681  "-" "-" "-"
134 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
135 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
136 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.317 0.000  2.318   2.318  "-" "-" "-"
137 127.0.0.1 - - [11/Sep/2019:11:53:09 +0800] "GET /index.php HTTP/1.1" 200 14 2.321 0.000  2.321   2.321  "-" "-" "-"
138 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
139 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
140 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.684   2.684  "-" "-" "-"
141 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.681 0.000  2.680   2.680  "-" "-" "-"
142 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.319   2.319  "-" "-" "-"
143 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
144 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.318 0.000  2.318   2.318  "-" "-" "-"
145 127.0.0.1 - - [11/Sep/2019:11:53:10 +0800] "GET /index.php HTTP/1.1" 200 14 2.321 0.000  2.321   2.321  "-" "-" "-"
146 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.001  2.684   2.684  "-" "-" "-"
147 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.683   2.683  "-" "-" "-"
148 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.684 0.000  2.683   2.683  "-" "-" "-"
149 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.680 0.000  2.681   2.681  "-" "-" "-"
150 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.319 0.000  2.319   2.319  "-" "-" "-"
151 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.319 0.000  2.319   2.319  "-" "-" "-"
152 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.319 0.000  2.319   2.319  "-" "-" "-"
153 127.0.0.1 - - [11/Sep/2019:11:53:11 +0800] "GET /index.php HTTP/1.1" 200 14 2.321 0.000  2.321   2.321  "-" "-" "-"
154 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 200 14 2.683 0.000  2.684   2.684  "-" "-" "-"
155 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 200 14 2.683 0.000  2.684   2.684  "-" "-" "-"
156 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 200 14 2.685 0.000  2.685   2.685  "-" "-" "-"
157 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 200 14 2.682 0.000  2.681   2.681  "-" "-" "-"
158 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.018 0.000  -   1.019  "-" "-" "-"
159 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.018 0.000  -   1.018  "-" "-" "-"
160 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.018 0.000  -   1.018  "-" "-" "-"
161 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.018 0.000  -   1.018  "-" "-" "-"
162 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.700 0.000  -   0.700  "-" "-" "-"
163 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.697 0.000  -   0.698  "-" "-" "-"
164 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.700 0.000  -   0.700  "-" "-" "-"
165 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.700 0.000  -   0.700  "-" "-" "-"
166 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.018 0.000  -   0.018  "-" "-" "-"
167 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.016 0.000  -   0.017  "-" "-" "-"
168 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.016 0.001  -   0.017  "-" "-" "-"
169 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 0.016 0.000  -   0.016  "-" "-" "-"
170 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 2.020 0.000  -   2.019  "-" "-" "-"
171 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 2.019 0.000  -   2.019  "-" "-" "-"
172 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 2.019 0.000  -   2.019  "-" "-" "-"
173 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 2.019 0.000  -   2.018  "-" "-" "-"
174 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.702 0.000  -   1.701  "-" "-" "-"
175 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.702 0.000  -   1.701  "-" "-" "-"
176 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.702 0.000  -   1.701  "-" "-" "-"
177 127.0.0.1 - - [11/Sep/2019:11:53:12 +0800] "GET /index.php HTTP/1.1" 499 0 1.699 0.000  -   1.698  "-" "-" "-"

```

可以观察出如下规律:
* 11:53:03开始到11:53:06响应时间从1s逐步增加到3s,之后下降稳定到2s多
* upstream_connect_time在us级别,request_time主要消耗在upstream的处理上.但上游每个请求实际耗时1s,说明其他请求在排队等待处理
* 出现一批499,根本没有收到响应

看一下php-fpm的日志:

```
[11-Sep-2019 11:53:05] WARNING: [pool default] server reached pm.max_children setting (8), consider raising it
```
11:53:05的时候打了一条warning,建议提高max_children.后续可以研究php-fpm是升到8个children之后打的日志还是升之前打的日志.根据nginx日志推测,2s之后才开始升到8个children.

## nginx499源码流程

nginx源码版本如下:
```
[root@rh ~ ]# ~/nginx-1.17.3/objs/nginx -v
nginx version: nginx/1.17.3
```
在ngx_http_upstream_init_request函数中,会做如下设置:
```
        r->read_event_handler = ngx_http_upstream_rd_check_broken_connection;
        r->write_event_handler = ngx_http_upstream_wr_check_broken_connection;
```

及将客户端和nginx的读写回调函数分别设置为ngx_http_upstream_rd_check_broken_connection和ngx_http_upstream_wr_check_broken_connection,而这两个函数都是调用ngx_http_upstream_check_broken_connection.顾名思义,该函数检测客户端断开的连接,检测到之后会记录499.

首先看看事件处理模块在ngx_epoll_process_events中会有如下代码:
```
#if (NGX_HAVE_EPOLLRDHUP)
            if (revents & EPOLLRDHUP) {
                rev->pending_eof = 1;
            }

            rev->available = 1;
#endif
```

在使用 2.6.17 之后版本内核的服务器系统中，对端连接断开触发的 epoll 事件会包含 EPOLLIN | EPOLLRDHUP,如果有EPOLLRDHUP事件之后会设置event的pending_eof为1

然后在ngx_http_upstream_check_broken_connection中有如下代码:

```
#if (NGX_HAVE_EPOLLRDHUP)

    if ((ngx_event_flags & NGX_USE_EPOLL_EVENT) && ngx_use_epoll_rdhup) {
        socklen_t  len;

        if (!ev->pending_eof) { //如果有RDHUP事件,则ev->pending_eof已被置为1
            return;
        }

        ev->eof = 1;
        c->error = 1;

        err = 0;
        len = sizeof(ngx_err_t);

        /*
         * BSDs and Linux return 0 and set a pending error in err
         * Solaris returns -1 and sets errno
         */

        if (getsockopt(c->fd, SOL_SOCKET, SO_ERROR, (void *) &err, &len)
            == -1)
        {
            err = ngx_socket_errno;
        }

        if (err) {
            ev->error = 1;
        }

        if (!u->cacheable && u->peer.connection) {
            ngx_log_error(NGX_LOG_INFO, ev->log, err,//打印错误日志
                        "epoll_wait() reported that client prematurely closed "
                        "connection, so upstream connection is closed too");
            ngx_http_upstream_finalize_request(r, u,
                                               NGX_HTTP_CLIENT_CLOSED_REQUEST);//调用该函数结束请求.可以看到上文中upstream_response_time也是在该函数中计算的
            return;
        }
        ...
    }
```
ngx_http_upstream_finalize_request中会做资源回收清理以及关闭上游的fd,然后继续调用        ngx_http_finalize_request(r, rc);

调用链如下:

```
(gdb) bt
#0  ngx_http_post_request (r=r@entry=0x25ffad0, pr=pr@entry=0x2600010) at src/http/ngx_http_request.c:2408
#1  0x000000000043e147 in ngx_http_terminate_request (r=r@entry=0x25ffad0, rc=rc@entry=499) at src/http/ngx_http_request.c:2658
#2  0x000000000043ef96 in ngx_http_finalize_request (r=r@entry=0x25ffad0, rc=rc@entry=499) at src/http/ngx_http_request.c:2454
#3  0x000000000044cd92 in ngx_http_upstream_finalize_request (r=r@entry=0x25ffad0, u=u@entry=0x2600e70, rc=rc@entry=499) at src/http/ngx_http_upstream.c:4450
#4  0x000000000044d0b2 in ngx_http_upstream_check_broken_connection (r=0x25ffad0, ev=0x26479b0) at src/http/ngx_http_upstream.c:1423
#5  0x000000000044d26e in ngx_http_upstream_rd_check_broken_connection (r=<optimized out>) at src/http/ngx_http_upstream.c:1296
#6  0x000000000043e1c5 in ngx_http_request_handler (ev=<optimized out>) at src/http/ngx_http_request.c:2349
#7  0x00000000004330c4 in ngx_epoll_process_events (cycle=<optimized out>, timer=<optimized out>, flags=<optimized out>) at src/event/modules/ngx_epoll_module.c:902
#8  0x000000000042a558 in ngx_process_events_and_timers (cycle=cycle@entry=0x26022d0) at src/event/ngx_event.c:242
#9  0x00000000004314a4 in ngx_worker_process_cycle (cycle=0x26022d0, data=<optimized out>) at src/os/unix/ngx_process_cycle.c:750
#10 0x000000000042fbe2 in ngx_spawn_process (cycle=cycle@entry=0x26022d0, proc=proc@entry=0x431433 <ngx_worker_process_cycle>, data=data@entry=0x0,
    name=name@entry=0x4830d5 "worker process", respawn=respawn@entry=-4) at src/os/unix/ngx_process.c:199
#11 0x000000000043076b in ngx_start_worker_processes (cycle=cycle@entry=0x26022d0, n=1, type=type@entry=-4) at src/os/unix/ngx_process_cycle.c:359
#12 0x00000000004321b8 in ngx_master_process_cycle (cycle=0x26022d0, cycle@entry=0x25fe2c0) at src/os/unix/ngx_process_cycle.c:244
#13 0x000000000040ca1b in main (argc=<optimized out>, argv=<optimized out>) at src/core/nginx.c:382
```

## 小结

本文关注nginx如何检测客户端连接从而生成499的access log以及upstream的一些状态指标的计算逻辑.