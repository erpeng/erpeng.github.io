---
title: cgo使用笔记
date: 2019-09-29
tags:
 - go
 - c
 - c++ 
---
>go如何调用c和c++.主要参考:https://chai2010.cn/advanced-go-programming-book/ch2-cgo/readme.html

## 语法

```
package rand

/*
#include <stdlib.h>
*/
import "C"

func Random() int {
    return int(C.random())
}

func Seed(i int) {
    C.srandom(C.uint(i))
}
```
几个关键点:
* import "C" C是一个伪包,代表C代码部分的命名空间.
* 注释部分的include <stdlib.h>编译c代码部分时会作为文件头,此部分可以为任意的c代码.注释类型的还有 #cgo指令,可以指定编译链接的一些选项.例如:
```
// #cgo CFLAGS: -DPNG_DEBUG=1 -I./include
// #cgo LDFLAGS: -L/usr/local/lib -lpng
// #include <png.h>
import "C"
```
* 注意Random部分需要将返回值从C.random()的long类型转换为go可以使用的int类型 


## 调用c++

hello.h
```
// hello.h
void SayHello(const char* s);
```

hello.cpp
```
#include <iostream>

extern "C" {
    #include "hello.h"
}

void SayHello(const char* s) {
    std::cout << "From C++:";
    std::cout << s;
}
```
hello.go
```
// hello.go
package main

//#include <hello.h>
import "C"

func main() {
    C.SayHello(C.CString("Hello, World\n"))
}
```

运行:
```
[root@zhangshihua003 e2]# go build .
[root@zhangshihua003 e2]# ls
e2  hello.cpp  hello.go  hello.h
[root@zhangshihua003 e2]# ./e2
From C++:Hello, World
```


## 类型转换
参考https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-03-cgo-types.html
即c语言空间和go语言空间的类型转换

## C调用Go函数

设想一种情况,c函数中的参数为一个回调函数.如果该函数使用go语言编写,那么显然cgo需要支持c语言调用go函数

hello.h
```
// hello.h
void SayHello(const char* s);
```

hello.go
```
package main

import "C"

import "fmt"

//export SayHello
func SayHello(s *C.char) {
    fmt.Print("From Go:"+C.GoString(s))
}
```
注意//export SayHello该条语句,会导出一个go函数SayHello

hello1.go
```
package main

//#include <hello.h>
import "C"

func main() {
    C.SayHello(C.CString("Hello, World\n"))
}
```

运行:

```
[root@zhangshihua003 e3]# ls
e3  hello.go  hello.h  hello1.go
[root@zhangshihua003 e3]# ./e3
From Go:Hello, World
```

## go调用动态库

hello.h
```
// hello.h
void SayHello(const char* s);

```

hello.c
```
// hello.c

#include <stdio.h>
#include "hello.h"

void SayHello(const char* s) {
    printf("From C:");
    puts(s);
}

```
生成动态库
```
gcc -shared -o libhello.so -fPIC hello.c
```

调用动态库
hello.go
```
package main

// #cgo CFLAGS: -I.
// #cgo LDFLAGS: -L${SRCDIR} -lhello
// #include "hello.h"
import "C"

func main() {
    C.SayHello(C.CString("Hello, World\n"))
}
```
执行:
```
[root@zhangshihua003 e5]# export LD_LIBRARY_PATH=$LD_LIBRARY_PATH:.
[root@zhangshihua003 e5]# go build .
[root@zhangshihua003 e5]# ./e5
From C:Hello, World
```


## 参考连接
* https://chai2010.cn/advanced-go-programming-book/ch2-cgo/ch2-01-hello-cgo.html
* https://golang.org/cmd/cgo/
* https://blog.golang.org/c-go-cgo
* https://golang.org/src/runtime/cgocall.go
