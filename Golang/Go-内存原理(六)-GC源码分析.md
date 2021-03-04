Go内存原理(六)-GC源码分析

------

本文介绍GC的源码分析(源码基于v1.14)

一、GC概述

GC会扫描哪些地方存有指针，首先变量要么分配到栈中，要么分配在在堆中。我们在之前的Go语言内存管理章节中学习到了堆对应的bitmap每2bit会指出arena哪些地址存储了对象，对象是否包含指针；还有我们的mcentral中，也会分为包含指针的span(noscan)，不包含指针的span，这样的分类会减少标记的时间，提升标记的效率。对于栈来说，栈空间的指针信息都存储在函数中，使用1 bit表示一个指针大小的内存 (位于stackmap.bytedata)

Go的GC是并行GC, 也就是GC的大部分处理和普通的go代码是同时运行的, 这让GO的GC流程比较复杂。
首先GC有四个阶段, 它们分别是

![1613808661560](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1613808661560.png)

- Sweep Termination: 对未清扫的span进行清扫, 只有上一轮的GC的清扫工作完成才可以开始新一轮的GC
- Mark: 扫描所有根对象, 和根对象可以到达的所有对象, 标记它们不被回收
- Mark Termination: 完成标记工作, 重新扫描部分根对象(要求STW)
- Sweep: 按标记结果清扫span

GC在满足一定条件后会被触发, 触发条件有以下几种:

- gcTriggerHeap: 当前分配的内存达到一定值就触发GC（自动）
- gcTriggerTime: 当一定时间没有执行过GC就触发GC（自动）
- gcTriggerCycle: 要求启动新一轮的GC, 已启动则跳过, 手动触发GC的`runtime.GC()`会使用这个条件（主动）

```go
type gcTriggerKind int
const (
	// gcTriggerHeap indicates that a cycle should be started when
	// the heap size reaches the trigger heap size computed by the
	// controller.
	gcTriggerHeap gcTriggerKind = iota

	// gcTriggerTime indicates that a cycle should be started when
	// it's been more than forcegcperiod nanoseconds since the
	// previous GC cycle.
    // 暂时设置是2分钟
	gcTriggerTime

	// gcTriggerCycle indicates that a cycle should be started if
	// we have not yet started cycle number gcTrigger.n (relative
	// to work.cycles).
	gcTriggerCycle
)
```

二、源码分析

参考这位大神的，本人感觉应该是全网最全的，大家完全可以对照源码进行学习~~[GC的实现原理](https://www.cnblogs.com/zkweb/p/7880099.html)

在此基础上我摘出几点并做补充：

1、STW都做了什么

​		目前整个GC流程会进行两次STW(Stop The World), 第一次是Mark阶段的开始, 第二次是Mark Termination阶段.
​		第一次STW会准备根对象的扫描, 启动写屏障(Write Barrier)和辅助GC(mutator assist).
​		第二次STW会重新扫描部分根对象, 禁用写屏障(Write Barrier)和辅助GC(mutator assist).
​		需要注意的是, 不是所有根对象的扫描都需要STW, 例如扫描栈上的对象只需要停止拥有该栈的G.
​		从go 1.9开始, 写屏障的实现使用了混合写屏障(Hybrid Write Barrier), 大幅减少了第二次STW的时间.

```go
// 由g0执行
func stopTheWorldWithSema() {
	_g_ := getg()

	// If we hold a lock, then we won't be able to stop another M
	// that is blocked trying to acquire the lock.
	if _g_.m.locks > 0 {
		throw("stopTheWorld: holding locks")
	}

	lock(&sched.lock)
    // 需要停止的P数量
	sched.stopwait = gomaxprocs
    // 设置gc等待标记, 调度时看见此标记会进入等待
	atomic.Store(&sched.gcwaiting, 1)
	preemptall()
	// stop current P
	_g_.m.p.ptr().status = _Pgcstop 
    //  减少需要停止的P数量(当前的P算一个)
	sched.stopwait--
	// 抢占所有在Psyscall状态的P, 防止它们重新参与调度
	for _, p := range allp {
		s := p.status
		if s == _Psyscall && atomic.Cas(&p.status, s, _Pgcstop) {
			if trace.enabled {
				traceGoSysBlock(p)
				traceProcStop(p)
			}
			p.syscalltick++
			sched.stopwait--
		}
	}
	// 防止所有空闲的P重新参与调度
	for {
		p := pidleget()
		if p == nil {
			break
		}
		p.status = _Pgcstop
		sched.stopwait--
	}
	wait := sched.stopwait > 0
	unlock(&sched.lock)

	// 如果仍有需要停止的P, 则等待它们停止
	if wait {
		for {
			// 循环等待 + 抢占所有运行中的G
			if notetsleep(&sched.stopnote, 100*1000) {
				noteclear(&sched.stopnote)
				break
			}
			preemptall()
		}
	}

	// 逻辑正确性检查
	bad := ""
	if sched.stopwait != 0 {
		bad = "stopTheWorld: not stopped (stopwait != 0)"
	} else {
		for _, p := range allp {
			if p.status != _Pgcstop {
				bad = "stopTheWorld: not stopped (status != _Pgcstop)"
			}
		}
	}
	if atomic.Load(&freezing) != 0 {
		// Some other thread is panicking. This can cause the
		// sanity checks above to fail if the panic happens in
		// the signal handler on a stopped thread. Either way,
		// we should halt this thread.
		lock(&deadlock)
		lock(&deadlock)
	}
	if bad != "" {
		throw(bad)
	}
    // 到这里所有运行中的G都会变为待运行, 并且所有的P都不能被M获取
	// 也就是说所有的go代码(除了当前的)都会停止运行, 并且不能运行新的go代码
}
```

2、辅助GC的作用

辅助GC(mutator assist)：为了防止heap增速太快, 在GC执行的过程中如果同时运行的G分配了内存, 那么这个G会被要求辅助GC做一部分的工作.
在GC的过程中同时运行的G称为"mutator", "mutator assist"机制就是G辅助GC做一部分工作的机制
辅助GC做的工作有两种类型, 一种是标记(Mark), 另一种是清扫(Sweep).

3、根对象

在GC的标记阶段首先需要标记的就是"根对象", 从根对象开始可到达的所有对象都会被认为是存活的.
根对象包含了全局变量, 各个G的栈上的变量等, GC会先扫描根对象然后再扫描根对象可到达的所有对象.

```go
func gcMarkRootPrepare() {
	work.nFlushCacheRoots = 0

	// Compute how many data and BSS root blocks there are.
	nBlocks := func(bytes uintptr) int {
		return int((bytes + rootBlockBytes - 1) / rootBlockBytes)
	}

	work.nDataRoots = 0
	work.nBSSRoots = 0

	// Scan globals.
	for _, datap := range activeModules() {
        //扫描可读写的全局变量
		nDataRoots := nBlocks(datap.edata - datap.data)
		if nDataRoots > work.nDataRoots {
			work.nDataRoots = nDataRoots
		}
	}

	for _, datap := range activeModules() {
        //扫描只读的全局变量
		nBSSRoots := nBlocks(datap.ebss - datap.bss)
		if nBSSRoots > work.nBSSRoots {
			work.nBSSRoots = nBSSRoots
		}
	}

	// We're only interested in scanning the in-use spans,
	// which will all be swept at this point. More spans
	// may be added to this list during concurrent GC, but
	// we only care about spans that were allocated before
	// this mark phase.
    //扫描各个span中特殊对象
	work.nSpanRoots = mheap_.sweepSpans[mheap_.sweepgen/2%2].numBlocks()

	// Scan stacks.
	//
	// Gs may be created after this point, but it's okay that we
	// ignore them because they begin life without any roots, so
	// there's nothing to scan, and any roots they create during
	// the concurrent phase will be scanned during mark
	// termination.
    // 扫描各个G的栈
	work.nStackRoots = int(atomic.Loaduintptr(&allglen))

	work.markrootNext = 0
	work.markrootJobs = uint32(fixedRootCount + work.nFlushCacheRoots + work.nDataRoots + work.nBSSRoots + work.nSpanRoots + work.nStackRoots)
}
```

3、三色标记法在代码中怎么体现的

我们了解到，对于v1.9以后GC开始使用三色标记+混合屏障

在Go内部对象并没有保存颜色的属性, 三色只是对它们的状态的描述,
	白色的对象在它所在的span的gcmarkBits中对应的bit为0,
	灰色的对象在它所在的span的gcmarkBits中对应的bit为1, 并且对象在标记队列中,
	黑色的对象在它所在的span的gcmarkBits中对应的bit为1, 并且对象已经从标记队列中取出并处理.
gc完成后, gcmarkBits会移动到allocBits然后重新分配一个全部为0的bitmap, 这样黑色的对象就变为了白色.

4、清除对象是干了什么

```go
// 清扫单个span
func (s *mspan) sweep(preserve bool) bool {
	_g_ := getg()
	if _g_.m.locks == 0 && _g_.m.mallocing == 0 && _g_ != _g_.m.g0 {
		throw("MSpan_Sweep: m is not locked")
	}
	sweepgen := mheap_.sweepgen
	if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
		print("MSpan_Sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
		throw("MSpan_Sweep: bad span state")
	}

	if trace.enabled {
		traceGCSweepSpan(s.npages * _PageSize)
	}

	// 统计已清理的页数
	atomic.Xadd64(&mheap_.pagesSwept, int64(s.npages))

	spc := s.spanclass
	size := s.elemsize
	res := false

	c := _g_.m.mcache
	freeToHeap := false

	// 判断在special中的析构器, 如果对应的对象已经不再存活则标记对象存活防止回收
	specialp := &s.specials
	special := *specialp
	for special != nil {

		objIndex := uintptr(special.offset) / size
		p := s.base() + objIndex*size
		mbits := s.markBitsForIndex(objIndex)
		if !mbits.isMarked() {
			// This object is not marked and has at least one special record.
			// Pass 1: see if it has at least one finalizer.
			hasFin := false
			endOffset := p - s.base() + size
			for tmp := special; tmp != nil && uintptr(tmp.offset) < endOffset; tmp = tmp.next {
				if tmp.kind == _KindSpecialFinalizer {
					// Stop freeing of object if it has a finalizer.
					mbits.setMarkedNonAtomic()
					hasFin = true
					break
				}
			}
			// Pass 2: queue all finalizers _or_ handle profile record.
			for special != nil && uintptr(special.offset) < endOffset {
				// Find the exact byte for which the special was setup
				// (as opposed to object beginning).
				p := s.base() + uintptr(special.offset)
				if special.kind == _KindSpecialFinalizer || !hasFin {
					// Splice out special record.
					y := special
					special = special.next
					*specialp = special
					freespecial(y, unsafe.Pointer(p), size)
				} else {
					// This is profile record, but the object has finalizers (so kept alive).
					// Keep special record.
					specialp = &special.next
					special = *specialp
				}
			}
		} else {
			// object is still live: keep special record
			specialp = &special.next
			special = *specialp
		}
	}

	// 除错用
	if debug.allocfreetrace != 0 || raceenabled || msanenabled {
		// Find all newly freed objects. This doesn't have to
		// efficient; allocfreetrace has massive overhead.
		mbits := s.markBitsForBase()
		abits := s.allocBitsForIndex(0)
		for i := uintptr(0); i < s.nelems; i++ {
			if !mbits.isMarked() && (abits.index < s.freeindex || abits.isMarked()) {
				x := s.base() + i*s.elemsize
				if debug.allocfreetrace != 0 {
					tracefree(unsafe.Pointer(x), size)
				}
				if raceenabled {
					racefree(unsafe.Pointer(x), size)
				}
				if msanenabled {
					msanfree(unsafe.Pointer(x), size)
				}
			}
			mbits.advance()
			abits.advance()
		}
	}

	// 计算释放的对象数量
	nalloc := uint16(s.countAlloc())
	if spc.sizeclass() == 0 && nalloc == 0 {
		// 如果span的类型是0(大对象)并且其中的对象已经不存活则释放到heap
		s.needzero = 1
		freeToHeap = true
	}
	nfreed := s.allocCount - nalloc
	if nalloc > s.allocCount {
		print("runtime: nelems=", s.nelems, " nalloc=", nalloc, " previous allocCount=", s.allocCount, " nfreed=", nfreed, "\n")
		throw("sweep increased allocation count")
	}

	// 设置新的allocCount
	s.allocCount = nalloc

	// 判断span是否无未分配的对象
	wasempty := s.nextFreeIndex() == s.nelems

	// 重置freeindex, 下次分配从0开始搜索
	s.freeindex = 0 // reset allocation index to start of span.
	if trace.enabled {
		getg().m.p.ptr().traceReclaimed += uintptr(nfreed) * s.elemsize
	}

	// gcmarkBits变为新的allocBits
	// 然后重新分配一块全部为0的gcmarkBits
	// 下次分配对象时可以根据allocBits得知哪些元素是未分配的
	// gcmarkBits becomes the allocBits.
	// get a fresh cleared gcmarkBits in preparation for next GC
	s.allocBits = s.gcmarkBits
	s.gcmarkBits = newMarkBits(s.nelems)

	// 更新freeindex开始的allocCache
	// Initialize alloc bits cache.
	s.refillAllocCache(0)

	// 如果span中已经无存活的对象则更新sweepgen到最新
	// 下面会把span加到mcentral或者mheap
	// We need to set s.sweepgen = h.sweepgen only when all blocks are swept,
	// because of the potential for a concurrent free/SetFinalizer.
	// But we need to set it before we make the span available for allocation
	// (return it to heap or mcentral), because allocation code assumes that a
	// span is already swept if available for allocation.
	if freeToHeap || nfreed == 0 {
		// The span must be in our exclusive ownership until we update sweepgen,
		// check for potential races.
		if s.state != mSpanInUse || s.sweepgen != sweepgen-1 {
			print("MSpan_Sweep: state=", s.state, " sweepgen=", s.sweepgen, " mheap.sweepgen=", sweepgen, "\n")
			throw("MSpan_Sweep: bad span state after sweep")
		}
		// Serialization point.
		// At this point the mark bits are cleared and allocation ready
		// to go so release the span.
		atomic.Store(&s.sweepgen, sweepgen)
	}

	if nfreed > 0 && spc.sizeclass() != 0 {
		// 把span加到mcentral, res等于是否添加成功
		c.local_nsmallfree[spc.sizeclass()] += uintptr(nfreed)
		res = mheap_.central[spc].mcentral.freeSpan(s, preserve, wasempty)
		// freeSpan会更新sweepgen
		// MCentral_FreeSpan updates sweepgen
	} else if freeToHeap {
		// 把span释放到mheap
		// Free large span to heap

		if debug.efence > 0 {
			s.limit = 0 // prevent mlookup from finding this span
			sysFault(unsafe.Pointer(s.base()), size)
		} else {
			mheap_.freeSpan(s, 1)
		}
		c.local_nlargefree++
		c.local_largefree += size
		res = true
	}
	
	// 如果span未加到mcentral或者未释放到mheap, 则表示span仍在使用
	if !res {
		// 把仍在使用的span加到sweepSpans的"已清扫"队列中
		mheap_.sweepSpans[sweepgen/2%2].push(s)
	}
	return res
}go
```



引用：

[Golang垃圾回收剖析](https://www.cnblogs.com/zkweb/p/7880099.html)

[GC的实现原理](https://www.cnblogs.com/zkweb/p/7880099.html)