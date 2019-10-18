---
title: swig使用-go调用c++
date: 2019-10-11
tags:
 - go
 - c
 - c++ 
---
>通过swig可以将c/c++的接口导出,然后go可以直接调用

## 一个例子

```
MacBook-Pro-4:~ zhangshihua$ go get github.com/zacg/simplelib
can't build package github.com/zacg/simplelib because it contains C++ files (simpleclass.cpp) but it's not using cgo nor SWIG

```
继续执行

```
cd $GOPATH/github.com/zacg/simplelib
swig -go -cgo -c++ -intgosize 64 simplelib.i

```
看下swig的参数:

```
swig --help
...
-go             - Generate Go wrappers
...
-c++            - Enable C++ processing
...
```
看下swig go语言的特殊参数:
```
swig -go -help
     -cgo                - Generate cgo input files
     -no-cgo             - Do not generate cgo input files
     -gccgo              - Generate code for gccgo rather than gc
     -go-pkgpath <p>     - Like gccgo -fgo-pkgpath option
     -go-prefix <p>      - Like gccgo -fgo-prefix option
     -import-prefix <p>  - Prefix to add to %import directives
     -intgosize <s>      - Set size of Go int type--32 or 64 bits
     -package <name>     - Set name of the Go package to <name>
     -use-shlib          - Force use of a shared library
     -soname <name>      - Set shared library holding C/C++ code to <name>
```

执行测试集:

```
MacBook-Pro-4:simplelib zhangshihua$ go test -v
=== RUN   TestHello
--- PASS: TestHello (0.00s)
=== RUN   TestHelloString
--- PASS: TestHelloString (0.00s)
=== RUN   TestHelloBytes
--- PASS: TestHelloBytes (0.00s)
PASS
ok  	github.com/zacg/simplelib	0.007s
```

## 代码解释

原理可以通过追踪代码自行查看,我们首先看看simplelib.h头文件
```
#ifndef SIMPLECLASS_H
#define SIMPLECLASS_H

#include <iostream>
#include <vector>

class SimpleClass
{
public:
    SimpleClass(){};
    std::string hello();
    void helloString(std::vector<std::string> *results);
    void helloBytes(std::vector<char> *results);
};

#endif
```
c++定义了一个类SimpleClass,其中有三个方法,hello(),helloString(),helloBytes().我们首先通过编写一个swig接口文件simplelib.i导出这三个函数:
```
%module simplelib //go的包名

//%{}之中的数据直接导出到c++ wrapper中,即simplelib_wrap.cxx
%{
#include "simpleclass.h"
%}


//通过如下的语法将vector<string>类型导出为go中的StringVector结构,vector<char>导出为ByteVector结构
%include <typemaps.i>
%include "std_string.i"
%include "std_vector.i"

// This will create 2 wrapped types in Go called
// "StringVector" and "ByteVector" for their respective
// types.
namespace std {
   %template(StringVector) vector<string>;
   %template(ByteVector) vector<char>;
}

%include "simpleclass.h"
```
调用'swig -go -cgo -c++ -intgosize 64 simplelib.i'命令生成'simplelib_wrap.cxx'和'simplelib.go'文件.

编写如下测试代码:
```
package main

import (
	"fmt"
	"github.com/zacg/simplelib"
)

func main() {

//调用Hello方法
	simpleClass := simplelib.NewSimpleClass()
	result := simpleClass.Hello()
	fmt.Println(result)

//调用HelloString方法
	strings := simplelib.NewStringVector()
	simpleClass.HelloString(strings)

	var i int64
	for i = 0; i < strings.Size(); i++ {
		fmt.Println(strings.Get(int(i)))
	}

//调用HelloBytes方法
	bytes := simplelib.NewByteVector()
	simpleClass.HelloBytes(bytes)

	for i = 0; i < bytes.Size(); i++ {
		fmt.Printf("%c", bytes.Get(int(i)))
	}
	fmt.Println("")

}
```
可以看到,调用时与使用go语言写的库没什么区别.甚至把cgo的逻辑都已经隐藏.

## 增加一个回调

如果c++代码中设置了回调类,回调类中包括各种阶段的回调函数,如何用go实现这个回调类和回调函数呢?

c++代码修改如下:
```
class SimpleClassCallback
{
    public:
    virtual ~SimpleClassCallback() {}
    virtual void onStart() {};
};
```
增加了一个c++类SimpleClassCallback,其中有一个函数onStart

```
class SimpleClass
{
public:
    SimpleClass(){};
    std::string hello();
    void helloString(std::vector<std::string> *results);
    void helloBytes(std::vector<char> *results);
    void setCallBack(SimpleClassCallback *scc);
};

```
SimpleClass中增加一个函数setCallBack,参数为增加的新类SimpleClassCallback.然后增加一个实现:

```
void SimpleClass::setCallBack(SimpleClassCallback *scc){
	std::cout << "world from callback!" << "\n";
	scc->onStart();
}
```
输出"world from callback!"并且调用onStart方法.那么go中如何实现onStart方法呢？
首先需要修改swig接口文件:
```
%module(directors="1") simplelib

%feature("director") SimpleClassCallback;
```
增加feature语句表明需要被实现的类,module语句后边增加'directors="1"'

然后实现如下go代码:
```
package simplelib

import "fmt"

type SCCGo interface {
	SimpleClassCallback
	deleteSimpleClassCallback()
}

type sccGo struct {
	SimpleClassCallback
}

func (scc *sccGo) deleteSimpleClassCallback() {
	DeleteDirectorSimpleClassCallback(scc.SimpleClassCallback)
}


type overwrittenMethodsOnSimpleClassCallback struct {
	scc SimpleClassCallback
}

func (om *overwrittenMethodsOnSimpleClassCallback) OnStart()  {
	fmt.Println("Go Function")
}


func NewSCCGo() SCCGo {
	
	om := &overwrittenMethodsOnSimpleClassCallback{}
	scc := NewDirectorSimpleClassCallback(om)
	om.scc = scc

	scci := &sccGo{SimpleClassCallback: scc}
	return scci
}

func DeleteSCCGo(scc SCCGo) {
	scc.deleteSimpleClassCallback()
}
```
解释如下(注意该写法完全参照链接2中示例,官方文档也强烈建议需要按这种方法编写):
* 定义SCCGo接口,是SimpleClassCallback的超类.调用时通过NewSCCGo调用,返回的就是该接口类型
* 调用回调类时按NewDirectorSimpleClassCallback,传入的参数是一个实现了SimpleClassCallback类方法的结构体.
* sccGo实现了SCCGo接口

调用swig命令生成新的wrapper
```
swig -go -cgo -c++ -intgosize 64 simplelib.i
```
然后再代码中就可以如下使用:

```
	c := simplelib.NewSCCGo()
	simpleClass.SetCallBack(c)
	simplelib.DeleteSCCGo(c)
```

```
MacBook-Pro-4:simplelib zhangshihua$ go run simple.go
world
world
world
world from callback!
Go Function
```

## 参考文档

* https://www.cnblogs.com/terencezhou/p/10059156.html
* https://github.com/swig/swig/tree/master/Examples/go/director
* http://www.swig.org/Doc3.0/Go.html#Go_director_foobargo_class


