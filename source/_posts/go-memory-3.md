---
title: Go语言内存管理-内存释放
date: 2022-03-28
tags: Go
---
> 内存管理中涉及到内存释放的部分。文中使用Go版本为1.17.6

## 释放方法

在《Go语言内存管理-底层视角》中提到，位于go/src/runtime/mgcscavenge.go中pageAlloc的方法scavengeRangeLocked会使用sysUnused将一段内存从ready状态转为prepared状态，此时内存可以被操作系统回收，详细看下该方法：

```
// pageAlloc中根据chunks的索引可以查到一个pallocData,每个pallocData代表一个chunk的bitmap.bitmap中分两部分，一部分是8个uint64，共代表8*64 
// = 512个比特位(一个chunk正好为512个页)，每个比特位表明一个page是否使用，1为使用，0为未使用。第二部分是scavenged,代表一个page是否被回收，1为回
// 收，0为未回收
type pageAlloc struct {
	...
	chunks [13]*[13]pallocData
	...
}
type pallocData struct {
	pallocBits
	scavenged pageBits
}

type pallocBits pageBits
type pageBits [8]uint64


func (p *pageAlloc) scavengeRangeLocked(ci chunkIdx, base, npages uint) uintptr {
	assertLockHeld(p.mheapLock)
	// 通过某个chunk的index找到代表该chunk的bitmap(bitmap中每一位代表一个page,1为使用，0为未使用).设置scavenged比特位，最后使用sysUnused释放
	p.chunkOf(ci).scavenged.setRange(base, npages)

	// 通过chunk索引计算该块空间的地址
	addr := chunkBase(ci) + uintptr(base)*pageSize

	...

	// 调用sysUnused将该空间内存状态转换为prepared
	sysUnused(unsafe.Pointer(addr), uintptr(npages)*pageSize)
	...

	return addr
}
```

查看调用链，主要如下：
* gcenable->bgscavenge->scavenge->scavengeOne->scavengeRangeLocked
* mheap.grow->scavenge->scavengeOne->scavengeRangeLocked
第一条链路为后台gc进程，第二条链路为通过mheap扩展内存时触发，二者一个是主动一个是被动，首先看mheap扩容时的触发方式

## 被动触发

```
	if retained := heapRetained(); retained+uint64(totalGrowth) > h.scavengeGoal {
		todo := totalGrowth
		if overage := uintptr(retained + uint64(totalGrowth) - h.scavengeGoal); todo > overage {
			todo = overage
		}
		h.pages.scavenge(todo, false)
	}
```
查看已经申请的内存 + 本次申请的内存是否大于mheap中的scavengeGoal,scavengeGoal会根据GC触发策略动态变化，如果满足条件，则触发一次回收

## 主动触发

Go程序启动时就会执行gcenable函数，该函数启动一个goroutine执行bgscavenge()函数

bgscavenge()函数会根据权重计算逐步回收pages，但每次只回收一个page。如果没有回收任务可做，则通过调用gopark将自己调度出去。有两种情形可以再将回收goroutine调度执行，如下：
* 通过调用readyForScavenger（当执行完毕sweepone后调用)，会将一个标记位置为1，sysmon通过判断该标记位判断是否需要调度回收任务
* 当执行完所有spans的清扫后（finishsweep_m）会将回收任务调度执行

## 总结

本文主要讲了内存回收流程，最后讲到GC部分会主动回收。具体GC的介绍待后续文章讲解。