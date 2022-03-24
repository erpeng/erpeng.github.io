---
title: Go语言内存管理-底层视角
date: 2022-03-21
tags: Go
---
> 内存管理涉及初始化、分配以及回收三个方面。文中使用Go版本为1.17.6

## 系统调用

GO使用mmap和munmap调用操作系统进行内存分配和释放。mmap和munmap原型如下：
```
       #include <sys/mman.h>

       void *mmap(void *addr, size_t length, int prot, int flags,
                  int fd, off_t offset);
       int munmap(void *addr, size_t length);

```
mmap中addr代表建议分配的地址，如果为NULL，则由kernel自己选择地址；length为分配的内存大小；prot描述内存的安全性，包括PROT_EXEC(可执行)，PROT_READ（可读），PROT_WRITE(可写)，PROT_NONE(不可访问)；flags比较多，此处只需要关注MAP_FIXED，代表必须从addr开始的位置进行分配，fd与offset代表映射一个文件到内存中，Go中未使用。nummap中addr和length代表需要释放的内存地址和大小。

Go还会使用madvise给kernel一些建议，通过这种方式提高系统或者应用性能。madvise原型如下：
```
       #include <sys/mman.h>

       int madvise(void *addr, size_t length, int advice);
```
addr与length仍然是内存地址与大小，advice关注MADV_HUGEPAGE（适用于Linux 2.6.38之后，kernel收到这个建议之后会使用大页管理分配的内存空间），MADV_FREE（kernel收到此建议后，当有内存压力时，可以释放指定内存空间的pages。注意此时内存空间仍然可以写入，并且之后的应用写入不会丢失）

## 内存状态

Go通过抽象出四种内存状态屏蔽底层具体的操作系统实现，四种状态通过调用函数可以相互转换，具体如下图所示：
![memory](/img/Go-memory-state.png)
关注两条转换线路，一条是标绿的sysAlloc和sysFree，其底层实现如下（状态转移图中函数均位于src/runtime/mem_linux.go文件）：
```
func sysAlloc(n uintptr, sysStat *sysMemStat) unsafe.Pointer {
	p, err := mmap(nil, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
    ...
	return p
}
func sysFree(v unsafe.Pointer, n uintptr, sysStat *sysMemStat) {
    ...
	munmap(v, n)
}
```
调用sysAlloc，内存状态为Ready，可以直接使用，调用sysFree释放内存，不能再使用
另一条紫色线路，分别调用sysReserve转换为Reserved状态，此时该内存不可以使用，sysReserve函数如下：
```
func sysReserve(v unsafe.Pointer, n uintptr) unsafe.Pointer {
	p, err := mmap(v, n, _PROT_NONE, _MAP_ANON|_MAP_PRIVATE, -1, 0)
    ...
	return p
}
```
注意使用mmap时prot为PROT_NONE，因此该内存不能访问。继续调用sysMap转换为Prepared状态：
```
func sysMap(v unsafe.Pointer, n uintptr, sysStat *sysMemStat) {
    ...
	p, err := mmap(v, n, _PROT_READ|_PROT_WRITE, _MAP_ANON|_MAP_FIXED|_MAP_PRIVATE, -1, 0)
    ...
}
```
此时flag中有MAP_FIXED，并且指定了addr和length字段，从操作系统层面该内存空间已经可用，那么Prepared状态和Ready状态有啥区别呢？
madvise可以上场了，我们知道madvise可以用来在内存管理中提高系统或者应用性能，在Go中Prepared状态的内存仍然不可使用，需要继续调用sysUsed将状态转换为Ready，sysUsed函数如下：
```
func sysUsed(v unsafe.Pointer, n uintptr) {
	sysHugePage(v, n)
}

func sysHugePage(v unsafe.Pointer, n uintptr) {
	if physHugePageSize != 0 {
		beg := alignUp(uintptr(v), physHugePageSize)
		end := alignDown(uintptr(v)+n, physHugePageSize)
		if beg < end {
			madvise(unsafe.Pointer(beg), end-beg, _MADV_HUGEPAGE)
		}
	}
}
```
使用MADV_HUGEPAGE建议内核如果内存比较大，使用大页管理。同理，sysUnUsed除了大页管理，还会设置MADV_FREE，告诉kernel如果内存告急，可以回收该内存。


## Go内存分配组件

### 绿色线路

通过全局搜索sysAlloc，可以观察Go在什么情况下会直接调用mmap分配内存空间。首先是src/runtime/malloc.go中的persistentalloc函数，通过该函数分配的内存空间会转换为notInHeap类型，定义如下：

```
type notInHeap struct{}

```
Go内部内存管理结构体或者debug相关的内存需求都是通过persistentalloc进行分配，notInHeap也就是不在堆上，这些数据不会进行释放，除非Go程序退出。该函数分配时通过四个步骤获取内存：
* 如果单次分配大于64K,直接调用sysAlloc后返回地址
* 从每个p结构体的palloc成员变量获取，该成员变量类型为persistentAlloc，此时不需要加锁
```
type persistentAlloc struct {
	base *notInHeap   // 基地址
	off  uintptr      // 偏移，分配时从偏移处开始
}
```
* 从一个全局变量globalAlloc分配，此时需要加锁
```
var globalAlloc struct {
	mutex // 锁
	persistentAlloc // 全局存储空间
}
```
* 如果上述空间不足，则调用sysAlloc分配256K空间，保存到链表中，并且赋值给globalAlloc.persistentAlloc

调用persistentalloc函数的地方很多，分配各种元数据都会用到

## 紫色线路

sysReserve：主要位于go/src/runtime/malloc.go中mheap的方法sysAlloc，该方法会分配 heap arena空间，空间大小为64MB，分配完毕后的空间状态为Reserved
sysMap：主要位于go/src/runtime/mheap.go中mheap的方法grow，grow方法会从mheap当前的heap arena分配空间，如果空间不够，则会调用mheap的sysAlloc方法继续分配heap arena
sysUsed：主要位于go/src/runtime/mheap.go中mheap的方法allocSpan
sysUnUsed：主要位于go/src/runtime/mgcscavenge.go中pageAlloc的方法scavengeRangeLocked，调用mheap的grow函数时会进行判断是否需要scavenge。



## 总结

本文从最底层的系统调用逐步追踪到上层Go的内存管理。Go内存管理整体思路类似TCMALLOC，分级管理，减少锁的竞争。涉及到的数据结构包括mspan，mcentral，mheap，mheap通过heapArena管理pages。详细细节下篇文章介绍。


