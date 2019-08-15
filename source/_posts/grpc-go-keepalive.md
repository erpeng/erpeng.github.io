---
title: grpc-go源码解析6-keepalive
date: 2019-08-15
tags: 
- grpc-go
- HTTP2
---

>分析客户端keepalive实现

## 概览

grpc客户端客户可以配置keepalive,具体配置如下
```
var kacp = keepalive.ClientParameters{
	Time:                10 * time.Second, // send pings every 10 seconds if there is no activity
	Timeout:             time.Second,      // wait 1 second for ping ack before considering the connection dead
	PermitWithoutStream: true,             // send pings even without active streams
}

conn, err := grpc.Dial(*addr, grpc.WithInsecure(), grpc.WithKeepaliveParams(kacp))
```

keepalive.ClientParameters有三个参数,分别为Time,Timeout和PermitWithoutStream,上述代码配置解释如下:
* Time:如果没有activity,则每隔10s发送一个ping包
* Timeout:如果ping ack 1s之内未返回则认为连接已断开
* PermitWithoutStream: 如果没有active的stream,是否允许发送ping

我们设想一下代码如何实现:

* 首先得有一个独立的goroutine做keepalive的实现
* 其次得有一个定时器,10s触发一次,但触发之后如何判断ping超时时间呢
* 如何判断是否有activity以及是否有active stream呢?

## 代码实现

```
func newHTTP2Client(connectCtx, ctx context.Context, addr TargetInfo, opts ConnectOptions, onPrefaceReceipt func(), onGoAway func(GoAwayReason), onClose func()) (_ *http2Client, err error) {
	...
	if t.keepaliveEnabled {
		go t.keepalive()
	}
	...
}
```
在新建一个HTTP2Client的时候会起一个goroutine处理keepalive

```
func (t *http2Client) keepalive() {
	p := &ping{data: [8]byte{}} //ping的内容
	timer := time.NewTimer(t.kp.Time) //起一个定时器,触发时间为配置的Time值
	//死循环
	for {
		select {
		//定时器触发
		case <-timer.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				timer.Reset(t.kp.Time)
				continue
			}
			// Check if keepalive should go dormant.
			t.mu.Lock()
			if len(t.activeStreams) < 1 && !t.kp.PermitWithoutStream {
				// Make awakenKeepalive writable.
				<-t.awakenKeepalive
				t.mu.Unlock()
				select {
				case <-t.awakenKeepalive:
					// If the control gets here a ping has been sent
					// need to reset the timer with keepalive.Timeout.
				case <-t.ctx.Done():
					return
				}
			} else {
				t.mu.Unlock()
				if channelz.IsOn() {
					atomic.AddInt64(&t.czData.kpCount, 1)
				}
				// Send ping.
				t.controlBuf.put(p)
			}

			// By the time control gets here a ping has been sent one way or the other.
			timer.Reset(t.kp.Timeout)
			select {
			case <-timer.C:
				if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
					timer.Reset(t.kp.Time)
					continue
				}
				t.Close()
				return
			case <-t.ctx.Done():
				if !timer.Stop() {
					<-timer.C
				}
				return
			}
		//context结束
		case <-t.ctx.Done():
			if !timer.Stop() {
				<-timer.C
			}
			return
		}
	}
}
```
步骤如下:
* 填充ping包内容,为8个字节的空字节
* 起定时器,触发时间为Time字段
* 进入死循环
	* 查看是否触发定时器
	* 查看是否context结束
	
首先看context结束的处理流程.执行timer.Stop(),如果返回true,说明定时器已经销毁,否则说明定时器正在销毁或者已经触发,此时从timer.C管道中读取内容然后返回

接着看主流程及定时器的触发,触发之后执行流程如下:

* 原子CAS操作,查看activity的值是否为1,如果为1说明客户端和服务端存在activity,则将activity置为0并且重置定时器
* 接着判断客户端是否和服务端有active stream,如果没有并且PermitWithoutStream设置为false,则阻塞等待
* 否则将ping包放入control buffer(即异步发送,有其他goroutine会将control buffer中的包发送)
* 重置定时器的触发时间为Timeout时间,并且等待触发.Timeout时间之后,如果activity的值为1,说明已经收到了ping包的回复,则重置定时器时间为Time并进入下一循环,否则结束客户端

从中可以看出,如果client收到server的stream,会将activity置为1.

## 小结

go的time和goroutine组合实现一个keepalive还是很简单的.下一节继续查看server端的keepalive实现.HTTP2是一个全双工流式协议,服务端也可以主动ping客户端,并且服务端还会有一些检测连接可用性和控制客户端ping包频次的配置.
