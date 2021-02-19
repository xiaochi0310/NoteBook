Go堆内存(四)

Golang什么情况下会申请堆内存，由什么决定的，学到堆内存一定会有这样的一个疑问…

栈内存：一般由系统申请和释放。比如函数的入参、局部变量、返回值等

堆内存：一般由程序员申请和释放(malloc/free new/delete等)。使用malloc关键字申请的内存就在堆内存，申请和释放要成对使用，否则会造成内存泄漏。对于Golang系统会自动回收已经不使用的堆内存，这就是GC的工作啦

> How do I know whether a variable is allocated on the heap or the stack?
>
> From a correctness standpoint, you don't need to know. Each variable in Go exists as long as there are references to it. The storage location chosen by the implementation is irrelevant to the semantics of the language.
>
> The storage location does have an effect on writing efficient programs. When possible, the Go compilers will allocate variables that are local to a function in that function's stack frame. However, if the compiler cannot prove that the variable is not referenced after the function returns, then the compiler must allocate the variable on the garbage-collected heap to avoid dangling pointer errors. Also, if a local variable is very large, it might make more sense to store it on the heap rather than the stack.
>
> In the current compilers, if a variable has its address taken, that variable is a candidate for allocation on the heap. However, a basic escape analysis recognizes some cases when such variables will not live past the return from the function and can reside on the stack.

先来看下逃逸分析，逃逸分析也就是由编译器决定哪些变量放在栈，哪些放在堆中

若将变量放在栈中，则函数执行完成由系统立即释放；若放在堆中，函数执行完成由GC进行处理

> go build -gcflags=-m  可以检查代码的编译优化情况，包括逃逸情况和函数是否内联

**1、指针逃逸**

指针指向的内存指向的是堆中的地址，不是栈的

```go
type stuInFo struct {
	name string
	age int
}

func main() {
	s := new(stuInFo)
	s.name = "name"
	s.age = 7
	fmt.Println(s)
}

# go build -gcflags=-m
.\main.go:17:10: new(stuInFo) escapes to heap
```

在编译阶段，可以看到s指向的地址是发生逃逸到heap，此时后续函数执行完，会触发GC处理

**2、栈空间不足**

看一下这个例子eg1，很简单就是创建一个切片，然后赋值。说明编译器认为局部变量较小不足以放在堆上

```go
// eg1:
func main() {
	s := make([]int, 1000, 1000)
	for index, _ := range s {
		s[index] = index
	}
}

# go build -gcflags=-m
.\main.go:s:11: make([]int, 1000, 1000) does not escape
```

这里有个诡异的问题，纠结了小编很久

再看下面的例子eg2

```go
// eg2:
func main() {
	s := make([]int, 1000, 1000)
	for index, _ := range s {
		s[index] = index
	}
	fmt.Println(s)
}

# go build -gcflags=-m
.\main.go:6:13: inlining call to fmt.Println
.\main.go:2:11: make([]int, 1000, 1000) escapes to heap
.\main.go:6:13: s escapes to heap
```

这个例子比上面的例子多了一个println打印，其余的都是一样的，但是莫名的就发生逃逸。

查了资料，解释是fmt.Println接口是func Println(a ...interface{})，参数是可变类型的，所以会导致参数总是逃逸。

接下来我们在eg1的基础上，增加栈内存(1000->10000)，看eg3

```go
// eg3:
func main() {
	s := make([]int, 10000, 10000)
	for index, _ := range s {
		s[index] = index
	}
}

# go build -gcflags=-m
.\main.go:3:11: make([]int, 10000, 10000) escapes to heap
```

在eg3中可以看到扩大栈内存，发生了逃逸，把局部变量s分配到堆上

**3、动态类型**

理解了2中的eg2，就知道这里的动态类型逃逸是什么意思了

动态类型就是编译期间不确定参数的类型、参数的长度也不确定的情况下就会发生逃逸，如interface{}接口eg2

```go
func main() {
   var a string = "hello"
   test(a)
}

func test(a ...string) {
	fmt.Println(a)
}

.\main.go:42:8: a escapes to heap
```

string类型不确定长度，所以发生逃逸

**4、闭包引用对象**

```go
type Func func(x int) int
func A() Func {
	var i int = 1
	return func(x int) int {
		i++
		return x+i
	}
}

func main() {
	var res Func= A()
	res(1)
}

D:\GO_WORK\src\xctest2\main>go build -gcflags=-m
# xctest2/main
.\main.go:32:9: can inline A.func1
.\main.go:31:6: moved to heap: i
.\main.go:32:9: func literal escapes to heap
```

闭包函数A()中局部变量i在后续函数是继续使用的，所以分配到堆中



**总结：**

可以看到编译器越来越聪明了，在编译阶段进行逃逸分析，把变量分配在哪安排的明明白白，所以这样可以降低GC的压力，只要分配到堆的变量才触发GC进行垃圾回收。

可以看到有时候传指针也不一定高效，因为无论变量的大小，只要是指针变量都会在堆上分配，所以对于小变量我们还是使用传值效率更高一点。

想必到现在你已经摸清了什么情况下变量被分配到堆中，在代码编写的时候，我们可以尽量最优化，减少堆的分配和GC的压力



**引用**

> Golang的GC原理https://studygolang.com/articles/29930