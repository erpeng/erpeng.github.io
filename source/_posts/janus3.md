---
title: Janus笔记-3
date: 2020-10-19
tags:
 - Janus
---

## 环境配置
启动janus demo:
```
cd $janus_path/html/
php -S 10.26.29.25:8888 -t .
```

由于webrtc的使用需要https,因此需要生成自签名证书:
```
openssl genrsa -out www.example.com.key 1024
openssl req -new -key www.example.com.key -out www.example.com.csr
openssl x509 -req -in www.example.com.csr -out www.example.com.crt -signkey www.example.com.key -days 3650
```

配置nginx:
```
server {
    listen              8443 ssl;
    ...
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```
配置janus:
/opt/janus/etc/janus/janus.transport.websockets.jcfg
```
general: {
        json = "indented"                               # Whether the JSON messages should be indented (default),
        ws = true                                               # Whether to enable the WebSockets API
        ws_port = 8188                                  # WebSockets server port
        wss = true                                              # Whether to enable secure WebSockets
        wss_port = 8989                         # WebSockets server secure port, if enabled
}
certificates: {
        cert_pem = "/usr/local/matrix/etc/www.example.com.crt"
        cert_key = "/usr/local/matrix/etc/www.example.com.key"
}
```
修改demo中的transport地址:
echotest.js
```
var server = null;
if(window.location.protocol === 'http:')
        server = "ws://10.26.29.25:8188/janus";
else
        server = "wss://10.26.29.25:8989/janus";
```
启动janus:
```
./bin/janus
```

## 调用链

启动echo test demo,可以看到服务端会启动一个udp端口的监听:
![janus1](/img/janus-1.png)
抓包能够看到,udp 41993开始传输数据:
![janus1](/img/janus-2.png)


启动janus server并且测试janus_echotest plugin,查看rtp的处理流程
```
gdb ./bin/janus
(gdb) b janus_echotest_incoming_rtp
(gdb) b main
(gdb) r
(gdb) c

```
在janus demo中操作echo test,会进入janus_echotest_create_session,通过bt查看调用链如下:
```
(gdb) bt
#0  0x00007fffd8569f00 in janus_echotest_incoming_rtp (handle=0x7fffc40016e0, packet=0x7fffce7eb7d0)
    at plugins/janus_echotest.c:547
#1  0x0000000000441c7e in janus_ice_cb_nice_recv (agent=agent@entry=0x7fffbc0100a0, stream_id=stream_id@entry=1, component_id=component_id@entry=1, len=len@entry=1088, buf=buf@entry=0x7fffce7ebad0 "\220`", ice=ice@entry=0x7fffbc016fc0) at ice.c:2588
#2  0x00007ffff7998554 in nice_component_emit_io_callback (agent=0x7fffbc0100a0, component=0x7fffbc013000, buf=0x7fffce7ebad0 "\220`", buf_len=1088) at ../agent/component.c:957
#3  0x00007ffff799401f in component_io_cb (gsocket=<optimized out>, condition=<optimized out>, user_data=0x7fffbc00b3a0)
    at ../agent/agent.c:5664
#4  0x00007ffff765e286 in socket_source_dispatch () at /lib64/libgio-2.0.so.0
#5  0x00007ffff70c5099 in g_main_context_dispatch () at /lib64/libglib-2.0.so.0
#6  0x00007ffff70c53f8 in g_main_context_iterate.isra.19 () at /lib64/libglib-2.0.so.0
#7  0x00007ffff70c56ca in g_main_loop_run () at /lib64/libglib-2.0.so.0
#8  0x000000000043a1e0 in janus_ice_handle_thread (data=0x7fffc4001730) at ice.c:1165
#9  0x00007ffff70ec540 in g_thread_proxy () at /lib64/libglib-2.0.so.0
#10 0x00007ffff59e6e25 in start_thread () at /lib64/libpthread.so.0
#11 0x00007ffff5710bad in clone () at /lib64/libc.so.6

```
单步调试进入gateway的relay_rtp:
```
(gdb) p *gateway
$6 = {push_event = 0x459940 <janus_plugin_push_event>, relay_rtp = 0x449ee0 <janus_plugin_relay_rtp>, relay_rtcp =
    0x449fa0 <janus_plugin_relay_rtcp>, relay_data = 0x4494e0 <janus_plugin_relay_data>,
  send_pli = 0x44a060 <janus_plugin_send_pli>, send_remb = 0x44a100 <janus_plugin_send_remb>,
  close_pc = 0x44a1a0 <janus_plugin_close_pc>, end_session = 0x44a2a0 <janus_plugin_end_session>,
  events_is_enabled = 0x42af20 <janus_events_is_enabled>, notify_event = 0x44a650 <janus_plugin_notify_event>,
  auth_is_signature_valid = 0x448db0 <janus_plugin_auth_is_signature_valid>, auth_signature_contains =
    0x448df0 <janus_plugin_auth_signature_contains>}
```

最终调用到janus_ice_relay_rtp,该函数处理完毕会调用janus_ice_queue_packet放入异步队列,最终分发报文通过janus_ice_outgoing_traffic_dispatch该函数调用janus_ice_outgoing_traffic_handle分发
