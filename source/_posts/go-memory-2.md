---
title: Go语言内存管理-番外篇之算法分析
date: 2022-03-25
tags: Go
---
> 内存管理涉及到的算法，主要包括队列、各种位运算。文中使用Go版本为1.17.6

## 位运算

使用位移操作，实现各类算法

### 2的幂次
判断一个正整型数字，是否是2的幂次方，例如 1=>false，2=>true
```
func IsPowerOf2 (n uint) bool {
	return n&(n - 1）== 0 
}

```
2幂次方的数字只有最高位为1，其余位都为0，例如2=>10，4=>100，8=>1000。所以n&(n-1)等于0

### 对齐

内存管理中会要求某个地址按2的幂次方对齐，常见的例如按8字节对齐，假设地址为100，则按8字节对齐后为 8 * 13 = 104 ,地址为 112，则对齐之后也为112。
对齐就是将地址调整为 8 的整数倍

向上对齐
```
func alignUp(n, a uintptr) uintptr {
	return (n + a - 1) &^ (a - 1)
}
```

向下对齐
```
func alignDown(n, a uintptr) uintptr {
	return n &^ (a - 1)
}
```

向上，则n + a -1 ，向下，直接使用n，同理，计算ceil(n/a)，将n/a后向上取整

```
func divRoundUp(n, a uintptr) uintptr {
	return (n+a-1) / a
}
```


### 正整数二进制表示后末尾连续0个数

例如4=>100，则末尾连续0的个数为2，5=>101，末尾连续0的个数为0。我们看下移位版64位正整型末尾0的个数
```
func trailingZeros(n uint64) int{
  var mask uint64 = 1
  for i:=0;i<64;i++ {
	if n&mask != 0 {
	  return i
	}
	mask = mask << 1
  }
  return 64
}

```
Go使用http://supertech.csail.mit.edu/papers/debruijn.pdf 中的算法，代码为：
```
func Ctz64(x uint64) int {
	x &= -x                       // isolate low-order bit
	y := x * deBruijn64ctz >> 58  // extract part of deBruijn sequence
	i := int(deBruijnIdx64ctz[y]) // convert to bit index
	z := int((x - 1) >> 57 & 64)  // adjustment if zero
	return i + z
}
```
详细解释参考论文
Go中8位整型直接使用查表法获取，代码如下：
```
func Ctz8(x uint8) int {
	return int(ntz8tab[x])
}
var ntz8tab = [256]uint8{
	0x08, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x05, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x06, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x05, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x07, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x05, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x06, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x05, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
	0x04, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00, 0x03, 0x00, 0x01, 0x00, 0x02, 0x00, 0x01, 0x00,
}
```
8位只有256个元素，直接查表获取，例如0=>0x08，1=>0x00，2=>0x01


## 无锁队列

Go内存管理mcenter结构体保存了两个span队列，一个是有空闲元素的，一个没有。span队列使用无锁队列实现，其结构体如下：
```
type spanSet struct {
	spineLock mutex           // spine可以理解为二维数组，每一个数组有512个元素，每个元素是一个mspan。增加一个数组时，需要加锁
	spine     unsafe.Pointer //  spine初始地址
	spineLen  uintptr        //  spine当前长度
	spineCap  uintptr        //  spine容量
	index headTailIndex      //  索引，由头尾两个索引组成一个64位的整型，各占32个比特
}
```
一个mspan至少代表一个page，大小为8KB，则一个数组占用 512 * 8 KB = 2^9 * 2^13 B = 2^22 B = 4MB。spine初始容量为256，则一个spine可容纳 256 * 4MB = 2^8 * 2^22 B = 1GB。示意图如下：
![spine](/img/go-memory-spanset.png)

spine类似一个切片，大小不够时，会进行2倍的扩容，每次写入数据时通过尾部压入，弹出时通过头部弹出，除了扩充spine容量之外，其他时候不需要加锁，通过原子操作进行。代码如下：

```
func (b *spanSet) push(s *mspan) {
	cursor := uintptr(b.index.incTail().tail() - 1) // 通过尾部插入
	top, bottom := cursor/spanSetBlockEntries, cursor%spanSetBlockEntries // top和bottom分别代表第一维的索引和第二维的索引

	spineLen := atomic.Loaduintptr(&b.spineLen)
	var block *spanSetBlock
retry:
	if top < spineLen {   // 如果仍然有空间可以插入mspan，则直接通过原子操作取出地址
		spine := atomic.Loadp(unsafe.Pointer(&b.spine))
		blockp := add(spine, sys.PtrSize*top)
		block = (*spanSetBlock)(atomic.Loadp(blockp))
	} else {
		// 否则增加一个spine数组
	}

	// 通过原子操作将mspan存入
	atomic.StorepNoWB(unsafe.Pointer(&block.spans[bottom]), unsafe.Pointer(s))
}
```

```
func (b *spanSet) pop() *mspan {
	var head, tail uint32
claimLoop:
	for { // 注意此处是一个循环，需要考虑两个goroutine同时获取一个mspan或者一个在push一个在pop，通过原子操作，成功获取后才会退出
		headtail := b.index.load()
		head, tail = headtail.split()
		...
		spineLen := atomic.Loaduintptr(&b.spineLen)
		want := head
		for want == head {
			if b.index.cas(headtail, makeHeadTailIndex(want+1, tail)) {
				break claimLoop
			}
			headtail = b.index.load()
			head, tail = headtail.split()
		}
	}
	top, bottom := head/spanSetBlockEntries, head%spanSetBlockEntries // top和bottom分别代表第一维的索引和第二维的索引


	spine := atomic.Loadp(unsafe.Pointer(&b.spine))
	blockp := add(spine, sys.PtrSize*uintptr(top))
	block := (*spanSetBlock)(atomic.Loadp(blockp))
	s := (*mspan)(atomic.Loadp(unsafe.Pointer(&block.spans[bottom])))
	for s == nil {
		s = (*mspan)(atomic.Loadp(unsafe.Pointer(&block.spans[bottom])))
	}

	atomic.StorepNoWB(unsafe.Pointer(&block.spans[bottom]), nil)

	if atomic.Xadd(&block.popped, 1) == spanSetBlockEntries {
		// 清理回收
	}
	return s
}
```

## 总结

Go语言代码需要极致考虑性能，因此其可读性并不高。并且语言功能和实现方法还在不断的更新变动之中，希望文章能给大家了解Go实现有所帮助。