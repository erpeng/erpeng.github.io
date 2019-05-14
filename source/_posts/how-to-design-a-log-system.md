---
title: 如何设计一个高并发的日志系统
date: 2019-05-13 
tags: architecture
---

## 日志系统基本概念

### 日志系统必要的因素
* 日志级别:FATAL,WARNING,NOTICE,DEBUG,ALL ...
* 调用栈:文件,函数,行数,日期
  php可以通过debug_backtrace()获取,go通过runtime.Caller()获取
* 日志信息:自定义

### 日志系统性能考量

我们知道日志系统是需要写入磁盘的,在大并发量下,写磁盘是一个昂贵的操作.那么如何避免写入磁盘呢

* 缓冲然后写入
* 通过本机起一个udp服务收集日志.每次写入时通过往127.0.0.1:udpport发送日志

## 各种不同的写入日志方式

* 正常写入
如下为代码示例
```
package logger

import (
	"fmt"
	"log"
	"os"
)

var fileName string
var fileHandle *os.File
var err error

func init() {
	fileName = "/tmp/logger.log"
	fileHandle, err = os.OpenFile(fileName, os.O_RDWR|os.O_CREATE, 0755)
	if err != nil {
		log.Fatal(err)
	}
}

//Logger direct logger
func Logger(log string) {
	fmt.Fprint(fileHandle, log)
}

//Close close logger filehandle
func Close() {
	fileHandle.Close()
}
```

* 缓冲写入
如下为代码示例
```
package logger

import (
	"bufio"
	"fmt"
	"log"
	"os"
)

var maxBufferSize int
var fileNameBuffer string
var fileHandleBuffer *os.File
var writer *bufio.Writer

func init() {
	maxBufferSize = 1 * 1024 * 1024
	fileNameBuffer = "/tmp/loggerbuffer.log"
	var err error
	fileHandleBuffer, err = os.OpenFile(fileNameBuffer, os.O_RDWR|os.O_CREATE, 0755)
	if err != nil {
		log.Fatal(err)
	}
	writer = bufio.NewWriterSize(fileHandleBuffer, maxBufferSize)
}

//BufferLogger buffer logger
func BufferLogger(log string) {
	if writer.Available() < len(log) {
		writer.Flush()
	}
	fmt.Fprint(writer, log)
}

//BufferFlush destruct buffer logger
func BufferFlush() {
	writer.Flush()
}

//BufferClose close buffer logger filehandle
func BufferClose() {
	fileHandleBuffer.Close()
}

```

* 起一个udp服务,然后发送日志到udp服务
如下为代码示例
```
package logger

import (
	"log"
	"net"
)

var conn *net.UDPConn

func init() {
	sip := net.ParseIP("127.0.0.1")
	srcAddr := &net.UDPAddr{IP: net.IPv4zero, Port: 0}
	dstAddr := &net.UDPAddr{IP: sip, Port: 9981}
	var err error
	conn, err = net.DialUDP("udp", srcAddr, dstAddr)
	if err != nil {
		log.Fatal(err)
	}
}

//UDPLogger buffer logger
func UDPLogger(log string) {
	conn.Write([]byte(log))
}

```

udp server的代码如下:
```
package main

import (
	"fmt"
	"log"
	"net"
	"os"
)

func main() {
	var fileNameUDP = "/tmp/loggerUdp.log"
	fileHandleUDP, err := os.OpenFile(fileNameUDP, os.O_RDWR|os.O_CREATE, 0755)
	if err != nil {
		log.Fatal(err)
	}
	listener, err := net.ListenUDP("udp", &net.UDPAddr{IP: net.ParseIP("127.0.0.1"), Port: 9981})
	if err != nil {
		log.Fatal(err)
	}
	data := make([]byte, 1024)
	for {
		n, err := listener.Read(data)
		if err != nil {
			fmt.Printf("error during read: %s", err)
		}
		fmt.Fprint(fileHandleUDP, string(data[:n]))

	}
}
```

三种方法的压测函数如下:

```
package logger

import (
	"testing"
)

func BenchmarkLogger(b *testing.B) {
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		Logger("this is a long long test")
	}
}

func BenchmarkBufferLogger(b *testing.B) {
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		BufferLogger("this is a long long test")

	}
}

func BenchmarkUDPLogger(b *testing.B) {
	b.ResetTimer()

	for i := 0; i < b.N; i++ {
		UDPLogger("this is a long long test")
	}
}

```

压测结果如下:
```
 go test -bench=.
goos: darwin
goarch: amd64
pkg: copywriter.io/logger
BenchmarkLogger-4                 300000              4348 ns/op
BenchmarkBufferLogger-4         20000000                98.7 ns/op
BenchmarkUDPLogger-4              500000              3314 ns/op
PASS
ok      copywriter.io/logger    5.216s
```

再次执行,如下:
```
go test -bench=.
goos: darwin
goarch: amd64
pkg: copywriter.io/logger
BenchmarkLogger-4                 300000              5528 ns/op
BenchmarkBufferLogger-4         10000000               127 ns/op
BenchmarkUDPLogger-4              500000              3216 ns/op
PASS
ok      copywriter.io/logger    4.868s
```

## 结论

缓冲写入>udp写入>直接写入
观察测试结果可以看到,随着写入数据的增加,直接写入会有一个寻址时间导致逐步变慢.而缓冲写入和udp写入不受影响.并且缓冲写入几乎等价于内存操作,但缺点是系统崩溃时可能会丢失部分日志数据