---
title: go sync包源码分析
date: 2019-05-19
tags: go
---
>go的sync包中所有的结构都适用于goroutine并发执行的情况

## sync.Once包

### 一个示例
```
package main

import (
	"fmt"
	"sync"
)

func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once")
	}
	done := make(chan bool)
	for i := 0; i < 10; i++ {
		go func() {
			once.Do(onceBody)
			done <- true
		}()
	}
	for i := 0; i < 10; i++ {
		<-done
	}
}

```

虽然起10个goroutine去调用onceBody,但是使用once.Do函数将onceBody包裹之后,onceBody只会执行一次.运行该代码,结果如下:
```
Only once
```
### 代码分析

#### 设想一下原理

应该是once结构体中有个bool值决定是否已经执行过该函数.如果没有执行,首先加锁,然后执行函数,执行完毕之后修改bool值并且释放锁

#### sync.Once结构

```
type Once struct {
	m    Mutex
	done uint32
}
```
一个锁,一个done的uint32值

```
func (o *Once) Do(f func()) {
	if atomic.LoadUint32(&o.done) == 1 {
		return
	}
	// Slow-path.
	o.m.Lock()
	defer o.m.Unlock()
	if o.done == 0 {
		defer atomic.StoreUint32(&o.done, 1)
		f()
	}
}
```
加锁,执行,最后置done标记.done为uint32估计是因为可以使用atomic包中的LoadUint32和StoreUint32原子性的加载和存储.

## sync.Pool包

New字段定义如何生成对象,然后可以通过pool.Get()和pool.Put()获取和放回对象.一系列临时对象的对象池

关键结构体如下:
```
type Pool struct {
	noCopy noCopy

	local     unsafe.Pointer  
	localSize uintptr        
	New func() interface{}
}
```
local可以理解为指向一个poolLocal数组,大小由localSize指定(localSize由cpu cores决定).
New指定一个函数,用来创建Pool中的对象

```
type poolLocal struct {
	poolLocalInternal
	pad [128 - unsafe.Sizeof(poolLocalInternal{})%128]byte
}
type poolLocalInternal struct {
	private interface{}   // Can be used only by the respective P.
	shared  []interface{} // Can be used by any P.
	Mutex                 // Protects shared.
}
```
poolLocal中关键的字段为private和shared,private保存只能由相应的P(go调度器中概念,P/M/G)使用的对象.shared中保存可以由所有的P共同使用的对象

我们看一下Put的代码
```
func (p *Pool) Put(x interface{}) {
	if x == nil {
		return
	}
	...
	l := p.pin()
	if l.private == nil {
		l.private = x
		x = nil
	}
	runtime_procUnpin()
	if x != nil {
		l.Lock()
		l.shared = append(l.shared, x)
		l.Unlock()
	}
	...
}
```
关键路径为首先将要放回的对象放入poolLocal中的private,否则放入共享的shared结构.

看一下Get()的代码路径
```
func (p *Pool) Get() interface{} {
	...
	l := p.pin()
	x := l.private
	l.private = nil
	runtime_procUnpin()
	if x == nil {
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
		}
		l.Unlock()
		if x == nil {
			x = p.getSlow()
		}
	}
	...
	if x == nil && p.New != nil {
		x = p.New()
	}
	return x
}
```
首先从poolLocal的private中获取,如果未获取到则获取shared的最后一个元素,否则调用getSlow函数.
依然获取不到的话调用pool结构中的New函数生成.

```
func (p *Pool) getSlow() (x interface{}) {
	size := atomic.LoadUintptr(&p.localSize) // load-acquire
	local := p.local                         // load-consume
	pid := runtime_procPin()
	runtime_procUnpin()
	for i := 0; i < int(size); i++ {
		l := indexLocal(local, (pid+i+1)%int(size))
		l.Lock()
		last := len(l.shared) - 1
		if last >= 0 {
			x = l.shared[last]
			l.shared = l.shared[:last]
			l.Unlock()
			break
		}
		l.Unlock()
	}
	return x
}
```
getSlow函数会从当前P的下一个P开始依次遍历其他poolLocal中的shared结构去获取对象.

### 问题
* runtime_procPin()会返回当前P的pid,需要了解go scheduler相关知识加深理解
* pool中会注册一个runtime_registerPoolCleanup(poolCleanup),与Go GC相关,需了解go GC相关知识



## sync.Cond包

条件变量,控制goroutine间的同步位点
通过NewCond生成一个Cond,调用cond.Wait()之后阻塞,其他goroutine调用cond.Signal()或者cond.Broadcast()唤醒wait的一个或者全部goroutine

### 一个示例
```
    c.L.Lock()
    for !condition() {
        c.Wait()
    }
    ... make use of condition ...
    c.L.Unlock()
```

一个goroutine中要执行Wait之前,首先将cond中的c.L.Lock()

### 关键结构体和方法
```
type Cond struct {
	noCopy noCopy
	// L is held while observing or changing the condition
	L Locker
	notify  notifyList
	checker copyChecker
}
```

```
func NewCond(l Locker) *Cond {
	return &Cond{L: l}
}
```
调用NewCond需要传入一个实现了Locker接口的结构

```
func (c *Cond) Wait() {
	c.checker.check()
	t := runtime_notifyListAdd(&c.notify)
	c.L.Unlock()
	runtime_notifyListWait(&c.notify, t)
	c.L.Lock()
}
```
Wait函数首先调用runtime将c.notify加入notifyList,然后解锁,然后等待.当条件变量满足后首先获取锁后Wait才会返回

```

func (c *Cond) Signal() {
	c.checker.check()
	runtime_notifyListNotifyOne(&c.notify)
}
```
Signal()函数唤醒一个在等待的goroutine

```
func (c *Cond) Broadcast() {
	c.checker.check()
	runtime_notifyListNotifyAll(&c.notify)
}
```

Broadcast唤醒全部等待的goroutine

### 问题

* 进一步的实现依赖于runtime,需要熟悉runtime