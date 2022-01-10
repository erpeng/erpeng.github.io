---
title: 如何写一篇技术文章
date: 2022-01-10
tags: 
 - GO
 - Writing
---
>提炼一种能够写出通熟易懂技术文章的方法,以netpoller为例

## Morsing's Blog

### The Go Netpoller
Morsing' Blog中关于netpoller的文章在码农桃花源讲调度器以及Draven的<<Go语言设计与实现>>中都有引用,我们先看看博客是如何写的
共三段,都是文字.第一部分引言忽略,第二部分为Blocking

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

第二段讲述每一个文件句柄都会设置称非阻塞的,如果一个文件句柄没有ready,就会直接返回错误码.此时通过调度器和netpoller去把goroutine调度出去,直到有数据时再调度回来.

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





## 参考链接
* [Morsing's Blog netpoller](https://morsmachine.dk/netpoller)
* [Morsing's Blog scheduler](https://morsmachine.dk/go-scheduler)