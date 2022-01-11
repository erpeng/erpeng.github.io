---
title: 如何写一篇技术文章
date: 2022-01-10
tags: 
 - GO
 - Writing
---
>提炼一种能够写出通熟易懂技术文章的方法,以Go语言的网络轮询器netpoller的介绍为例

## Morsing's Blog

### The Go Netpoller
Morsing' Blog中关于netpoller的文章在码农桃花源讲调度器以及Draven的<<Go语言设计与实现>>中都有引用,我们先看看博客是如何写的.
文章共三段,都是文字.第一部分引言忽略,第二部分为Blocking

#### Blocking
第二部分第一段如下:

```
Blocking
In Go, all I/O is blocking.
```

抛出结论,Go语言中所有的I/O都是阻塞的,并且说明Go通过Goroutine和Channel处理并发.然后举了net/http的例子,说明这样设计的好处:

```
This construct means that the request handler can be written in a very straightforward manner. First do this, then do that.
```

代码可以以一种非常直接或者直观的方式书写.首先做这个,然后做这个.

第二部分第一段最后一句继续引出下文-"但是我们不能直接拿操作系统的阻塞I/O来实现Go想要的阻塞I/O",为什么呢?引出第二部分第二段,关键结论是:

```
To handle a blocking syscall, we need a thread that can be blocked inside the operating system.
```

每一个阻塞的操作系统I/O都需要一个线程去处理,因此不可行.接着引出第二部分第三段:需要使用操作系统的异步I/O去处理,但是通过在goroutines处阻塞来实现Go的阻塞式IO.

第二部分总结如下:抛出设计结论-举例说明这样设计的好处-如何实现这样的设计,接着引出第三部分,netpoller

#### The netpoller

第一段:

```
The part that converts asynchronous I/O into blocking I/O is called the netpoller
```

将异步I/O转换为阻塞I/O的部分称为netpoller.第一段剩下部分举例说明netpoller在Linux下使用epoll,BSD使用kqueue....

第二段讲述每一个文件句柄都会设置为非阻塞,如果一个文件句柄没有ready,就会直接返回错误码.此时通过调度器和netpoller去把goroutine调度出去,直到有数据时再调度回来.

第三段讲了如果netpoller收到了os可以在某个文件句柄执行I/O的信号,netpoller会将阻塞到这个句柄上的goroutine调度回来重试I/O操作.

第四段讲了这种方法屏蔽了函数指针和一些状态信息(这些信息都通过goroutine去表达),比直接用epoll要方便不少.


#### 总结
文章写的高屋建瓴,第二段首先说明Go的I/O设计,然后举例说明,引出如何实现.第三段说明具体实现.整体结构是结论-举例-总结.

### The Go scheduler
具体内容不再详细叙述,整体逻辑为 
* 引言
* 为什么go需要一个应用层的调度器
* 调度器模型（M:N的模型,分为M、P、G三个角色）并介绍模型标准状态
* 为什么需要P?当执行syscall时的非标准态
* 偷取任务这种非标准态
* 其他待讨论细节,例如cgo threads,netpoller

### 总结
两篇文章都是偏设计与实现,首先是讲解设计以及为什么这样设计,然后用汇总性的语言讲解实现,没有源码分析


## Go语言设计与实现

并发编程这一部分分为7个章节,其中一个章节为网络轮询器.7个章节中每个章节的前两部分都是如下:

* 设计原理
* 数据结构

整体逻辑比较清晰,先讲解调度器、Channel或者网络轮询器的设计原理,并补充一下背景知识,然后讲解实现.根据

```
程序 = 数据结构 + 算法
```
讲解数据结构之后,每个章节继续讨论其具体实现.我们以网络轮询器章节为例继续说明

### 设计原理
讲解I/O模型,包括阻塞I/O,非阻塞I/O,I/O多路复用这些,然后引出netpoll的接口定义:

```
func netpollinit()
func netpollopen(fd uintptr, pd *pollDesc) int32
func netpoll(delta int64) gList
func netpollBreak()
func netpollIsPollDescriptor(fd uintptr) bool
```
接着说明了一下多模块实现,即Epoll、Kqueue等等
但这块关于接口为什么设计成这样其实缺少一些背景,最好是能补充一下Morsing's Blog中关于为什么需要netpoller,以及netpoller的功能.

### 数据结构
讲解了poolDesc数据结构,该结构比较关键.接着讲解了poolCache,用来分配和回收poolDesc,但其实poolCache跟下文关系不大,似乎可以省略.

### 多路复用
分别讲解了网络轮询器的初始化、如何向网络轮询器加入待监控的任务、如何从网络轮询器获取触发的事件

### 小结&&延伸阅读
不再详细叙述

### 总结
Go语言设计与实现是本非常好的书,单就这个章节来说,netpoller与计时器以及调度器的关系只是引导了一下,没有一些叙述或者与前边章节的关联.


## Redis设计与实现
第12章事件关于网络I/O,分为文件事件、时间事件、事件的调度与执行、重点回顾以及参考资料五个章节.

### 文件事件
文件事件章节分为:
* 文件事件处理器的构成
* I/O多路复用程序的实现
* 事件的类型
* API
* 文件事件的处理器

### 时间事件
时间事件章节分为:
* 实现
* API
* 时间事件应用实例:serverCron函数

时间事件首先是一段概括性叙述,包括两部分,Redis时间事件的分类:定时与周期性,时间事件的三个属性:id,when,timeProc,讲解并没有写出具体的数据结构,只是文字介绍.
时间事件的第一部分讲解实现,一个无序列表,画图举例说明.第二部分API通过图片举例和伪代码对API进行了介绍.第三部分也是文字介绍

### 事件的调度与执行:
首先通过伪代码介绍aeProcessEvents函数,负责统一调度执行文件事件和时间事件.接着用伪代码和图片介绍了Redis的main函数流程,最后举例子说明

重点回顾和参考资料不再详细叙述

### 总结
Redis设计与实现很少出现代码,非得出现,就用伪代码,降低阅读难度.整体感特别好,图片说明以及示例多.Go语言设计与实现中开始讲解具体实现时其实也可以参考这种做法.

## 最佳模版
以netpoll为例:

### 示例引出问题
学习的最佳模式其实是带着问题来,所以开头最好通过示例或者其他方式引出疑问,例如如下一个常见的写法:
```
package main

func main() {
	l, err := net.Listen("tcp", ":8080")-----1
    ...
	for {
		conn, err := l.Accept()----2
        ...
		go serve(conn)
	}
}

func serve(conn net.Conn) {
	for {
		data := make([]byte, 1024)
		conn.SetDeadline(time.Now().Add(5 * time.Second))----3
		n, err := conn.Read(data)----4
        ...
	}
}
```
上文标注1,2,3,4处都与netpoller相关,监听一个套接字之后netpoller做了什么,Accept一个连接以及在连接上设置Deadline,在连接上读取netpoller又分别做了什么

### 设计原理

#### Go I/O的设计哲学
Morsing's Blog中的内容,Go I/O都是阻塞的,但没法直接用操作系统的阻塞I/O实现,如何实现呢?通过netpoller来桥接两者

#### 操作系统I/O模型
阻塞I/O,非阻塞I/O,多路复用

#### netpoller的接口设计以及具体实现
介绍接口设计时需要包含为什么这么设计,这就需要涉及到netpoller调用方的使用方式,netpoller的触发时机

### 数据结构

pollerDesc,Glist等介绍

### API
具体的API介绍,可以用伪代码和图片以及文字描述,不需要具体的代码

### 示例说明

介绍到此处,可以用具体实现来阐述例子中的问题
### 小结

### 拓展资料


## 参考链接
* [Morsing's Blog netpoller](https://morsmachine.dk/netpoller)
* [Morsing's Blog scheduler](https://morsmachine.dk/go-scheduler)
* [Go语言设计与实现-网络轮询器](https://draveness.me/golang/docs/part3-runtime/ch06-concurrency/golang-netpoller/)
