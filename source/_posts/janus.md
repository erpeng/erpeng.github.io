---
title: Janus笔记-1
date: 2020-09-28
tags:
 - Janus
---

## RTP

### 场景
rfc3550:Real-time transport Protocol,适用场景如下:

* 多播音频会议:
    * RTP header需要有编码格式,时间戳以及序列号,用来做乱序重排以及丢包重传
    * RTCP通过传输reception report 来报告新用户的进入退出以及当前接收情况,用来协商自适应码率
* 音视频会议:
    * 通过不同的RTP session来传输音视频
* Mixers和Translators:
    * Mixer可以用来放置到一个低带宽的用户前边,用来合成码流
    * Translator放到一个有防火墙的内外网之间,用来传递数据
* Layered Encodings

### 格式
```
  0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |V=2|P|X|  CC   |M|     PT      |       sequence number         |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |                           timestamp                           |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |           synchronization source (SSRC) identifier            |
   +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
   |            contributing source (CSRC) identifiers             |
   |                             ....                              |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

* version（2bit):固定为2
* P(1bit):是否有pading
* X(1bit):是否有扩展头
* CC(4):CSRC个数
* M:Mark位
* PT:payload type
* 序列号、时间戳、SSRC、CSRC(最多16个)
* 扩展头

## RTCP

### 功能

* 数据分发的质量监控
* RTCP包括一个每个参与者都唯一的标识
* 知晓参与的各方

### 格式

SR:Sender Report RTCP Packet

```
        0                   1                   2                   3
        0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
header |V=2|P|    RC   |   PT=SR=200   |             length            |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         SSRC of sender                        |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
sender |              NTP timestamp, most significant word             |
info   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |             NTP timestamp, least significant word             |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         RTP timestamp                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                     sender's packet count                     |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      sender's octet count                     |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_1 (SSRC of first source)                 |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  1    | fraction lost |       cumulative number of packets lost       |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |           extended highest sequence number received           |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                      interarrival jitter                      |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                         last SR (LSR)                         |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
       |                   delay since last SR (DLSR)                  |
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
report |                 SSRC_2 (SSRC of second source)                |
block  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
  2    :                               ...                             :
       +=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+=+
       |                  profile-specific extensions                  |
       +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```


## SDP 

```

v=0
o=- 7017624586836067756 2 IN IP4 127.0.0.1
s=-
t=0 0

//下面 m= 开头的两行，是两个媒体流：一个音频，一个视频。
m=audio 9 UDP/TLS/RTP/SAVPF 111 103 104 9 0 8 106 105 13 126
...
m=video 9 UDP/TLS/RTP/SAVPF 96 97 98 99 100 101 102 122 127 121 125 107 108 109 124 120 123 119 114 115 116
...
```
```
* "v="开始到"m="是会话级描述
* "m="开始到下一个"m="是媒体级描述
* o=<username> <session id> <version> <network type> <address type> <address> 
* s=<session name>
* t=<start time> <stop time> 会话的开始和结束时间

* m=<media> <port> <transport> <fmt list> fmt list表示payload type
* a=<TYPE>或 a=<TYPE>:<VALUES> 如 a=rtpmap:<payload type> <encoding name>/<clock rate>[/<encodingparameters>]
                                 a=fmtp:<payload type> <format specific parameters>
```

## STUN

### Candidate收集
* host 类型,即本机内网的 IP 和端口
* srflx 类型,即本机 NAT 映射后的外网的 IP 和端口
* relay 类型,即中继服务器的 IP 和端口

### srflx获取
通过STUN协议获取本机在外网映射的端口和地址
STUN:session traversal utilities for nat RFC5389

* 首先在外网搭建一个 STUN 服务器，现在比较流行的 STUN 服务器是 CoTURN，你可以到 GitHub 上自己下载源码编译安装
* 当 STUN 服务器安装好后，从内网主机发送一个 binding request 的 STUN 消息到 STUN 服务器
* STUN 服务器收到该请求后，会将请求的 IP 地址和端口填充到 binding response 消息中，然后顺原路将该消息返回给内网主机。此时，收到 binding response 消息的内网主机就可以解析 binding response 消息了，并可以从中得到自己的外网 IP 和端口


## TURN

RFC5766 Traversal Using Relays around NAT (TURN):

## ICE
rfc5245 Interactive Connectivity Establishment

## NAT

NAT种类:
* 完全锥形 
{
  内网IP，
  内网端口，
  映射的外网IP，
  映射的外网端口
}
* IP限制锥形
{
  内网IP，
  内网端口，
  映射的外网IP，
  映射的外网端口，
  被访问主机的IP
}
* 端口限制锥形
{
  内网IP，
  内网端口，
  映射的外网IP，
  映射的外网端口，
  被访问主机的IP,
  被访问主机的端口
}
* 对称型NAT
同端口限制锥形的要求,并且对称型NAT访问不同的服务时,NAT上边会开不同的端口进行访问.例如访问A服务时NAT开portA,访问B服务时NAT开portB

 
## 参考链接:
* https://time.geekbang.org/column/article/112325