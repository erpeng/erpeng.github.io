---
title: GO程序平滑升级
date: 2019-09-21
tags: go
---

>平滑升级或者优雅升级或者优雅重启的目的是不影响当前正在执行请求的情况下实现升级或者重启.上一讲我们说了NGINX的平滑重启,本文以gracehttp说明GO的优雅升级.

## 两个关键步骤

我们以两个例子来说明平滑升级的两个关键节点:
* 如何让新的进程能够监听并且执行请求
* 如何让旧的进程不再接收请求

### 新进程监听接收请求


``` test.c
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>


int main(int argc,char *argv[])
{

   int fd = -1;
   int sz;
   char buf[128];

   fd = open("./helloworld.txt",O_RDONLY);

   sz = read(fd,buf,5);
   if(sz < 0)
   {
      printf("read failure\n");
      exit(-1);
   }

  buf[sz] = 0;
  printf("test buf: %s\n",buf);

  sprintf(buf,"%d",fd);

  execl("./read","read",buf,NULL);

  return 0x0;

}

```

```read.c
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>


int main(int argc,char *argv[])
{
    int fd = -1;
    char buf[128];
    int sz;

    if(argc < 2)
    {
       printf("argc must equal to 2\n");

       return -1;
    }

    fd = atoi(argv[1]);
    if(fd < 0)
    {
       printf("error fd:%d\n",fd);
       return -2;
    }

    sz = read(fd,buf,16);
    if(sz < 0)
    {
       printf("read error\n");
       return -3;
    }

    buf[sz] = 0;
    printf("read buf:%s\n",buf);

    return 0x0;
}

```

```
[root@localhost test-src]# cat helloworld.txt 
hello,world
[root@localhost test-src]# gcc -o test test.c
[root@localhost test-src]# gcc -o read read.c
[root@localhost test-src]# ./test
test buf: hello
read buf:,world
```
简单解释一下,helloworld.txt中文本为"hello,world".第一个文件test.c读取了五个字符,"hello",然后以exec系列函数启动第二个文件read.c,read.c中继续读取helloworld.txt,可以看到读出了",world"6个字符

即A进程通过exec系列函数启动另一个B进程后,B进程会继承A进程的文件描述符,可以继续使用A进程的文件描述符读取数据.参考上节中NGINX的平滑升级,正是利用该特性使得新进程能够监听并且接收请求.

### 旧有进程不再接收请求并退出

```
package main

import (
	"net"
	"fmt"
	"time"
)

func main(){

    ln, err := net.Listen("tcp", ":8080")
    if err != nil {
    }
    go closeListener(ln)
    for {
        fmt.Println("begin\n");
        conn, err := ln.Accept()
        if err != nil {
            fmt.Println(err)
            return
        }
        fmt.Println("end\n");
        go handleConnection(conn)

    }

}

func handleConnection(c net.Conn){
	fmt.Println("ok start\n");
	time.Sleep(10*time.Second);
	fmt.Println("ok\n");
}

func closeListener(ln net.Listener){
	time.Sleep(5*time.Second)
	ln.Close()
	fmt.Println("time close")
}
```

起一个程序监听8080端口,在一个for循环中进行accept,正常情况下如果没有请求会阻塞到accept.每次有新的请求时起一个goroutine处理,并且单独起一个goroutine5s钟之后将listener关闭.我们看下执行情况

```
bogon:~ zhangshihua$ go run net.go
begin

time close
accept tcp [::]:8080: use of closed network connection
```
可以看到,正常情况下阻塞到accept,当5s钟之后关闭listener之后,accept不再阻塞,输出错误,"accept tcp [::]:8080: use of closed network connection".

所以只要关闭listener,旧的进程就不会再处理新来的请求.

## gracehttp实现

代码分三部分,分别是gracehttp,gracenet以及httpdown,先看下gracenet中的StartProcess.

```
func (n *Net) StartProcess() (int, error) {
	listeners, err := n.activeListeners()
	if err != nil {
		return 0, err
	}

	// Extract the fds from the listeners.
	files := make([]*os.File, len(listeners))
	for i, l := range listeners {
		files[i], err = l.(filer).File()
		if err != nil {
			return 0, err
		}
		defer files[i].Close()
	}   //从listener中把fd取出

	// Use the original binary location. This works with symlinks such that if
	// the file it points to has been changed we will use the updated symlink.
	argv0, err := exec.LookPath(os.Args[0])
	if err != nil {
		return 0, err
	}

	// Pass on the environment and replace the old count key with the new one.
	var env []string
	for _, v := range os.Environ() {
		if !strings.HasPrefix(v, envCountKeyPrefix) {
			env = append(env, v)
		}
	}
	env = append(env, fmt.Sprintf("%s%d", envCountKeyPrefix, len(listeners))) //导出监听的listerner

	allFiles := append([]*os.File{os.Stdin, os.Stdout, os.Stderr}, files...)
	process, err := os.StartProcess(argv0, os.Args, &os.ProcAttr{    //启动一个进程
		Dir:   originalWD,
		Env:   env,
		Files: allFiles,
	})
	if err != nil {
		return 0, err
	}
	return process.Pid, nil
}
```

同样的处理方法,将所有listener中的文件描述符导出

接着看httpdown中的Stop方法:

```
func (s *server) Stop() error {
        ...

		// first disable keep-alive for new connections
		s.server.SetKeepAlivesEnabled(false)

		// then close the listener so new connections can't connect come thru
		closeErr := s.listener.Close() //关闭所有旧进程的listener
		<-s.serveDone

		// then trigger the background goroutine to stop and wait for it
		stopDone := make(chan struct{})
		s.stop <- stopDone

	    ...
}
```

旧进程关闭了listener不再处理请求,新进程启动时导出了所有监听文件描述符,那么导出之后新进程启动时如何处理的呢

```
// ListenTCP announces on the local network address laddr. The network net must
// be: "tcp", "tcp4" or "tcp6". It returns an inherited net.Listener for the
// matching network and address, or creates a new one using net.ListenTCP.
func (n *Net) ListenTCP(nett string, laddr *net.TCPAddr) (*net.TCPListener, error) {
	if err := n.inherit(); err != nil {
		return nil, err
	}

	n.mutex.Lock()
	defer n.mutex.Unlock()

	// look for an inherited listener
	for i, l := range n.inherited {
		if l == nil { // we nil used inherited listeners
			continue
		}
		if isSameAddr(l.Addr(), laddr) {
			n.inherited[i] = nil
			n.active = append(n.active, l)
			return l.(*net.TCPListener), nil
		} //如果有从环境变量中找到的listener,就直接返回
	}

	// make a fresh listener
	l, err := net.ListenTCP(nett, laddr) //否则,新生成一个listener
	if err != nil {
		return nil, err
	}
	n.active = append(n.active, l)
	return l, nil
}
```
调用net.ListenTCP时如果有从环境变量继承的listener,就会直接返回这些listener,否则生成新的listener.

## 小结

可以看到gracehttp与nginx平滑升级的原理基本一致.