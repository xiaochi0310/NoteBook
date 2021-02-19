### Go语言

------

#### 1、omitempty的使用

omitempty作用是在json数据结构转换时，当该字段的值为该字段类型的零值时，忽略该字段。例如为false，0，零指针，nil接口值以及任何空数组，切片，映射或字符串，则该字段应从编码中省略。举个例子：

```go
// 场景一 类型不加omitempty
type stu struct {
	Name string `json:"name,omitempty"`
	Age  int    `json:"age,omitempty"`
}
// 场景二 int、string 类型不加omitempty
type stu struct {
	Name string `json:"name"`
	Age  int    `json:"age"`
}

func main() {
	data := stu{
		Name: "",
	}
	fmt.Printf("start,%v\n", data)
	// 有omitempty { 0}
	// 没有omitempty { 0}
	res, err := json.Marshal(data)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("marshall,%s\n", res)
	// BAD!!有omitempty {} 优点：““以及0的字段不显示, 缺陷：如果填了有意义的"",则不会显示
	// BAD!!没有omitempty {"name":"","age":0} ““以及0的字段显示，如果没填会显示默认值
	var req = new(stu)
	err = json.Unmarshal(res, req)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("unmarshall,%v\n", req)
	// 有omitempty { 0}
	// 没有omitempty { 0}
}
```

但是这个时候，如果我们想要Name为""时就是有意义，即要显示出来。如果我不填这个字段，这个字段不显示。也就是要区分空字符串到底是没传这个字段还是就是填了一个"",那这个时要怎么做呢?

答案就是把字段定义成指针类型

```go
type stu struct {
	Name *string `json:"name,omitempty"`
	Age  *int    `json:"age,omitempty"`
}
func main() {
	a := ""
	data := stu{
		Name: &a,
	}
	fmt.Printf("start,%v\n", data)
	// 有omitempty {0xc00003a260 <nil>}
	// 没有omitempty {0xc00003a260 <nil>}
	res, err := json.Marshal(data)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("marshall,%s\n", res)
	// GOOD!!有omitempty {"name":""} 我们填的空依旧显示，但是没填的age没有显示，说明name是填了“” 
	// BAD!!没有omitempty {"name":"","age":null} 优点：name填了空显示，没填的会显示默认值 缺陷：无法区分name的“”是没填这个值还是填了一个“”
	var req = new(stu)
	err = json.Unmarshal(res, req)
	if err != nil {
		fmt.Printf("%v\n", err)
	}
	fmt.Printf("unmarshall,%v\n", req)
}
```

定义成指针我们就可以识别空串，0等空值

20201112补充

对于字段中存在结构体类型,加omitempty字段的区别

```go
type Student struct {
	Age  string `json:"age"`
	Name []Name `json:"name,omitempty"` // omitempty
}

type Name struct {
	first string `json:"first"`
	next  string `json:"next"`
}

func main() {
	data := `{
	   "age": "9"
	}`
	payload := []byte(data)
	var Student = new(Student)
	err := json.Unmarshal(payload, Student)
	if err == nil {
		fmt.Println(Student)
	} else {
		fmt.Println(err)
	}
	stu, err := json.Marshal(Student)
	fmt.Printf("%s", stu) // {"age":"9"}，忽略空字段
    // 若不加omitempty {"age":"9","name":null}
}
```



#### 2、数据类型的占位符

```go
	var i1 int = 1
	var i2 int8 = 2
	var i3 int16 = 3
	var i4 int32 = 4
	var i5 int64 = 5
	fmt.Println(unsafe.Sizeof(i1)) // 8
	fmt.Println(unsafe.Sizeof(i2)) // 1
	fmt.Println(unsafe.Sizeof(i3)) // 2
	fmt.Println(unsafe.Sizeof(i4)) // 4
	fmt.Println(unsafe.Sizeof(i5)) // 8
```

#### 3、init函数

go语言的init函数，分2种情况：

1>  引入包只为了使用包里面的init函数（如注册函数），此时需手动调用

```
import _"packageName“
```

2>  包里面的其他函数被其他包调用，此时该包的init无需手动调用，编译器会识别引用包里面的init函数

#### 4、闭包与匿名函数的区别与作用

**匿名函数**：没有名字的函数。是为了开辟封闭的变量作用域环境。用于构造闭包

```go
func main() {
    // 方法1
	func() {
		fmt.Println("hello1")
	}()
	// 方法2
	fn := func() {
		fmt.Println("hello2")
	}
	fn()
	// 方法3
	fn1 := func(str string) {
		fmt.Println(str)
	}
	fn1("hello3")
	// 方法4
	fn2 := func() string {
		return "hello4"
	}
	fmt.Println(fn2())
}

/*
    hello1
    hello2
    hello3
    hello4
*/
```

**闭包**：利用匿名函数实现。可以在函数外部访问函数的变量，函数调用返回后一个没有释放资源的栈区。不需传值也在创建函数中使用自己的私有变量。闭包返回的是一个复合结构：包括匿名函数的地址以及变量的地址

优点：减少代码量，使这些局部变量始终保存着内存中，避免使用全局变量
缺点：这些局部变量不会立即销毁，浪费内存

```go
type Func func(x int) int //定义函数类型
func A() Func { 
	var i = 1
	return func(a int) int{  // 闭包
		i++
		return a+i
	}
}

func main() {
	res := A() // res = A.func1 
    fmt.Println(res(1)) // A.func1(1) = 3
	fmt.Println(res(1)) // A.func1(1) = 4
}
```

#### 5、range函数

几个case 说明常见的问题

```go
func main() {
	// case1
	a := []int{100,200,300}
	for _,v := range a {
		a = append(a,v)
	} // 注意这里不会无限循环下去
	fmt.Println(a) // 1,2,3,1,2,3
	// 解释：这里是因为汇编的时候，已经对数组的长度设定为局部变量，进行值传递
	// case2
	b := []int{100,200,300}
	addr := []*int{}
	for _,v := range b {
		addr = append(addr,&v)
	}
	for _,address := range addr {
		fmt.Println(address) // 0xc0000140e0 0xc0000140e0 0xc0000140e0
	}
	// 解释：这里也是汇编的时候，获取值的时候会把值赋给局部变量，相当于v的地址一直是局部变量的地址，所以最后输出的值也是一直不变的
    // 对于所有的 range 循环，Go 语言都会在编译期间将原切片或者数组赋值给一个新的变量 ha，在赋值的过程中其实就发生了拷贝，所以我们遍历的切片其实已经不是原有的切片变量了
	// 针对2的问题修改case2
	b2 := []int{100,200,300}
	addr2 := []*int{}
	for i,_:= range b2 {
		addr2 = append(addr2,&b2[i])
	}
	for _,address := range addr2 {
		fmt.Println(address) // 0xc00000c3c0 0xc00000c3c8 0xc00000c3d0
	}
	// case3
	c := map[int]int {
		1:1,
		2:2,
		3:3,
	}
	for _,v := range c {
		fmt.Println(v) // 不稳定会按照1,2,3顺序输出
	}
	// 解释：汇编过程中 会找一个随机的节点开始，这就是GO团队的设计
}
```

#### 6、斐波那契数列（使用select）

```go
func main() {
    var c,stop chan int
	c = make(chan int)
	stop = make(chan int)
	go func() {
		fmt.Println("消费者开始")
		for i:=0;i<5;i++ {
			num := <-c
			fmt.Println(num)
		}
		stop <- 0
	}()
	fmt.Println("生产者开始")
	fibonacci(c,stop)
}

func fibonacci(c, stop chan int)  {
	x,y := 0,1
	for {
        // 监听通道数据
		select {
		case c <- x:
			fmt.Println("x =",x)
			x,y = y,x+y
		case <- stop:
			fmt.Printf("stop")
			return
		}
	}

}
```

#### 7、Selsect

1. `select` 能在 Channel 上进行非阻塞的收发操作；加上default分支即可，有时候我们不关系通道消息是否接收到，只在乎通道是否有错。
2. `select` 在遇到多个 Channel 同时响应时会随机挑选 `case` 执行；



