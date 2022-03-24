---
title: Go语言内存管理-顶层视角
date: 2022-03-24
tags: Go
---
> 内存管理涉及初始化、分配以及回收三个方面。文中使用Go版本为1.17.6

## 概览

为减少分配时的竞争，go内存管理借鉴TCMALLOC的分级管理思想，每一个P（G-M-P，参考Go的调度管理）有自己的内存结构mcache，同一个P内的G分配内存时不需要加锁。当P内无内存可分配时，通过加锁，使用mcenter中的内存，当mcenter中也无内存时，通过mheap申请新的内存。
mheap结构中通过heapArena管理内存申请，每一个heapArena大小为64M，包括8192个pages，每个page的大小为8KB。
mcache中具体的内存管理单元是mspan，mspan分为68个级别，从8字节到32KB到大对象，如果分配一个13字节的对象，则需要使用一个大小为16字节的mspan
一个mspan可以占用一个或者多个page。总体概览图如下：
![level](/img/go-memory-level.png)

结构体的关键成员如下所示：

```
type mspan struct {
	...
	startAddr uintptr // span的开始地址
	npages    uintptr // span包括的pages个数
	freeindex uintptr // 从freeindex开始寻找下一个空闲的元素位置
	nelems uintptr // span可以保存的元素个数

	
	allocCache uint64 // 辅助快速寻找空闲元素位置
	allocBits  *gcBits // 位图，标记一个元素位置是否使用
	gcmarkBits *gcBits // 位图，标记一个元素位置是否已经进行过标记

	sweepgen    uint32 // h是mheap结构，根据他两的sweepgen，判断当前span的状态

	// if sweepgen == h->sweepgen - 2, the span needs sweeping
	// if sweepgen == h->sweepgen - 1, the span is currently being swept
	// if sweepgen == h->sweepgen, the span is swept and ready to use
	// if sweepgen == h->sweepgen + 1, the span was cached before sweep began and is still cached, and needs sweeping
	// if sweepgen == h->sweepgen + 3, the span was swept and then cached and is still cached
	// h->sweepgen is incremented by 2 after every GC
	... 
	allocCount  uint16        // 已经分配的元素个数
	spanclass   spanClass     // size class and noscan (uint8)
	...
	elemsize    uintptr       // 元素大小
	limit       uintptr       // span结尾
	...
}

type mcache struct {
	...
	tiny       uintptr // 微小对象分配（<16字节）地址
	tinyoffset uintptr // 当前分配的偏移量 
	tinyAllocs uintptr // 分配的个数
	alloc [numSpanClasses]*mspan // 所有的span
	...
	flushGen uint32
}

type mcentral struct {
	spanclass spanClass // 类型
	partial [2]spanSet // 有空闲元素位置的列表
	full    [2]spanSet // 无空闲元素位置的列表
}

type mheap struct {

	lock  mutex
	pages pageAlloc // page分配管理

	sweepgen     uint32 // sweep generation, see comment in mspan; written during STW
	sweepDrained uint32 // all spans are swept or are being swept
	sweepers     uint32 // number of active sweepone calls

	allspans []*mspan // 所有的mspan


	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena // 所有的heapArena


	arenaHints *arenaHint // 指向heapArena的地址数组
	allArenas []arenaIdx

	curArena struct {
		base, end uintptr
	} // 当前可分配空间的heapArena

	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	} // mcenter数组

	spanalloc             fixalloc // 分配span管理结构的分配器
	cachealloc            fixalloc // allocator for mcache*
	arenaHintAlloc        fixalloc // allocator for arenaHints
	...
}
```

## 页分配器概览

为了加快从mheap中申请pages的速度，1.17.6中的page allocator使用了一个分为5层的radix tree，tree中的每个叶子节点为一个chunk的summary，一个chunk包括512个pages，summary其实就是一个bitmap的汇总，每一个bit代表一个page是否使用，1为使用，0为未使用。summary会汇总一个chunk开头（start）、结尾（end）以及中间最大的连续0的个数（max），即可分配的连续pages个数。每8个叶子节点会再向上汇总，再形成一个8 chunks的summary，包括的内容还是start、max、end，注意当8个chunks汇总时，两个chunk的start和end可以合并，如果其值大于max，需要更新max的值。
为什么是5层呢，因为按5层计算，所有的叶子节点可表示的内存空间为 2^48字节，正好是48位地址总线能够寻找的最大地址。

![level](/img/go-memory-page-allocator.png)

## 入口

分配内存的入口是src/runtime/malloc.go中的mallocgc
```
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	...
	var span *mspan
	var x unsafe.Pointer
	noscan := typ == nil || typ.ptrdata == 0
	...
	if size <= maxSmallSize { //  小于等于32K
		if noscan && size < maxTinySize { // 不包含指针并且小于16字节
			// 微小对象分配
		} else {
			// 小对象分配
		}
	} else {
		// 大于32K的大对象分配
	}
	...
}
```
大对象分配:
分配后也会放到span中，调用链为mcache.allocLarge()->mheap.alloc->mheap.allocspan()，mheap.allocspan函数首先判断分配pages个数，如果较少，会首先在每个P自己的cache进行page分配，否则通过页分配器进行页面分配

小对象分配:
会从mcache自己管理的span中进行分配，首先调用nextFreeFast，如果不行则继续调用nextFree。两个函数解释如下
* nextFreeFast ：通过一个64bit大小的alloCache来快速计算，看是否有空闲位置，详细代码及注释如下：

```
func nextFreeFast(s *mspan) gclinkptr {
	theBit := sys.Ctz64(s.allocCache) // allocCache是allocBits的反码，因此1表示未使用。此函数计算allocCache结尾处0的个数，如果都是0，则返回64
	if theBit < 64 { // allocCache为一个uint64类型，小于64说明找到了一个空闲位置
		result := s.freeindex + uintptr(theBit) // 计算空闲位置在span中的索引
		if result < s.nelems { // 必须小于总大小
			freeidx := result + 1 // 计算span新的freeindex
			// 如果新的freeindex是64的整数倍，并且因此如果不是最后的数据，则说明allocCache需要根据allocBits重新计算(因为allocCache只能表示64位的大小)，此处直接返回0
			if freeidx%64 == 0 && freeidx != s.nelems {
				return 0
			}
			s.allocCache >>= uint(theBit + 1) // 将allocCache右移位，生成新的allocCache
			s.freeindex = freeidx // span的freeindex重新赋值
			s.allocCount++ // span的allocCount + 1
			return gclinkptr(result*s.elemsize + s.base()) // 返回新元素的地址
		}
	}
	return 0
}
```

* nextFree：通过freeindex查找可用元素位置，如果该span已经满，则放回到mcentral，然后从mcentral重新获取一个span，如果mcentral中没有空闲span，调用mheap获取，调用链路为mcache.refill->mcentral.cacheSpan->mheap.alloc，可以看到，如果没有从mcache和mcentral中获取到span，最终又回到了大对象分配的链路

微小对象分配:
微小对象为不含指针，并且大小小于16字节的内存空间分配
mcache中tiny，tinyoffset，tinyAllocs分别表示分配的地址，当前的偏移以及分配的元素个数。如果能分配则直接分配，否则调用nextFreeFast和nextFree从16字节大小的span中获取一个新的地址进行分配。

## 总结

本文从最上层的入口开始探究Go语言内存管理，最上层入口为mallocgc，涉及到的结构包括mspan、mcache、mcenter、mheap，分级管理，分类分配。未涉及到具体的页分配以及内存回收。内存回收涉及到了Go的GC流程，另文再述。






