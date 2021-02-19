浅析Go内存源码

学习了上2篇Go语言的内存管理的原理（一、TCMalloc原理https://juejin.cn/post/6919005994966056968  二、Go内存管理原理https://juejin.cn/post/6920485540391288839）我们理解源码的流程就非常easy啦。

首先来看一下，mcache，mcentral，mheap这三个结构体。我们选则最长使用的字段进行分析。注：这里都是基于go1.14源码分析（对照代码看体验更佳

```go
// mcache
type mcache struct {
	tiny             uintptr  // 分配器起始地址
	tinyoffset       uintptr //下一个空闲地偏置
	local_tinyallocs uintptr //已经分配的对象个数 
	alloc [numSpanClasses]*mspan // 申请的134个span
	// 栈链表
	stackcache [_NumStackOrders]stackfreelist

	// Local allocator stats, flushed during GC.
	local_largefree  uintptr  
	local_nlargefree uintptr      
	local_nsmallfree [_NumSizeClasses]uintptr
	flushGen uint32
}
// mcentral
type mcentral struct {
	lock      mutex  // 互斥锁
	spanclass spanClass // 67*2
	nonempty  mSpanList // 含有空闲对象链表
	empty     mSpanList // 不含空闲对象链表
	nmalloc uint64 // 已分配对象个数
}

// mheap
type mheap struct {
	lock      mutex   // 互斥锁
	pages     pageAlloc // page allocation data structure
	sweepgen  uint32    // GC相关
	sweepdone uint32    // GC相关
	sweepers  uint32    // GC相关

	allspans []*mspan // 所有申请的span

	sweepSpans [2]gcSweepBuf // GC相关
	pagesInUse         uint64  // pages of spans in stats mSpanInUse; updated atomically

	arenas [1 << arenaL1Bits]*[1 << arenaL2Bits]*heapArena

	// arenaHints is a list of addresses at which to attempt to
	// add more heap arenas. This is initially populated with a
	// set of general hint addresses, and grown with the bounds of
	// actual heap arena ranges.
	arenaHints *arenaHint

	// arena is a pre-reserved space for allocating heap arenas
	// (the actual arenas). This is only used on 32-bit.
	arena linearAlloc

	// allArenas is the arenaIndex of every mapped arena. This can
	// be used to iterate through the address space.
	//
	// Access is protected by mheap_.lock. However, since this is
	// append-only and old backing arrays are never freed, it is
	// safe to acquire mheap_.lock, copy the slice header, and
	// then release mheap_.lock.
	allArenas []arenaIdx

	// sweepArenas is a snapshot of allArenas taken at the
	// beginning of the sweep cycle. This can be read safely by
	// simply blocking GC (by disabling preemption).
	sweepArenas []arenaIdx  // GC相关

	// curArena is the arena that the heap is currently growing
	// into. This should always be physPageSize-aligned.
	curArena struct {
		base, end uintptr
	}

	// mcentral 内存分配中心，mcache没有足够的内存分配的时候，会从mcentral分配
    // mcentral 是mheap在管理
	central [numSpanClasses]struct {
		mcentral mcentral
		pad      [cpu.CacheLinePadSize - unsafe.Sizeof(mcentral{})%cpu.CacheLinePadSize]byte
	}
}
```

接下来我们按照Tiny对象，小对象，大对象分类来介绍内存分配的流程。

0、对象的内存分配都在newobject()接口，这个接口包括内存申请和回收。本文主要学习内存申请源码

```go
func newobject(typ *_type) unsafe.Pointer {
	return mallocgc(typ.size, typ, true)
}

func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	// 参数检查,是否包含指针等
    ...
    // size <= 32KB
	if size <= maxSmallSize {
        // tiny对象的内存分配 < 16B
        if noscan && size < maxTinySize {
            // tiny对象的内存分配
		} else {
			// 小对象分配
	} else {
		// 大对象分配
	}
	...
    // GC回收
	return x
}
```

1、Tiny对象内存申请(<16B)

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
    ....
    // 小于16B的时候Tiny对象申请
	if noscan && size < maxTinySize {
		// Tiny allocator.
		off := c.tinyoffset
		// 对齐 调整偏移量
		if size&7 == 0 { // 7的倍数的大小调整成8的
			off = alignUp(off, 8)
		} else if size&3 == 0 {
			off = alignUp(off, 4)
		} else if size&1 == 0 {
			off = alignUp(off, 2)
		}
		// 若偏移量+size小于16B 则进行小对象分配
		if off+size <= maxTinySize && c.tiny != 0 {
			// The object fits into existing tiny block.
			x = unsafe.Pointer(c.tiny + off)
			c.tinyoffset = off + size
			c.local_tinyallocs++ // 申请的个数+1
			mp.mallocing = 0
			releasem(mp)
			return x
		}
		// 当前tiny 块内存空间不足，向mache申请一个span
		// tinySpanClass = 5 (0,0,1,1,2 我们介绍过0,0不使用，1，1是8B此时不够用，这里使用下一级别的也就是5->16B)
		span := c.alloc[tinySpanClass]
		// 尝试从当前的span 获取内存，获取不到返回0,也就是我需要大小为16B的span
		v := nextFreeFast(span)
		if v == 0 {
			// 没有从 allocCache 获取到内存，netxtFree函数
			// 尝试从 mcentral获取一个新的对应规格的span内存
			// 替换原先内存空间不足的内存块，并分配内存
			v, _, shouldhelpgc = c.nextFree(tinySpanClass)  // 下面会讲解
		}
		x = unsafe.Pointer(v)
		(*[2]uint64)(x)[0] = 0
		(*[2]uint64)(x)[1] = 0
		// See if we need to replace the existing tiny block with the new one
		// based on amount of remaining free space.
		if size < c.tinyoffset || c.tiny == 0 {
			c.tiny = uintptr(x)
			c.tinyoffset = size
		}
		size = maxTinySize
	}
    ....
}
```

若本地的tiny内存块不足，则向mcache继续进行申请

```go
// 在mache申请空闲内存
func (c *mcache) nextFree(spc spanClass) (v gclinkptr, s *mspan, shouldhelpgc bool) {
	s = c.alloc[spc]
	shouldhelpgc = false
	freeIndex := s.nextFreeIndex()
	if freeIndex == s.nelems {
		// The span is full.x
		if uintptr(s.allocCount) != s.nelems {
			println("runtime: s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
			throw("s.allocCount != s.nelems && freeIndex == s.nelems")
		}
		c.refill(spc)  // 这个地方 mcache 向 mcentral 申请
		shouldhelpgc = true
		s = c.alloc[spc]
		// mcache 向 mcentral 申请完之后，再次从 mcache 申请
		freeIndex = s.nextFreeIndex()
	}

	if freeIndex >= s.nelems {
		throw("freeIndex is not valid")
	}

	v = gclinkptr(freeIndex*s.elemsize + s.base())
	s.allocCount++
	if uintptr(s.allocCount) > s.nelems {
		println("s.allocCount=", s.allocCount, "s.nelems=", s.nelems)
		throw("s.allocCount > s.nelems")
	}
	return
}
```

总结一下，Tiny对象的申请(小于16B)若当前Tiny分配器本地有足够的内存，则在本地申请；否则向mcache申请sizeclass=5的sapn，若mcache没有对应级别的span，则向mcentral申请。

2、小内存申请（< 32KB）

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
	{	// 小对象的内存分配  <32KB
		var sizeclass uint8
		// 这里将所有级别分为2类
		// 1) size <= 1024-8
		if size <= smallSizeMax-8 {
			// eg: size =700 (size+smallSizeDiv-1)/smallSizeDiv = (700+8-1/8)= 88  sizeclass=28 =>704
			// 获取size对应的sizeclass
			sizeclass = size_to_class8[(size+smallSizeDiv-1)/smallSizeDiv]
		} else { // 2) size >1024-8
			// 获取size对应的sizeclass
			sizeclass = size_to_class128[(size-smallSizeMax+largeSizeDiv-1)/largeSizeDiv]
		}
		// sizeclass对应的size，也就是span的大小
		size = uintptr(class_to_size[sizeclass])
		spc := makeSpanClass(sizeclass, noscan)
		// 找到对应的span
		span := c.alloc[spc]
		//尝试从 allocCache 获取内存，获取不到返回0
		v := nextFreeFast(span) // 下面会讲解
		if v == 0 {
			v, span, shouldhelpgc = c.nextFree(spc)
		}
		x = unsafe.Pointer(v)
		if needzero && span.needzero != 0 {
			memclrNoHeapPointers(unsafe.Pointer(v), size)
		}
	}
}
```

若cache没有对应的span，使用nextFreeFast(span)接口向central申请，该接口已经在Tiny对象内存申请介绍过。

在nextFreeFast(span)接口中主要使用refill接口

```go
func (c *mcache) refill(spc spanClass) {
	// Return the current cached span to the central lists.
	s := c.alloc[spc]

	if uintptr(s.allocCount) != s.nelems {
		throw("refill of span with free space remaining")
	}
	if s != &emptymspan {
		// Mark this span as no longer cached.
		if s.sweepgen != mheap_.sweepgen+3 {
			throw("bad sweepgen in refill")
		}
		atomic.Store(&s.sweepgen, mheap_.sweepgen)
	}

	// 向central进行申请
	s = mheap_.central[spc].mcentral.cacheSpan() // 下面会讲解
	if s == nil {
		throw("out of memory")
	}

	if uintptr(s.allocCount) == s.nelems {
		throw("span has no free space")
	}

	// Indicate that this span is cached and prevent asynchronous
	// sweeping in the next sweep phase.
	s.sweepgen = mheap_.sweepgen + 3

	c.alloc[spc] = s
}
```

cacheSpan是向mcentral申请span的核心接口

```go
func (c *mcentral) cacheSpan() *mspan {
	...
	// mcentral是需加锁的
	lock(&c.lock)
	traceDone := false
	if trace.enabled {
		traceGCSweepStart()
	}
	sg := mheap_.sweepgen
retry:
	var s *mspan
    // 从nonempty链表查找空闲span
	for s = c.nonempty.first; s != nil; s = s.next {
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) { // 符合span
			c.nonempty.remove(s) // 在nonempty链表删除该span
			c.empty.insertBack(s) // 在empty链表增加该span
			unlock(&c.lock)
			s.sweep(true)
			goto havespan
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// we have a nonempty span that does not require sweeping, allocate from it
		c.nonempty.remove(s)
		c.empty.insertBack(s)
		unlock(&c.lock)
		goto havespan
	}
	// 在empty链表查找是否有可用的span,是因为某些span可能被垃圾回收标记为空闲但是还没清理
	for s = c.empty.first; s != nil; s = s.next {
		if s.sweepgen == sg-2 && atomic.Cas(&s.sweepgen, sg-2, sg-1) {
			// we have an empty span that requires sweeping,
			// sweep it and see if we can free some space in it
			c.empty.remove(s)
			// swept spans are at the end of the list
			c.empty.insertBack(s)
			unlock(&c.lock)
			s.sweep(true)
			freeIndex := s.nextFreeIndex()
			if freeIndex != s.nelems {
				s.freeindex = freeIndex
				goto havespan
			}
			lock(&c.lock)
			// the span is still empty after sweep
			// it is already in the empty list, so just retry
			goto retry
		}
		if s.sweepgen == sg-1 {
			// the span is being swept by background sweeper, skip
			continue
		}
		// already swept empty span,
		// all subsequent ones must also be either swept or in process of sweeping
		break
	}
	if trace.enabled {
		traceGCSweepDone()
		traceDone = true
	}
	unlock(&c.lock)
    ...
}
```

小对象是在mache申请适合自己大小的span，若mache没有可用的span，mache会向mcentral申请，加锁，找一个可用的span，从nonempty删除该span，然后放到empty链表中，将span返回给工作线程，解锁；若没有足够的内存，mcentral还会继续向mheap继续申请。

3、大对象内存申请

大对象直接在mheap进行申请

```go
func mallocgc(size uintptr, typ *_type, needzero bool) unsafe.Pointer {
		var s *mspan
		shouldhelpgc = true
		systemstack(func() {
			// 直接从mheap申请
			s = largeAlloc(size, needzero, noscan) // 下面会讲解
		})
		s.freeindex = 1
		s.allocCount = 1
		x = unsafe.Pointer(s.base())
		size = s.elemsize
}
```

使用classsize=0进行申请

```go
// 大对象申请
func largeAlloc(size uintptr, needzero bool, noscan bool) *mspan {
	// 内存溢出判断
	if size+_PageSize < size {
		throw("out of memory")
	}
	// 计算出对象所需的页数 对于大于32K的大内存分配都是整数页
	npages := size >> _PageShift
	if size&_PageMask != 0 {
		npages++
	}

	deductSweepCredit(npages*_PageSize, npages)
	// 分配函数的具体实现，使用span->sizeclass=0
	s := mheap_.alloc(npages, makeSpanClass(0, noscan), needzero)  // 下面会讲解
	if s == nil {
		throw("out of memory")
	}

	s.limit = s.base() + size
	// bitmap 记录分配的span
	heapBitsForAddr(s.base()).initSpan(s)
	return s
}
```

向mheap申请一个span

```go
func (h *mheap) alloc(npages uintptr, spanclass spanClass, needzero bool) *mspan {
	var s *mspan
	systemstack(func() {
		// To prevent excessive heap growth, before allocating n pages
		// we need to sweep and reclaim at least n pages.
		if h.sweepdone == 0 {
			// 为了阻止内存的大量占用和堆的增长，我们在分配对应页数的内存前需要先调用 runtime.mheap.reclaim 方法回收一部分内存
			h.reclaim(npages)
		}
        //从mheap申请一个mspan
		s = h.allocSpan(npages, false, spanclass, &memstats.heap_inuse)  // 下面会讲解
	})

	if s != nil {
		if needzero && s.needzero != 0 {
			memclrNoHeapPointers(unsafe.Pointer(s.base()), s.npages<<_PageShift)
		}
		s.needzero = 0
	}
	return s
}
```

```go
func (h *mheap) allocSpan(npages uintptr, manual bool, spanclass spanClass, sysStat *uint64) (s *mspan) {
	// Function-global state.
	gp := getg() // 获取当前协程
	base, scav := uintptr(0), uintptr(0)  // 基址，回收地址(bit位)

	// If the allocation is small enough, try the page cache!
	pp := gp.m.p.ptr()
	// 当申请页数小于 8*64/4=128时从P的pageCache申请【pageCache后续会介绍】
    // 注意pageCache是连续的
	if pp != nil && npages < pageCachePages/4 {
		c := &pp.pcache

		// If the cache is empty, refill it.
		// p对应的pageCache为空，则申请cache内存 --- pageCache下面会讲解
		if c.empty() {
			lock(&h.lock)
            // 填充本地的pageCache
			*c = h.pages.allocToCache()
			unlock(&h.lock)
		}

		// Try to allocate from the cache.
		// 1、先从p的页缓存获取内存区域的基地址和大小
		base, scav = c.alloc(npages)  // 下面会讲解
		// 申请成功
		if base != 0 {
			// 对应span
			s = h.tryAllocMSpan()

			if s != nil && gcBlackenEnabled == 0 && (manual || spanclass.sizeclass() != 0) {
				goto HaveSpan
			}
		}
	}

	// For one reason or another, we couldn't get the
	// whole job done without the heap lock.
	lock(&h.lock)
	// 2、P的页缓存没有足够的内存，则在页堆上申请内存
	if base == 0 {
		// Try to acquire a base address.
		// mheap全局堆区
		base, scav = h.pages.alloc(npages)
		if base == 0 {
			if !h.grow(npages) { // 从系统申请固定页数大小的内存区域
				unlock(&h.lock)
				return nil
			}
			base, scav = h.pages.alloc(npages) //重新从mheap获取
			if base == 0 {
				throw("grew heap, but no adequate free space found")
			}
		}
	}
	....
}
```

从页缓存申请，只能申请连续个页，若不能连续则返回失败

```go
// 申请pageCache
func (c *pageCache) alloc(npages uintptr) (uintptr, uintptr) {
	if c.cache == 0 {
		return 0, 0
	}
	// 页数为1
	if npages == 1 {
		i := uintptr(sys.TrailingZeros64(c.cache))
		scav := (c.scav >> i) & 1
		c.cache &^= 1 << i // 把使用的页数对应的bit位置为0 set bit to mark in-use
		c.scav &^= 1 << i  // clear bit to mark unscavenged
		return c.base + i*pageSize, uintptr(scav) * pageSize
	}
	// 申请连续的n页
	return c.allocN(npages) // 下面会讲解
}
```

```go
func (c *pageCache) allocN(npages uintptr) (uintptr, uintptr) {
	i := findBitRange64(c.cache, uint(npages))
	if i >= 64 {
		return 0, 0
	}
	mask := ((uint64(1) << npages) - 1) << i
	scav := sys.OnesCount64(c.scav & mask)
	c.cache &^= mask // 把使用的页数对应的bit位置为0 mark in-use bits  例如1110&^10=1100
	c.scav &^= mask  // clear scavenged bits
	return c.base + uintptr(i*pageSize), uintptr(scav) * pageSize
}
```

这里插播一下pcache的原理

go1.14中使用bitmap来管理内存页,并在每个P中维护一份pageCache。pageCache是一个位图，1表示未分配，0表示已分配

```go
type pageCache struct {
	base  uintptr // 虚拟内存的基址 base address of the chunk
	cache uint64  // bit位标记内存是否被分配 
	scav  uint64  // bit位标记内存是否被回收 
}
```

![1611823157606](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1611823157606.png)

一位表示一页(8KB)，所以最大表示8KB*64=512KB缓存。当需要分配的页数小于 512/4=128KB时，需要首先从cache中分配

若页缓存没有足够的页，则向虚拟内存申请页

```go
// 在heap进行页申请
func (s *pageAlloc) alloc(npages uintptr) (addr uintptr, scav uintptr) {
	// If the searchAddr refers to a region which has a higher address than
	// any known chunk, then we know we're out of memory.
	if chunkIndex(s.searchAddr) >= s.end {
		return 0, 0
	}

	// If npages has a chance of fitting in the chunk where the searchAddr is,
	// search it directly.
	searchAddr := uintptr(0)
	// chunkPages总页数-当前使用页数>=需要申请的页数 说明chunk满足申请的页数
	if pallocChunkPages-chunkPageIndex(s.searchAddr) >= uint(npages) {
		// npages is guaranteed to be no greater than pallocChunkPages here.
		i := chunkIndex(s.searchAddr)
		if max := s.summary[len(s.summary)-1][i].max(); max >= uint(npages) {
			j, searchIdx := s.chunkOf(i).find(npages, chunkPageIndex(s.searchAddr))
			if j < 0 {
				print("runtime: max = ", max, ", npages = ", npages, "\n")
				print("runtime: searchIdx = ", chunkPageIndex(s.searchAddr), ", s.searchAddr = ", hex(s.searchAddr), "\n")
				throw("bad summary data")
			}
			addr = chunkBase(i) + uintptr(j)*pageSize
			searchAddr = chunkBase(i) + uintptr(searchIdx)*pageSize
			goto Found
		}
	}
	// We failed to use a searchAddr for one reason or another, so try
	// the slow path.
	addr, searchAddr = s.find(npages)
	if addr == 0 {
		if npages == 1 {
			// We failed to find a single free page, the smallest unit
			// of allocation. This means we know the heap is completely
			// exhausted. Otherwise, the heap still might have free
			// space in it, just not enough contiguous space to
			// accommodate npages.
			s.searchAddr = maxSearchAddr
		}
		return 0, 0
	}
	....
	return addr, scav
}
```

至此大对象的内存申请已经结束，对于大对象，使用mheap直接分配，若mheap没有足够的内存，则mheap向虚拟内存申请若干个pages。可以看到，约到后面申请内存的代价就越来越大

从源码分析，我们可以学习到

1、相比与之前的go版本使用树来管理内存块，go1.14使用bit位更为高效，管理也很方便，在源码会看到各种位操作也可以达到相同的效果，看的时候直呼：妙啊 （基础差的还得自己算一算
2、看到多处使用空间换时间的操作，都是很大情况下提供了效率

3、

下一篇，我们将共同学习一下GC是什么

**引用**

> [分配单元大小_Go内存管理三部曲]https://blog.csdn.net/weixin_39941732/article/details/111219285
> [golang内存管理]http://legendtkl.com/2017/04/02/golang-alloc/
> [深入理解Go-内存分配]https://studygolang.com/articles/22785



