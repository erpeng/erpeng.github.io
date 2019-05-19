---
title: go sync包源码分析
date: 2019-05-19
tags: go
---
>go的sync.Once结构可以用在并发情况下只需要执行一次某个函数的情况

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

应该是once结构体中有个bool值决定是否已经执行过该函数.如果没有,首先加锁,然后执行函数,执行完毕之后修改bool值并且释放锁

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

## sync.Cond包

条件变量,控制goroutine间的同步位点
通过NewCond生成一个Cond,调用cond.Wait()之后阻塞,其他goroutine调用cond.Signal()或者cond.Broadcast()唤醒wait的一个或者全部goroutine