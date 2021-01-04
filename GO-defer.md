#### 一、defer关键字的说明

defer是一种延迟函数，每增加一个defer函数会将defer函数压栈，所以在执行的时候是后压栈先执行

1、defer**触发的场景**主要有三

> A "defer" statement invokes a function whose execution is deferred to the moment the surrounding function returns, either because the surrounding function executed a return statement, reached the end of its function body, or because the corresponding goroutine is panicking

1）主函数return返回时，在return执行之前
2）主函数执行结束时
3）当前goroutine发生panic

2、官方对defer也提出**3条规则**

1）延迟函数的参数在defer语句出现时就已经确定下来了
这是因为在defer语句出现的时候，同时把需要的参数拷贝到栈上，后续主函数对该参数进行改变，不会影响defer中的值

```go
func testDerfer() {
	i := 5
	defer fmt.Println("defer",i)
	i++
	fmt.Println(i)
}
/*
	6
	defer 5
*/
```

但是指针的情况下，我们可以看下有什么不同：

```go
func testDerfer() {
	var i int
	defer test(&i)  // 拷贝的值地址
	i++
	fmt.Println(&i,i)
}

func test(a *int) {
	fmt.Println("defer",a,*a)
}
/*
0xc0000140e0 1
defer 0xc0000140e0 1
*/
```

2）延迟函数执行按后进先出顺序执行，即先出现的defer最后执行
3）延迟函数可以操作主函数的具名返回值

```go
func testDerfer() (result int) {
	i := 1
	defer func() {
		result++
	}()

	return i
}
// testDerfer返回2
```

#### 二、源码分析

可以先看一下_defer结构体字段，无特殊版本标记都是1.2版本出现的字段

```go
type _defer struct {
	siz     int32 // includes both arguments and results
	// 记录defer参与与返回值共占多少空间,这些空间会直接分配在defer结构体后面，用于在注册时保存参数，并在执行时拷贝到调用者参数与返回值空间
	started bool // 标记defer是否已经执行
	heap    bool // 1.13 是否为堆分配
	openDefer bool   // 1.14 表示当前 defer 是否经过开放编码的优化
	sp        uintptr  // 调用者栈指针，通过这个字段，函数可以判断自己注册的defer是否已经执行完成
	pc        uintptr  // deferporc 的返回地址
	fn        *funcval // can be nil for open-coded defers 注册的defer函数
	_panic    *_panic  // panic that is running defer 是触发延迟调用的结构体，可能为空；
	link      *_defer // 链接到前一个defer结构体，用于连接多个defer

	// 1.14以下三个 主要是找到未注册到链表的defer函数，并按照正确的函数顺序执行，
	fd   unsafe.Pointer // funcdata for the function associated with the frame  // 1.14
	varp uintptr        // value of varp for the stack frame // 1.14
	// framepc is the current pc associated with the stack frame. Together,
	// with sp above (which is the sp associated with the stack frame),
	// framepc/sp can be used as pc/sp pair to continue a stack trace via
	// gentraceback().
	framepc uintptr // 1.14
}
```

通过看汇编代码，可以看到在1.14版本，会有如下三种情况处理defer

```go
func (s *state) stmt(n *Node) {
    ...
	case ODEFER:
		if Debug_defer > 0 {
			var defertype string
			if s.hasOpenDefers {
				defertype = "open-coded"
			} else if n.Esc == EscNever {
				defertype = "stack-allocated"
			} else {
				defertype = "heap-allocated"
			}
			Warnl(n.Pos, "%s defer", defertype)
		}
		if s.hasOpenDefers {  // 开放源码
			s.openDeferRecord(n.Left)
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack  // 栈分配
			}
			s.call(n.Left, d)
		}
    ...
}
```

在1.12版本使用的是堆分配，1.13版本加入栈分配，现在1.14版本又加入开放源码，可以看到堆分配是最后的兜底方案。现在我们会来介绍每一种方案。

##### **1、堆分配(go 1.12)**

1) 	声明defer关键字处使用**deferproc**() 注册defer处理函数，将对应的_defer结构体值拷贝到堆上。

对于新创建好的defer结构体会添加在defer链表的表头，goroutine的defer指针指向链表的表头，这也是为什么倒序执行defer的原因。

![1609743425385](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1609743425385.png)



```go
 //注册的函数类型是funcval类型，也就是defer的执行函数
func deferproc(siz int32, fn *funcval) {
    ...
    // 插入头部
    d.link = gp._defer
	gp._defer = d
    ...
}
```

没有捕获列表（没有外部函数使用的变量）的funcval（闭包） ，在编译阶段会进行优化 在只读数据段分配一个共用的funcval结构体。但当有捕获列表的funcval 在堆里分配funcval结构体的大小，捕获列表使用的变量在堆中会存储该变量，其他地方使用该变量则使用该变量的地址，所以值改变的时候引用的地方同时改变

主要流程是获取一个siz大小的_defer结构体，可以在协程对应的处理器的defer池获取，若获取不到，需要在堆上分配size大小的结构体，然后将该结构体放在链表头。

2)	在ret指令返回时，插入**deferreturn()**，将defer链表的头取出执行。  执行时，会将值从堆上拷贝到栈上。

```go
func deferreturn(arg0 uintptr) {
    ...
    // 取出链表头
	fn := d.fn
	d.fn = nil
	gp._defer = d.link
    ...
}
```

**分析go1.12 defer的缺点**

1)	_defer结构体是堆分配，即使有预分配的defer池，也需要在堆上获取和释放，参数还要在堆栈间来回拷贝

2)	使用链表注册defer信息，链表本身操作较慢



##### **2、栈分配(go 1.13)**

1)	声明defer关键字处使用**deferproc**() 注册defer处理函数，把栈上的defer结构体注册到defer链表中。

```go
func deferprocStack(d *_defer) {
    // 增加变量，把defer信息保存到当前函数栈的局部变量(d)区域
	gp := getg()
	if gp.m.curg != gp {
		throw("defer on system stack")
	}
	d.started = false
	d.heap = false // 栈上分配
	d.openDefer = false
	d.sp = getcallersp()
	d.pc = getcallerpc()
	d.framepc = 0
	d.varp = 0

	*(*uintptr)(unsafe.Pointer(&d._panic)) = 0
	*(*uintptr)(unsafe.Pointer(&d.fd)) = 0
	*(*uintptr)(unsafe.Pointer(&d.link)) = uintptr(unsafe.Pointer(gp._defer))
	*(*uintptr)(unsafe.Pointer(&gp._defer)) = uintptr(unsafe.Pointer(d))

	return0()
}
```

2)	defer执行时，依然是通过deferreturn实现的，在defer函数执行时拷贝参数，不过不是在堆栈间，而是在栈上的局部变量空间拷贝到栈上的参数空间，性能提升30%

**所以1.3的优化点**：主要减少defer信息的堆分配。但是在显示和隐式的defer处理(例如for循环)还是使用defer1.2



##### **3、开放编码(go 1.4)**

1）开放编码的使用是有条件的，满足以下3个条件才会启用开放源码

```go
func walkstmt(n *Node) *Node {
	case ODEFER:
		Curfn.Func.SetHasDefer(true)
		Curfn.Func.numDefers++
    	// 1、函数的defer数量少于或者等于 8 个
		if Curfn.Func.numDefers > maxOpenDefers {
			// Don't allow open-coded defers if there are more than
			// 8 defers in the function, since we use a single
			// byte to record active defers.
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		if n.Esc != EscNever {
            // 2、函数的defer关键字不能在循环中执行
			// If n.Esc is not EscNever, then this defer occurs in a loop,
			// so open-coded defers cannot be used in this function.
			Curfn.Func.SetOpenCodedDeferDisallowed(true)
		}
		fallthrough    
}
func buildssa(fn *Node, worker int) *ssa.Func {
	if s.hasOpenDefers &&
    	// 3、函数的 `return` 语句与 `defer` 语句的乘积小于或者等于 15 个
		s.curfn.Func.numReturns*s.curfn.Func.numDefers > 15 {
		// Since we are generating defer calls at every exit for
		// open-coded defers, skip doing open-coded defers if there are
		// too many returns (especially if there are multiple defers).
		// Open-coded defers are most important for improving performance
		// for smaller functions (which don't have many returns).
		s.hasOpenDefers = false
	}    
}
```

延迟比特和延迟记录是开放源码的主要结构：

```go
func buildssa(fn *Node, worker int) *ssa.Func {
	if s.hasOpenDefers {
		// Create the deferBits variable and stack slot.  deferBits is a
		// bitmask showing which of the open-coded defers in this function
		// have been activated.
        // 初始化延迟比特,共8位
		deferBitsTemp := tempAt(src.NoXPos, s.curfn, types.Types[TUINT8])
		s.deferBitsTemp = deferBitsTemp
		// For this value, AuxInt is initialized to zero by default
		startDeferBits := s.entryNewValue0(ssa.OpConst8, types.Types[TUINT8])
		s.vars[&deferBitsVar] = startDeferBits
		s.deferBitsAddr = s.addr(deferBitsTemp, false)
		s.store(types.Types[TUINT8], s.deferBitsAddr, startDeferBits)
		// Make sure that the deferBits stack slot is kept alive (for use
		// by panics) and stores to deferBits are not eliminated, even if
		// all checking code on deferBits in the function exit can be
		// eliminated, because the defer statements were all
		// unconditional.
		s.vars[&memVar] = s.newValue1Apos(ssa.OpVarLive, types.TypeMem, deferBitsTemp, s.mem(), false)
	}
}
```

延迟比特中的每一个比特位都表示该位对应的 `defer` 关键字是否需要被执行。例如下面最后1位是1，说明接下来该位的defer会执行，使用或运算，把df的第一位置为1，标志该位可以执行，在执行的时候判断即可，判断完置为0，避免重复执行

![1609748577777](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1609748577777.png)

在编译阶段插入代码，把defer的执行逻辑展开在所属函数内，从而免于创建defer结构体，去了构造defer链表项，并注册到链表的过程，不需要创建defer链表。

```go
func (s *state) stmt(n *Node) {
    ...
	case ODEFER:
		if Debug_defer > 0 {
			var defertype string
			if s.hasOpenDefers {
				defertype = "open-coded"
			} else if n.Esc == EscNever {
				defertype = "stack-allocated"
			} else {
				defertype = "heap-allocated"
			}
			Warnl(n.Pos, "%s defer", defertype)
		}
		if s.hasOpenDefers {  // 开放源码
			s.openDeferRecord(n.Left) // 开放源码主要处理函数
		} else {
			d := callDefer // 堆分配
			if n.Esc == EscNever {
				d = callDeferStack  // 栈分配
			}
			s.call(n.Left, d)
		}
    ...
}
```

openDeferRecord主要记录了defer的信息保存在openDeferInfo结构体中。

```go
type openDeferInfo struct {
	// The ODEFER node representing the function call of the defer
	n *Node
	// If defer call is closure call, the address of the argtmp where the
	// closure is stored.
	closure *ssa.Value // 调用的函数
	// The node representing the argtmp where the closure is stored - used for
	// function, method, or interface call, to store a closure that panic
	// processing can use for this defer.
	closureNode *Node
	// If defer call is interface call, the address of the argtmp where the
	// receiver is stored
	rcvr *ssa.Value //方法的接收者
	// The node representing the argtmp where the receiver is stored
	rcvrNode *Node
	// The addresses of the argtmps where the evaluated arguments of the defer
	// function call are stored.
	argVals []*ssa.Value // 存储了函数的参数
	// The nodes representing the argtmps where the args of the defer are stored
	argNodes []*Node
}
```

2）在延迟执行的时候,调用deferreturn函数进行执行。把defer需要的参数定义为局部变量，在函数返回前，直接调用defer的函数

```go
func deferreturn(arg0 uintptr) {
	gp := getg()
	d := gp._defer
	if d == nil {
		return
	}
	sp := getcallersp()
	if d.sp != sp {
		return
	}
	if d.openDefer {
		done := runOpenDeferFrame(gp, d)
		if !done {
			throw("unfinished open-coded defers in deferreturn")
		}
		gp._defer = d.link
		freedefer(d)
		return
	}
	...
}
```

在执行的时候先获取deferBits，然后判断哪个defer该进行执行

```go
func runOpenDeferFrame(gp *g, d *_defer) bool {
	done := true
	fd := d.fd

	// Skip the maxargsize
	_, fd = readvarintUnsafe(fd)
	deferBitsOffset, fd := readvarintUnsafe(fd)
	nDefers, fd := readvarintUnsafe(fd)
	deferBits := *(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset)))

	for i := int(nDefers) - 1; i >= 0; i-- {
		// read the funcdata info for this defer
		var argWidth, closureOffset, nArgs uint32
		argWidth, fd = readvarintUnsafe(fd)
		closureOffset, fd = readvarintUnsafe(fd)
		nArgs, fd = readvarintUnsafe(fd)
        // 判断该位是否要执行defer,若为1时表示要执行，为0时不执行
		if deferBits&(1<<i) == 0 { 
			for j := uint32(0); j < nArgs; j++ {
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
				_, fd = readvarintUnsafe(fd)
			}
			continue
		}
		closure := *(**funcval)(unsafe.Pointer(d.varp - uintptr(closureOffset)))
		d.fn = closure
		deferArgs := deferArgs(d)
		// If there is an interface receiver or method receiver, it is
		// described/included as the first arg.
		for j := uint32(0); j < nArgs; j++ {
			var argOffset, argLen, argCallOffset uint32
			argOffset, fd = readvarintUnsafe(fd)
			argLen, fd = readvarintUnsafe(fd)
			argCallOffset, fd = readvarintUnsafe(fd)
			memmove(unsafe.Pointer(uintptr(deferArgs)+uintptr(argCallOffset)),
				unsafe.Pointer(d.varp-uintptr(argOffset)),
				uintptr(argLen))
		}
        // 找到后把该位置为0，避免重复执行
		deferBits = deferBits &^ (1 << i)
		*(*uint8)(unsafe.Pointer(d.varp - uintptr(deferBitsOffset))) = deferBits
		p := d._panic
        // 调用需要执行的defer函数
		reflectcallSave(p, unsafe.Pointer(closure), deferArgs, argWidth)
		if p != nil && p.aborted {
			break
		}
		d.fn = nil
		// These args are just a copy, so can be cleared immediately
		memclrNoHeapPointers(deferArgs, uintptr(argWidth))
		if d._panic != nil && d._panic.recovered {
			done = deferBits == 0
			break
		}
	}

	return done
}
```

1.14的缺点是：在某个defer函数执行过程中发生panic，后面的defer就不会被执行，就要去执行deferl链表了，但是这时open coded defer并没有注册到链表，这个时候需要额外通过栈扫描的方式来实现，所以panic变得慢了，但是panic发生的几率更小



这就是defer三种实现的原理

