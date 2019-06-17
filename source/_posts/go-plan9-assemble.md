---
title: go汇编
date: 2019-06-13
tags: go
---
>go runtime中有不少汇编.因此有必要学习一下go的汇编语法


## 说明

go汇编器基于plan9汇编.注意该汇编不是底层机器的直接表示,而是一种半抽象的指令集.如果想看特定平台上边的汇编输出,可以看runtime或者math/big包,里边有许多示例.或者使用如下命令:

```
$ cat x.go
package main

func main() {
	println(3)
}
$ GOOS=linux GOARCH=amd64 go tool compile -S x.go        # or: go build -gcflags -S x.go

--- prog list "main" ---
0000 (x.go:3) TEXT    main+0(SB),$8-0
0001 (x.go:3) FUNCDATA $0,gcargs·0+0(SB)
0002 (x.go:3) FUNCDATA $1,gclocals·0+0(SB)
0003 (x.go:4) MOVQ    $3,(SP)
0004 (x.go:4) PCDATA  $0,$8
0005 (x.go:4) CALL    ,runtime.printint+0(SB)
0006 (x.go:4) PCDATA  $0,$-1
0007 (x.go:4) PCDATA  $0,$0
0008 (x.go:4) CALL    ,runtime.printnl+0(SB)
0009 (x.go:4) PCDATA  $0,$-1
0010 (x.go:5) RET     ,
...
```

其中可以看到第3条指令将3放入栈,第5条指令调用runtime.printint输出3,然后调用runtime.printnl输出一个换行符
FUNCDATA和PCDATA指令生成垃圾回收器需要的一些信息.后文叙述
## 常量

常量使用go的操作符优先级生成,例如3&1<<2为(3&1)<<2为4.整型被处理为64bit无符号类型.

## 符号

伪寄存器:注意上文说过go汇编是一种半抽象的存在,伪寄存器的存在也能说明该问题.

FP:frame pointer:栈指针,用来获取参数和局部符号
PC:program counter:跳转和分支指令使用
SB:static base pointer:全局符号使用
SP:stack pointer:栈顶部

所有用户定义的符号都会按FP和SB的偏移获取,区别在于FP获取参数和局部符号,SB获取全局符号

SB可以理解为内存的内存的其实地址,foo(SB)代表foo在内存中的位置;<>表示符号的可见性只在当前文件中可见,foo<>(SB),foo+4(SB)表示foo的位置偏移4bytes

FP用来获取函数参数的位置.例如first_arg+0(FP)表示第一个参数,注意first_arg没有实际意义,但是汇编器强制要求这么写.second_arg+0(FP)表示第二个参数 

SP用来获取函数局部变量的地址,由于是栈顶,因此以负的偏移来表示.例如x-8(SP),y-4(SP).注意硬件结构中本身有一个SP寄存器,他和伪SP寄存器可以这样区分,前边带symbol的是伪寄存器,不带的为真实SP.例如x-8(SP)和-8(SP)分别表示伪寄存器和真实寄存器

注意代码中的fmt.Printf 和 math/rand.Int在汇编中为fmt·Printf 和 math∕rand·Int

## 指令

DATA/GLOBAL/TEXT指令
TEXT定义一个函数
DATA指令初始化一段内存,即给变量赋值
GLOBAL定义一个全局符号
格式如下:

```
TEXT symbol,flags,framesize-argumentsize //frame大小和参数大小(参数包括入参和返回值)
DATA	symbol+offset(SB)/width, value //width表示占用内存大小,value表示初始化的值
GLOBAL symbol,flags,datasize
```

示例如下:

```
DATA divtab<>+0x00(SB)/4, $0xf4f8fcff
DATA divtab<>+0x04(SB)/4, $0xe6eaedf0
...
DATA divtab<>+0x3c(SB)/4, $0x81828384
GLOBL divtab<>(SB), RODATA, $64

GLOBL runtime·tlsoffset(SB), NOPTR, $4
```
定义了一个全局变量divtab,可以理解为一个数组,共64个字节,元素个数为16,每个元素4字节,从上到下依次赋值.然后定义了一个4bytes的tlsoffset变量

flag:定义在textflag.h中
NOPROF:废弃 
DUPOK = 2 一个二进制文件中有多个该符号的实例是合法的.linker可以选择一个去使用
NOSPLIT = 4 TEXT指令使用,不分裂stack.不会往函数前插入检查是否需要split栈的代码.
RODATA = 8 DATA和GLOBAL指令使用,将数据放入只读区
NOPTR = 16 DATA和GLOBAL指令使用,该数据中不包括指针,因此不需要垃圾回收器扫描
WRAPPER = 32  TEXT使用,wrapper function？
NEEDCTXT = 64 TEXT使用,闭包函数,因此使用输入上下文寄存器

## runtime协同
为了使垃圾回收器正确运行,runtime必须知道所有全局数据中和栈帧中的指针位置.

一般一个标记为NOPTR或者RODATA或者大小小于指针的可以认为不包括运行时分配的指针.
如果一个函数的返回值包括指针,函数应该首先初始化返回值并且执行伪指令GO_RESULTS_INITIALIZED.该指令表明结果已经被初始化并且应该在stack movement或者垃圾回收时被扫描

汇编函数需要给出go原型,既是为了提供参数和返回值的指针信息也是go vet检查获取他们时的offset是正确的


## CPU体系相关的细节

amd64:

```
get_tls(CX)
MOVQ	g(CX), AX     // Move g into AX.
MOVQ	g_m(AX), BX   // Move g.m into BX.

```

get_tls和g的宏定义如下:

```
#ifdef GOARCH_amd64
#define	get_tls(r)	MOVQ TLS, r
#define	g(r)	0(r)(TLS*1)
#endif
```


## 参考文档
* https://golang.org/doc/asm
* https://chai2010.cn/advanced-go-programming-book/ch3-asm/ch3-06-func-again.html