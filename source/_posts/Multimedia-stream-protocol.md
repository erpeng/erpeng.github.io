---
title: 多媒体协议一览
date: 2020-02-07
tags:
 - WebRTC
 - RTMP
 - SIP
 - SDP
 - RTP
 - RTSP
---

> 本文概要介绍多媒体传输协议

## WebRTC
* 支持音视频及数据实时通讯能力
* 支持主流浏览器以及android/ios平台.主流浏览器默认包含,可以从js中直接调用.android/ios平台以library形式提供

### Application flow
webrtc api包括两部分,媒体设备控制（camera/microphone)以及端对端连接

#### 媒体控制
* 查询设备
* 监听设备变化
...

#### 连接建立
* 发送一个SDP(会话描述协议) offer给对端-通过信令通道.SDP描述双方具有的能力
* 信令一般使用sip over websockets,开源实现jsSIP
* ICE(Internet Connectivity Establishment),使用一个STUN或者TURN作为ICE.ICE用来使双方发现.
  STUN(Session Traversal of User Datagram Protocol)
  TURN(Traversal Using Relay NAT)

#### 传输流

```js
const localStream = await getUserMedia({vide: true, audio: true});
const peerConnection = new RTCPeerConnection(iceConfig);
localStream.getTracks().forEach(track => {
    peerConnection.addTrack(track, localStream);
});
```
发送本地的流 

```js
const remoteStream = MediaStream();
const remoteVideo = document.querySelector('#remoteVideo');
remoteVideo.srcObject = remoteStream;

peerConenction.addEventListener('track', async (event) => {
    remoteStream.addTrack(event.track, remoteStream);
});
```
接收对端的流

#### Data channels

发送接收信息

#### TURN server

作为中继,如果两端不在一个局域网,可以传输信息


### webrtc gateway

#### 功能
![webrtc gateway](/img/mp1.png)
通过webrtc呼叫时,webrtc gateway查看是否被叫支持webrtc,如果不支持,需要转换到sip协议.具体包括以下四方面:
* 信令:如果webrtc本身使用sip over websockets那么只需转换为udp/tcp/tls
* media传输:webrtc支持srtp,如果被叫不支持,需要转为stp
* media content:webrtc支持audio codec为G.711或者opus,如果被叫不支持,需要做转换
* media 地址协商:webrtc使用ICE做地址协商,如果不支持,webrtc gateway需要作为ICE服务器中转

见下图:
![webrtc gateway1](/img/mp2.png)

#### 实现

webrtc gateway经常集成到sbc中.

开源实现:
OverSIP
Kamailio
Asterisk
reSIProcate and repro
WebRTC2SIP
Janus
FreeSWITCH
SylkServer
mediasoup

## RTMP&RTP

Real-Time messaging protocol,flash player和server之间传递音视频流的协议.基于TCP.随着flash退出历史舞台会被逐步淘汰,单现在直播场景因为有CDN良好的支持,仍然大量使用RTMP

RTP 基于UDP,和RTCP一起使用.可以传输不同格式的音视频数据,并且在UDP上层做了序列保证等功能.


## SIP 

* SIP Session Initiation Protocol用来初始化、维持、终止一个实时的音视频会话.与SDP,RTP写作完成多媒体信息任务

* 连接建立时,sip message的body中包括一个SDP data unit,制定media format,codec,media communication protocol.媒体流的传递一般用RTP或者SRTP.

* SIP是一个文本协议,request/response模式.复用了许多header以及状态码

* 可以使用TCP或者UDP,端口为5060/5061,后者加密流量,使用TLS




## 参考链接
* https://webrtc.org/
* https://en.wikipedia.org/wiki/WebRTC
* https://en.wikipedia.org/wiki/WebRTC_Gateway
* https://en.wikipedia.org/wiki/Audio_codec
* https://en.wikipedia.org/wiki/Pulse-code_modulation
* https://en.wikipedia.org/wiki/Audio_coding_format
* https://en.wikipedia.org/wiki/FreeSWITCH
* https://en.wikipedia.org/wiki/Real-time_Transport_Protocol