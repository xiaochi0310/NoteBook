### GO关键字发生了什么

------

> 以下是基于 go1.14.2 进行分析

​		在了解源码之前，因为涉及到goroutine的调度，所以先了解一下go语言的GMP模型。M相当于线程，一个M对应一个P控制器,P控制器负责goroutine调度在M上。这里涉及2个goroutine队列，一个是P本地的队列runq，这个队列存储待运行的goroutines，存储方式是用循环队列进行存储,在代码中是用数组实现的。另外一个队列是全局的schedt，用链表实现的，这里个数没有进行限制。当本地队列为空时，会向全局队列进行调度。

以下时简单的GMP模型：

![1605597086800](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1605597086800.png)

#### 一、案例分析

下面是一个简单的协程实例，使用go关键字开启一个协程执行funTest()

```go
func funTest() {
	fmt.Printf("this is test")
}
func main() {
	go funTest() // 开启协程
}
```

我们通过汇编来看一下具体的执行流程：

```shell
go tool compile -S main.go
```

这里我们只看main函数的汇编流程：

![1605597660129](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1605597660129.png)

> SP：指向当前栈帧的栈顶
> BP：指向当前栈帧的栈底，函数栈的起始位置。这个寄存器占8个字节，跳过8字节后才是函数栈上局部变量的内存

从汇编看来，使用CALL关键字，go调用会跳转到newproc处理函数

#### 二、源码分析

经过GC编译，会把内存的大小以及go需要执行的函数指针传进来。

下图的fn即为需要执行的参数

```go
//go:nosplit
// 编译器标识，跳过栈溢出检测，跳过此检测，提高性能
func newproc(siz int32, fn *funcval) {
	argp := add(unsafe.Pointer(&fn), sys.PtrSize)
	fmt.Printf(siz)
	gp := getg() // 获取goroutine
	pc := getcallerpc() // 获取调用方的程序计数器
	systemstack(func() {
		newproc1(fn, argp, siz, gp, pc)
	})
}
```

newproc1实现了调用goroutine具体流程

```go
/*
	1.获取或者创建新的 Goroutine 结构体
	2.将传入的参数移到 Goroutine 的栈上
	3.更新 Goroutine 调度相关的属性
	4.将 Goroutine 加入处理器的运行队列
*/
func newproc1(fn *funcval, argp unsafe.Pointer, narg int32, callergp *g, callerpc uintptr) {
	_g_ := getg()

	if fn == nil {
		_g_.m.throwing = -1 // do not dump full stacks
		throw("go of nil func value")
	}
	acquirem() // disable preemption because it can be holding p in a local var
	siz := narg
	siz = (siz + 7) &^ 7

	// We could allocate a larger initial stack if necessary.
	// Not worth it: this is almost always an error.
	// 4*sizeof(uintreg): extra space added below
	// sizeof(uintreg): caller's LR (arm) or return address (x86, in gostartcall).
	if siz >= _StackMin-4*sys.RegSize-sys.RegSize {
		throw("newproc: function arguments too large for new goroutine")
	}

	_p_ := _g_.m.p.ptr()
	// 先从处理器M->P->gFree列表查找空闲的goroutine
	newg := gfget(_p_)
	if newg == nil { // 若调度器的gFree和处理器的gFree列表不存在结构体时
		newg = malg(_StackMin) // 创建2048大小栈的新结构体
		casgstatus(newg, _Gidle, _Gdead) // 设置状态：_Gidle只是初始化  _Gdead分配栈
		allgadd(newg) // publishes with a g->status of Gdead so GC scanner doesn't look at uninitialized stack.
	}
	if newg.stack.hi == 0 {
		throw("newproc1: newg missing stack")
	}

	if readgstatus(newg) != _Gdead {
		throw("newproc1: new g is not Gdead")
	}

	totalSize := 4*sys.RegSize + uintptr(siz) + sys.MinFrameSize // extra space in case of reads slightly beyond frame
	totalSize += -totalSize & (sys.SpAlign - 1)                  // align to spAlign
	sp := newg.stack.hi - totalSize // goroutine的栈指针

	spArg := sp
	if usesLR {
		// caller's LR
		*(*uintptr)(unsafe.Pointer(sp)) = 0
		prepGoExitFrame(sp)
		spArg += sys.MinFrameSize
	}
	if narg > 0 {
		// 将fn函数的全部参数拷贝到栈上
		// 将所有参数对应的内存空间整片的拷贝到栈上
		memmove(unsafe.Pointer(spArg), argp, uintptr(narg))
		...
		if writeBarrier.needed && !_g_.m.curg.gcscandone {
			f := findfunc(fn.fn)
			stkmap := (*stackmap)(funcdata(f, _FUNCDATA_ArgsPointerMaps))
			if stkmap.nbit > 0 {
				// We're in the prologue, so it's always stack map index 0.
				bv := stackmapdata(stkmap, 0)
				bulkBarrierBitmap(spArg, spArg, uintptr(bv.n)*sys.PtrSize, 0, bv.bytedata)
			}
		}
	}

	memclrNoHeapPointers(unsafe.Pointer(&newg.sched), unsafe.Sizeof(newg.sched))
	// 设置新的goroutine结构体的参数
	newg.sched.sp = sp // 栈指针
	newg.stktopsp = sp
	// 程序计数器
	newg.sched.pc = funcPC(goexit) + sys.PCQuantum // +PCQuantum so that previous instruction is in same function
	newg.sched.g = guintptr(unsafe.Pointer(newg))
	// 设置调度相关信息
	gostartcallfn(&newg.sched, fn)
	// 调度goroutine的调用者的pc
	newg.gopc = callerpc
	newg.ancestors = saveAncestors(callergp)
	newg.startpc = fn.fn
	if _g_.m.curg != nil {
		newg.labels = _g_.m.curg.labels
	}
	if isSystemGoroutine(newg, false) {
		atomic.Xadd(&sched.ngsys, +1)
	}
	// 更新状态:_Grunnable
	casgstatus(newg, _Gdead, _Grunnable)

	if _p_.goidcache == _p_.goidcacheend {
		// Sched.goidgen is the last allocated id,
		// this batch must be [sched.goidgen+1, sched.goidgen+GoidCacheBatch].
		// At startup sched.goidgen=0, so main goroutine receives goid=1.
		_p_.goidcache = atomic.Xadd64(&sched.goidgen, _GoidCacheBatch)
		_p_.goidcache -= _GoidCacheBatch - 1
		_p_.goidcacheend = _p_.goidcache + _GoidCacheBatch
	}
	newg.goid = int64(_p_.goidcache)
	_p_.goidcache++
	if raceenabled {
		newg.racectx = racegostart(callerpc)
	}
	if trace.enabled {
		traceGoCreate(newg, newg.startpc)
	}
	runqput(_p_, newg, true) // 把g加入处理器对应的p的队列中

	if atomic.Load(&sched.npidle) != 0 && atomic.Load(&sched.nmspinning) == 0 && mainStarted {
		wakep()  //唤醒新的处理器执行goroutine
	}
	releasem(_g_.m) // 把g与m解绑
}
```

在上面的函数中会获取一个空闲的goroutine，以下是具体实现：

```go
// goroutine所在处理器的调度器或者全局调度器sched.gFree 列表中获取 runtime.g 结构体；找一个空闲goroutine
func gfget(_p_ *p) *g {
retry:
	if _p_.gFree.empty() && (!sched.gFree.stack.empty() || !sched.gFree.noStack.empty()) {
		lock(&sched.gFree.lock)
		// Move a batch of free Gs to the P.
		// 若处理器的goroutine列表是空的，加入32个空闲goroutine在gFree
		for _p_.gFree.n < 32 {
			// Prefer Gs with stacks.
			// 从sched调取空闲的goroutine取出
			gp := sched.gFree.stack.pop()
			if gp == nil {
				gp = sched.gFree.noStack.pop()
				if gp == nil {
					break
				}
			}
			sched.gFree.n--
			_p_.gFree.push(gp)
			_p_.gFree.n++
		}
		unlock(&sched.gFree.lock)
		goto retry
	}
	// 当处理器的goroutine数量充足时，会从列表头部返回一个新的goroutine
	gp := _p_.gFree.pop()
	if gp == nil {
		return nil
	}
	_p_.gFree.n--
	if gp.stack.lo == 0 {
		// Stack was deallocated in gfput. Allocate a new one.
		systemstack(func() {
			gp.stack = stackalloc(_FixedStack)
		})
		gp.stackguard0 = gp.stack.lo + _StackGuard
	} else {
		if raceenabled {
			racemalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
		}
		if msanenabled {
			msanmalloc(unsafe.Pointer(gp.stack.lo), gp.stack.hi-gp.stack.lo)
		}
	}
	return gp
}
```

将获取的goroutine放在运行队列具体实现功能：

```go
// 可能是全局的运行队列，也可能是处理器的运行队列
func runqput(_p_ *p, gp *g, next bool) {
	if randomizeScheduler && next && fastrand()%2 == 0 {
		next = false
	}
    // 将goroutine设置到处理器的runnext作为下一个处理器执行的任务
	if next {
	retryNext:
		oldnext := _p_.runnext
		if !_p_.runnext.cas(oldnext, guintptr(unsafe.Pointer(gp))) {
			goto retryNext
		}
		if oldnext == 0 {
			return
		}
		// Kick the old runnext out to the regular run queue.
		gp = oldnext.ptr()
	}

retry:
	h := atomic.LoadAcq(&_p_.runqhead) // load-acquire, synchronize with consumers
	t := _p_.runqtail
	if t-h < uint32(len(_p_.runq)) {
		// 本地运行队列runq:环形列表,最多256个
		_p_.runq[t%uint32(len(_p_.runq))].set(gp)
		atomic.StoreRel(&_p_.runqtail, t+1) // store-release, makes the item available for consumption
		return
	}
	// 若本地队列已经没有空间，则把goroutine增加到全局的运行队列中 sched
	if runqputslow(_p_, gp, h, t) {
		return
	}
	// the queue is not full, now the put above must succeed
	goto retry
}

```

