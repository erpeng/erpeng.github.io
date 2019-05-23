---
title: go net包源码分析
date: 2019-05-22
tags: go
---
>net包中按实体分层分为mac,ip,tcp,udp,unix.按逻辑有Listener,Conn,Resolver
## 实体分层

### mac

```
struct:
type HardwareAddr []byte
func:
将一个string类型的MAC地址解析为二进制格式,格式无效会返回错误:
func ParseMAC(s string) (hw HardwareAddr, err error)
将二进制解析为MAC地址,以':'分隔
func (a HardwareAddr) String() string
```

###  ip
```
struct:4字节的IPV4地址,16字节的 IPV6地址
type IP []byte
```

比较简单,不赘述

## 逻辑分层

### Addr
```
接口:实现Network与String两个函数
type Addr interface {
	Network() string 
	String() string  
}
```
#### IPAddr
```
type IPAddr struct {
	IP   IP
	Zone string 
}
该函数返回IPAddr类型
func ResolveIPAddr(network, address string) (*IPAddr, error) 
```
#### TCPAddr
```
type TCPAddr struct {
	IP   IP
	Port int
	Zone string // IPv6 scoped addressing zone
}
```
同理,通过该函数返回TCPAddr类型
func ResolveTCPAddr(network, address string) (*TCPAddr, error) 



### Conn
```
type Conn interface {
	Read(b []byte) (n int, err error)
	Write(b []byte) (n int, err error)
	Close() error
	LocalAddr() Addr
	RemoteAddr() Addr
	SetDeadline(t time.Time) error
	SetReadDeadline(t time.Time) error
	SetWriteDeadline(t time.Time) error
}
```
func Dial(network, address string) (Conn, error)
func DialTimeout(network, address string, timeout time.Duration) (Conn, error)
后者是前者的一个特例,加了超时时间,超时控制也是使用context包实现
需要注意的一点是阻塞相关的系统调用在底层也是起单独的goroutine实现

#### IPConn
```
type IPConn struct {
	conn
}

type conn struct {
	fd *netFD
}
```

#### TCPConn
```
type TCPConn struct {
	conn
}
```
如下函数返回一个 TCPConn
func DialTCP(network string, laddr, raddr *TCPAddr) (*TCPConn, error) 

### Listener

```
type Listener interface {
	Accept() (Conn, error)
	Close() error
	Addr() Addr
}

type TCPListener struct {
	fd *netFD
}
```
通过如下函数返回一个TCPListener
func ListenTCP(network string, laddr *TCPAddr) (*TCPListener, error) 
通过如下函数返回一个TCPConn
func (l *TCPListener) AcceptTCP() (*TCPConn, error) 


### UDP,Unix
UDP,Unix同理.UDPConn,UDPAddr,由于UDP不需要Accept这一步骤故没有UDPListener
UnixConn,UnixAddr,UnixListener同上
