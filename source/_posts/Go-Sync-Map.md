---
title: Go sync.Map笔记
date: 2020-08-04 
tags:
 - go
 - sync.Map
---


## 基本原理 

### 数据结构

```
type Map struct {
	mu Mutex
    read atomic.Value
	dirty map[interface{}]*entry
   	misses int
}

type readOnly struct {
	m       map[interface{}]*entry
	amended bool 
}

var expunged = unsafe.Pointer(new(interface{}))

type entry struct {
	p unsafe.Pointer
}

```


首先看看哪些数据的操作需要加锁,哪些不需要加锁
* Map.read是一个原子数据,可以直接使用load获取,store保存,因此不需加锁
* Map.dirty是一个go中的map结构,因此并发读写时需要进行加锁操作
* Map.read实际存储的是一个readOnly结构,readOnly中的m也是一个go的map,**readOnly的m只读,不会有写的操作**,因此Map.read.m也不需要加锁
* Map中的key表示为一个interface{},值表示为一个entry指针,该指针包含一个p字段,为unsafe.Pointer,go中操作unsafe.Pointer类型有 atomic.LoadPointer和atomic.StorePointer两个原子操作的函数,因此操作Map.read.m[key].p也不需要加锁

搞清楚什么时候需要加锁,什么时候不需要,sync.Map的理解就已经完成了50%

### 关系

readOnly中的amended表示Map.dirty中是否有Map.read.m中没有的key,true表示是,false表示否



* Map.read和Map.dirty的关系
写入新的key/value对时,首先会写入dirty,当到达一定的时机时,会将Map.read.m整体置换为dirty,然后将dirty置为nil,注意此时Map.read中的amened会置为false.即Map.dirty为nil,Map.read.amended为false,Map.read.m中为全部的key

* expunged的作用
    * 上文提到加锁时说Map.read.m只有读操作,没有写操作,因此不需要加锁.那么删除操作如何进行呢?实际上对Map.read.m进行删除时,只会将Map.read.m[key].p置为nil
    * 考虑写入的情景,假设此时Map.read.m刚刚整体置换为dirty,此时如果写入一个Map.read.m中不存在的key,首先需要写入dirty,而dirty此时为nil,因此需要首先将Map.read.m中的非nil(也就是没有删除的键)的键值拷贝一份到Map.dirty,并且将Map.read.m中的nil原子置换为expunged
expunged表示一个key在read中但是不在dirty中

### 代码

理清楚加锁时机以及Map.read及Map.dirty的关系以及expunged之后代码就比较好理解了,不再赘述

## Q&A

* 只有Map.read是否可以,不需要dirty字段?
load可以原子操作,没问题,store也可以原子操作,但是delete时如果置为nil,下次load会有问题.如果直接删除,那么Map.read.m就有了写操作,需要加锁

## 感想

整体代码逻辑比较复杂,并且如果写入比较频繁,多次触发read到dirty的拷贝,是一个O(N)操作,效率不高.因此适用于读多写少的场景





