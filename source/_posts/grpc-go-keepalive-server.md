---
title: grpc-go源码解析7-keepalive
date: 2019-08-15
tags: 
- grpc-go
- HTTP2
---

>分析服务端keepalive实现

## 概览
如下是服务端keepalive配置
```
var kaep = keepalive.EnforcementPolicy{
	MinTime:             5 * time.Second, // If a client pings more than once every 5 seconds, terminate the connection
	PermitWithoutStream: true,            // Allow pings even when there are no active streams
}

var kasp = keepalive.ServerParameters{
	MaxConnectionIdle:     15 * time.Second, // If a client is idle for 15 seconds, send a GOAWAY
	MaxConnectionAge:      30 * time.Second, // If any connection is alive for more than 30 seconds, send a GOAWAY
	MaxConnectionAgeGrace: 5 * time.Second,  // Allow 5 seconds for pending RPCs to complete before forcibly closing connections
	Time:                  5 * time.Second,  // Ping the client if it is idle for 5 seconds to ensure the connection is still active
	Timeout:               1 * time.Second,  // Wait 1 second for the ping ack before assuming the connection is dead
}

func main(){
	...
	s := grpc.NewServer(grpc.KeepaliveEnforcementPolicy(kaep), grpc.KeepaliveParams(kasp))
	...
}

```
服务端分为两部分配置,一部分为keepalive.EnforcementPolicy,有两个配置参数:

* MinTime:如果客户端两次ping的间隔小于5s,中止连接
* PermitWithoutStream:即使没有active stream,也允许ping

另一部分为keepalive.ServerParameters,五个配置参数:
* MaxConnectionIdle:如果一个client空闲超过15s,发送一个GOAWAY,为了防止同一时间发送大量GOAWAY,会在15s时间间隔上下浮动15*10%,即15+1.5或者15-1.5
* MaxConnectionAge:如果任意连接存活时间超过30s,发送一个GOAWAY
* MaxConnectionAgeGrace:在强制关闭连接之间,允许有5s的时间完成pending的rpc请求
* Time:如果一个clinet空闲超过5s,则发送一个ping请求
* Timeout:如果ping请求1s内未收到回复,则认为该连接已断开


## keepalive策略相关代码

```
func (t *http2Server) handlePing(f *http2.PingFrame) {
	...
	if ns < 1 && !t.kep.PermitWithoutStream {
		// Keepalive shouldn't be active thus, this new ping should
		// have come after at least defaultPingTimeout.
		if t.lastPingAt.Add(defaultPingTimeout).After(now) {
			t.pingStrikes++
		}
	} else {
		// Check if keepalive policy is respected.
		if t.lastPingAt.Add(t.kep.MinTime).After(now) {
			t.pingStrikes++
		}
	}
	if t.pingStrikes > maxPingStrikes {
		// Send goaway and close the connection.
		errorf("transport: Got too many pings from the client, closing the connection.")
		t.controlBuf.put(&goAway{code: http2.ErrCodeEnhanceYourCalm, debugData: []byte("too_many_pings"), closeConn: true})
	}
}
```
判断是否违反两条策略,如果违反则将pingStrikes++,当违反次数大于maxPingStrikes(2)时,打印一条错误日志并且发送一个goAway包.


## keepalive参数相关代码

```
func newHTTP2Server(conn net.Conn, config *ServerConfig) (_ ServerTransport, err error) {
	...
	go t.keepalive()
	...
}
```
新建一个HTTP2 server的时候会启动一个单独的goroutine处理keepalive.


```
func (t *http2Server) keepalive() {
	p := &ping{}
	var pingSent bool
	maxIdle := time.NewTimer(t.kp.MaxConnectionIdle)
	maxAge := time.NewTimer(t.kp.MaxConnectionAge)
	keepalive := time.NewTimer(t.kp.Time)
	// NOTE: All exit paths of this function should reset their
	// respective timers. A failure to do so will cause the
	// following clean-up to deadlock and eventually leak.
	defer func() {
		if !maxIdle.Stop() {
			<-maxIdle.C
		}
		if !maxAge.Stop() {
			<-maxAge.C
		}
		if !keepalive.Stop() {
			<-keepalive.C
		}
	}()
	for {
		select {
		case <-maxIdle.C:
			t.mu.Lock()
			idle := t.idle
			if idle.IsZero() { // The connection is non-idle.
				t.mu.Unlock()
				maxIdle.Reset(t.kp.MaxConnectionIdle)
				continue
			}
			val := t.kp.MaxConnectionIdle - time.Since(idle)
			t.mu.Unlock()
			if val <= 0 {
				// The connection has been idle for a duration of keepalive.MaxConnectionIdle or more.
				// Gracefully close the connection.
				t.drain(http2.ErrCodeNo, []byte{})
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxIdle.Reset(infinity)
				return
			}
			maxIdle.Reset(val)
		case <-maxAge.C:
			t.drain(http2.ErrCodeNo, []byte{})
			maxAge.Reset(t.kp.MaxConnectionAgeGrace)
			select {
			case <-maxAge.C:
				// Close the connection after grace period.
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				maxAge.Reset(infinity)
			case <-t.ctx.Done():
			}
			return
		case <-keepalive.C:
			if atomic.CompareAndSwapUint32(&t.activity, 1, 0) {
				pingSent = false
				keepalive.Reset(t.kp.Time)
				continue
			}
			if pingSent {
				t.Close()
				// Resetting the timer so that the clean-up doesn't deadlock.
				keepalive.Reset(infinity)
				return
			}
			pingSent = true
			if channelz.IsOn() {
				atomic.AddInt64(&t.czData.kpCount, 1)
			}
			t.controlBuf.put(p)
			keepalive.Reset(t.kp.Timeout)
		case <-t.ctx.Done():
			return
		}
	}
}
```
启动三个定时器,分别处理maxIdle,maxAge,keepAlive相关事件:
* maxIdle:判断client空闲时间是否超出配置时间,如果超时,则调用t.drain,该方法会发送一个GOAWAY包
* maxAge:触发之后首先调用t.drain发送GOAWAY包,接着重置定时器,时间设置为MaxConnectionAgeGrace,再次触发后调用t.Close()直接关闭
* keepalive:首先判断activity是否为1,如果不是则置pingSent为true,并且发送ping包,接着重置定时器时间为Timeout,再次触发后如果activity不为1(即未收到ping的回复)并且pingSent为true,则调用t.Close()关闭连接

## 小结

keepalive配置及实现通过client和server两端来实现.借助于go的timer和goroutine可以实现的相当简洁易懂

