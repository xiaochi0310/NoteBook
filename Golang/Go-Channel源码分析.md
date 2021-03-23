channel源码分析



#### 一、Channel的作用

Golang语言支持并发功能，并发口号是：不用通过共享内存来通信，而是通过通信来共享内存

其中关键之一在于Channel。Channel 相当于Llinux的管道，可以让多个协程之间通过读写操作进行消息传递。使用goroutine和channnel可以很容易产生一个生产者-消费者模式的任务队列。

下面这个例子更能说明协程之间是怎么交互的

```go
func main() {
	var ch1, ch2 chan int
	ch1 = make(chan int, 5)
	go writerChannel(ch1) // 协程1-通道1
	ch2 = make(chan int, 5)
	go writerChannel(ch2) // 协程2-通道2
	for { // 主goroutine监听其他2个协程，进行交互
		select { 
		case ch, ok := <-ch1: // 通道读数据
			if !ok {  // 通道关闭
				ch1 = nil
				fmt.Println("ch1 closed")
				continue
			}
			fmt.Println("ch1 reader", ch) // 打印读取数据

		case ch, ok := <-ch2: 
			if !ok {
				ch2 = nil
				fmt.Println("ch2 closed")
				continue
			}
			fmt.Println("ch2 reader", ch)
		default:
			break
		}
		if ch1 == nil && ch2 == nil {
			break
		}
	}
	fmt.Println("channel finished")
}

// 通道写数据
func writerChannel(ch chan int) {
	for i := 0; i < 5; i++ {
		ch <- i
	}
	close(ch)
}
```

通道的主要特性：

1、协程安全，因为有全局锁
2、通道的数据FIFO
3、可以使得协程阻塞和高效的恢复
4、通道的协程调用runtime scheduler实现，OS thread不需要阻塞，跨goroutine栈可以直接进行读写

#### 二、源码解析

##### 1、例子

现在我们举个简单的例子开始分析源码

```go
func main() {
	var ch chan int
	ch = make(chan int,10) // 缓存为10的通道
	go func() { // 协程写
		ch <-1 
	}()
	<-ch // 同步读
	close(ch) // 关闭通道
}
```

我们编译的时候查看汇编代码

```bash
go tool compile -N -l -S lock.go

# make(chan int,10) ==> makechan
CALL    runtime.makechan(SB)
# go func() ==> newproc(SB)
CALL    runtime.newproc(SB)
	# 子协程 ch <-1 ==> chansend1
	CALL    runtime.chansend1(SB)
# <-ch ==> chanrecv1
CALL    runtime.chanrecv1(SB)
# close(ch) ==> closechan
CALL    runtime.closechan(SB)
```

通过汇编，把代码中的符号链接对应的处理函数

##### 2、结构体

```go
type hchan struct {
	qcount   uint           // 队列总元素个数,实际队列里面的大小
	dataqsiz uint           // 环形队列长度,缓冲区的大小(make的时候指出的大小)，非缓存=0
	buf      unsafe.Pointer // 指向环形队列指针
	elemsize uint16 // 每个元素的大小
	closed   uint32 // 当前通道是否是关闭状态 关闭=1 非关闭=0
	elemtype *_type // 元素类型
	sendx    uint   // 环形缓冲区中发送位置索引(环形队列2个指针)
	recvx    uint   // 环形缓冲区中接收位置索引
	recvq    waitq  // 读消息的goroutine队列
	sendq    waitq  // 写消息的goroutine队列

	lock mutex // 全局互斥锁 读写时锁通道
}

type waitq struct {
	first *sudog
	last  *sudog
}
```

**缓冲区**是一个环形队列，使用双指针实现FIFO，sendx和recvx分别表示读写的索引。

![1616130033046](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616130033046.png)

**channel对于阻塞的协程**，分配到`send`和`recv`队列，队列里面都是持有goroutine的sudog元素，队列都是双链表实现的。队列是waitq类型，而waitq 为双向链表，sudog 代表一个封装的 goroutine，其参数 g 为 goroutine 实例结，构如下图：

![1616130881236](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1616130881236.png)

[该图来自Golang 源码导读 —— channel](https://studygolang.com/articles/21586?fr=sidebar)

##### 3、通道创建

```go
// ch = make(chan int,10)
// ch = make(chan int)
// go/src/runtime/chan.go
func makechan(t *chantype, size int) *hchan {
	elem := t.elem

	// buffer的大小  elem.size*size
	mem, overflow := math.MulUintptr(elem.size, uintptr(size))
    // 缓冲不能为负数，申请内存不能超过最大值
	if overflow || mem > maxAlloc-hchanSize || size < 0 {
		panic(plainError("makechan: size out of range"))
	}

	var c *hchan
	switch {
	case mem == 0:
		// size为0，则只分配结构体大小
		// hchanSize 表示空的hchan需要占用的字节大小
		c = (*hchan)(mallocgc(hchanSize, nil, true))
		c.buf = c.raceaddr()
	case elem.ptrdata == 0:
		// 表示没有指针
		// 数据项不为指针类型，调用 mallocgc 一次性分配内存大小，hchan 结构体大小 + 数据总量大小
		c = (*hchan)(mallocgc(hchanSize+mem, nil, true))
		// 指针偏移hchanSize大小，现在指的是数据开始的位置
		c.buf = add(unsafe.Pointer(c), hchanSize)
	default:
		// 有指针
		// Elements contain pointers.
		c = new(hchan)
        // hchan 和 buf 分开分配内存，GC 中指针类型判断 reachable and unreadchable
		c.buf = mallocgc(mem, elem, true)
	}
	// 设置chan的总大小
	c.elemsize = uint16(elem.size)
	c.elemtype = elem
	// 环形队列的大小 缓存buffer的值
	c.dataqsiz = uint(size)

	// 返回一个chan
	return c
}
```

##### 4、发送信息

​	分别是三种情况

- 判断等待读数据队列中有goroutine，直接发送数据给goroutine
- 缓冲区没满，将数据写在缓冲区中
- 缓冲器满，当且goroutine阻塞，放在等待写数据队列中

```go
// ep 指向要发送数据的首地址
// ch <- 1
func chansend(c *hchan, ep unsafe.Pointer, block bool, callerpc uintptr) bool {
    // 若是在空chan通道写数据，将该协程进入休眠状态
    // fatal error: all goroutines are asleep - deadlock! 
	if c == nil {
		if !block {
			return false
		}
		gopark(nil, nil, waitReasonChanSendNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	....
    
	// 加锁
	lock(&c.lock)
	// 给关闭的chan写数据，报错
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("send on closed channel"))
	}
	// 有goroutine在等待接收数据
	if sg := c.recvq.dequeue(); sg != nil {
		// 发送数据，直接发送给该goroutine
		send(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true
	}
	// 缓冲区没有满
	if c.qcount < c.dataqsiz {
		// 待增加元素的位置
		qp := chanbuf(c, c.sendx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		// 将添加的数据放在目的地址上
		typedmemmove(c.elemtype, qp, ep)
        // sendx索引++
		c.sendx++
		if c.sendx == c.dataqsiz { // 循环队列，sendx转成0
			c.sendx = 0
		}
		// 缓冲通道元素个数++
		c.qcount++
		// 结束返回
		unlock(&c.lock)
		return true
	}
	// 缓存区已经满了
	// 同步非阻塞的情况----
	// todo
	if !block {
		unlock(&c.lock)
		return false
	}
	// 同步阻塞
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.waiting = mysg
	gp.param = nil
	// 将新goroutine放在发送队列
	c.sendq.enqueue(mysg)
	atomic.Store8(&gp.parkingOnChan, 1)
    // 休眠
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanSend, traceEvGoBlockSend, 2)
	...
	releaseSudog(mysg)
	return true
}
```

- 等待读数据队列中有goroutine，直接发送数据给goroutine

```go
func send(c *hchan, sg *sudog, ep unsafe.Pointer, unlockf func(), skip int) {
	...
	if sg.elem != nil {
        // 直接将数据拷贝到接收的goroutine上-go对应的elem指向的内存地址
		sendDirect(c.elemtype, sg, ep)
		sg.elem = nil
	}
	gp := sg.g
	unlockf()
	gp.param = unsafe.Pointer(sg)
	if sg.releasetime != 0 {
		sg.releasetime = cputicks()
	}
    // 唤醒对应的goroutine
	goready(gp, skip+1)
}
```

##### 5、接收信息

分别三种情况：

- 判断等待写数据队列中有goroutine，直接读对应协程的数据
- 缓存区没控，则读缓冲区的数据
- 缓冲区为空，将当前读数据的goroutine放在等待读数据队列中

```go
func chanrecv(c *hchan, ep unsafe.Pointer, block bool) (selected, received bool) {

	if c == nil {
		if !block {
			return
		}
		gopark(nil, nil, waitReasonChanReceiveNilChan, traceEvGoStop, 2)
		throw("unreachable")
	}
	....
    
	// 上锁
	lock(&c.lock)
	// 若通道关闭且通道的值为0，不能继续读
	if c.closed != 0 && c.qcount == 0 {
		if raceenabled {
			raceacquire(c.raceaddr())
		}
		unlock(&c.lock)
		if ep != nil {
			typedmemclr(c.elemtype, ep)
		}
		return true, false
	}
	// 首先从等待发送数据队列的协程读取数据，
	if sg := c.sendq.dequeue(); sg != nil {
		recv(c, sg, ep, func() { unlock(&c.lock) }, 3)
		return true, true
	}
	// 从缓冲队列读数据
	if c.qcount > 0 { // 缓冲区不为空
		// Receive directly from queue
		// 找到对应数据的地址，索引对应的位置
		qp := chanbuf(c, c.recvx)
		if raceenabled {
			raceacquire(qp)
			racerelease(qp)
		}
		// 将数据拷贝到接收数据的协程
		if ep != nil {
			typedmemmove(c.elemtype, ep, qp)
		}
		typedmemclr(c.elemtype, qp)
		// 接收数据索引++
		c.recvx++
		// 环形队列，到末尾
		if c.recvx == c.dataqsiz {
			c.recvx = 0
		}
		// 缓冲数量--
		c.qcount--
		unlock(&c.lock)
		return true, true
	}
	// 同步非阻塞
	if !block {
		unlock(&c.lock)
		return false, false
	}

	// 同步阻塞
	// 阻塞该协程，将其加入recvq goroutine队列
	gp := getg()
	mysg := acquireSudog()
	mysg.releasetime = 0
	if t0 != 0 {
		mysg.releasetime = -1
	}
	// No stack splits between assigning elem and enqueuing mysg
	// on gp.waiting where copystack can find it.
	mysg.elem = ep
	mysg.waitlink = nil
	gp.waiting = mysg
	mysg.g = gp
	mysg.isSelect = false
	mysg.c = c
	gp.param = nil
	// 放入队列
	c.recvq.enqueue(mysg)

	atomic.Store8(&gp.parkingOnChan, 1)
    // 休眠
	gopark(chanparkcommit, unsafe.Pointer(&c.lock), waitReasonChanReceive, traceEvGoBlockRecv, 2)
	...
	return true, !closed
}
```

##### 6、关闭通道

```go
func closechan(c *hchan) {
	if c == nil {
		panic(plainError("close of nil channel"))
	}
	// 上锁
	lock(&c.lock)
	// 重复关
	if c.closed != 0 {
		unlock(&c.lock)
		panic(plainError("close of closed channel"))
	}

	// 设置标记位
	c.closed = 1

	var glist gList // 临时队列

	// release all readers
	// 处理所有读数据的goroutine
	for {
		sg := c.recvq.dequeue()
		if sg == nil {
			break
		}
		if sg.elem != nil {
			typedmemclr(c.elemtype, sg.elem)
			sg.elem = nil
		}
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
        // 加入临时队列
		glist.push(gp)
	}

	// release all writers (they will panic)
	for {
		sg := c.sendq.dequeue()
		if sg == nil {
			break
		}
		sg.elem = nil
		if sg.releasetime != 0 {
			sg.releasetime = cputicks()
		}
		gp := sg.g
		gp.param = nil
		if raceenabled {
			raceacquireg(gp, c.raceaddr())
		}
		// 加入临时队列
		glist.push(gp)
	}
	unlock(&c.lock)

	// Ready all Gs now that we've dropped the channel lock.
    // 唤醒所有的goroutine
	for !glist.empty() {
		gp := glist.pop()
		gp.schedlink = 0
		goready(gp, 3)
	}
}
```

#### 三、思考

1、创建通道的内存是什么内存

make chan其实是在堆中分配了一个hchan结构，并返回其指针指向堆地址空间以便于使用

2、创建 channel 所申请的内存，在其被 close 后何时才会释放内存？

需要等待垃圾回收器的配合（GC）。举例来说，对于某个 channel 而言，所通信双方的 goroutine 均已进入 dead 状态，则垃圾回收器会将 channel 创建时申请的内存回收到待回收的内存池，在当下一次用户态代码申请内存时候，会按需对内存进行清理（内存分配器的工作原理）；由此可见：如果我们能够确信某个 channel 不会使其通信的 goroutine 发生阻塞，则不必将其关闭，因为垃圾回收器会帮我们进行处理。

3、Channel的缺点

- channel可能会导致死锁（循环阻塞）
- channel中传递的都是数据的拷贝，可能会影响性能
- channel中传递指针会导致数据竞态问题（data race/ race conditions）。data race是多线程并发读写一个变量，对应到Golang中就是多个goroutine同时读写一个变量，这种行为是未定义的，也就是说读变量出来的值很有可能不是写入的值，这个值是任意值都有可能

4、互斥锁能解决并发问题，channel也能解决，那么什么情况下使用这2种方式呢







#### 引用

[golang并发三板斧系列之一：channel用于通信和同步](https://studygolang.com/articles/20318)

[Golang-Channel原理解析](https://blog.csdn.net/u010853261/article/details/85231944)

[Golang 源码导读 —— channel](https://studygolang.com/articles/21586?fr=sidebar)