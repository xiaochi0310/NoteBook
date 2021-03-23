Golang的设计模式



#### 一、设计模式

##### 1、是什么

> 设计模式(Design Pattern)是一套被反复使用、多数人知晓的、经过分类编目的、代码设计经验的总结，使用设计模式是为了可重用代码、让代码更容易被他人理解并且保证代码可靠性。

通俗来说：是一个项目的代码层面设计架构，代码功能的排版。相当于模板。这些模式都是巨人们在软件系统中总结出的成功的、能够实现可维护性复用的设计方案

##### 2、作用

使用了统一模板肯定会有很大的好处，都有哪些呢？

- 设计模式提供了一套通用的设计词汇和一种通用的形式来方便开发人员之间沟通和交流，使得设计方案更加通俗易懂
- 设计模式增强了系统的可重用性、可扩展性、可维护性
- 提高代码的易读性，有助于别人更快地理解系统
- 有助于更加深入地理解面向对象思想

#### 二、常用设计模式

##### 1、简单工厂

###### 1）结构

![1615951511175](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1615951511175.png)

- 工厂：创建对象，创建产品实例的内部逻辑，外部不感知。外部直接调用，创建所需的产品对象
- 产品：所有对象的公共方法，也就是父类。内部方法只声明不定义。
- 具体产品：具体产品的公共方法的具体实现。

###### 2）使用场景

- 工厂类负责创建的对象比较少，由于创建的对象较少，不会造成工厂方法中的业务逻辑太过复杂。
- 客户端只知道传入工厂类的参数，对于如何创建对象并不关心

###### 3）优点

- 工厂类包含必要的判断逻辑，可以决定在什么时候创建哪一个产品类的实例，简单工厂模式实现了对象创建和使用的分离

###### 4）缺点

- 由于工厂类集中了所有产品的创建逻辑，职责过重，一旦不能正常工作，整个系统都要受到影响
- 系统扩展困难，一旦添加新产品需要修改工厂逻辑，在产品类型较多时，有可能造成工厂逻辑过于复杂，不利于系统的扩展和维护

###### 5）代码

笔工厂需要生产2种笔，铅笔(pencil)和圆珠笔(ballPen)，其均有书写功能。笔工厂生产具体产品铅笔，具体产品铅笔实现书写功能

```go
const (
	pencil = iota // 铅笔
	ballPen // 圆珠笔
)

// 文具工厂
type PenFactory struct {}

// New不同的工厂实例，也就是具体产品。（铅笔、圆珠笔）
func (t *PenFactory) New(typ int) Pen{
	switch typ {
	case pencil:
		return new(Pencil)
	case ballPen:
		return new(BallPen)
	default:
		return nil
	}
}

// 产品，声明工厂所有产品的公共方法，放在具体产品来实现
type Pen interface {
	Write()
}

// 铅笔。具体产品
type Pencil struct {}
// 公共方法的具体实现
func (*Pencil) Write() {
	fmt.Println("Pencil Writing")
}
// 圆珠笔，具体产品
type BallPen struct {}
// 公共方法的具体实现
func (*BallPen) Write() {
	fmt.Println("BallPen Writing")
}


func main() {
	var factory PenFactory
	pen := factory.New(pencil) // 使用工厂来生产一个具体产品，具体如何定义，用户不感知
	pen.Write()
}
```

##### 2、工厂方法

###### 1）结构

![1615963041293](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1615963041293.png)

- 工厂：不在实现创建所有的具体产品实例，只声明一个创建接口，具体初始化动作交给具体工厂
- 具体工厂：实现工厂的方法，返回一个具体产品的实例
- 产品：所有对象的公共方法，也就是父类。内部方法只声明不定义。
- 具体产品：由具体工厂创建，实现父类方法

###### 2）使用场景

在工厂方法模式中，客户端不需要知道具体产品类的类名，只需要知道所对应的具体工厂即可，具体的产品对象由具体工厂类创建。

###### 3）优点

- 用户只需要关心所需产品对应的工厂，无须关心创建细节
- 基于工厂角色和产品角色的多态性
- 在系统中加入新产品时，只要添加一个具体工厂和具体产品就可以了，系统的可扩展性好

###### 4）缺点

- 在添加新产品时，需要编写新的具体产品类，而且还要提供与之对应的具体工厂类，系统中类的个数将成对增加，在一定程度上增加了系统的复杂度，有更多的类需要编译和运行，会给系统带来一些额外的开销

###### 5）代码

笔工厂需要生产2种笔，铅笔(pencil)和圆珠笔(ballPen)，其均有书写功能。笔工厂生产具体工厂-铅笔工厂，铅笔工厂生产具体产品铅笔，具体产品实现书写功能

```go
// 工厂，New由具体工厂实现
type PenFactory interface {
	New() Pen
}
// 产品，所有具体产品的父类
type Pen interface {
	Write()
}

// 具体产品，圆珠笔
type BallPen struct {}
func (*BallPen) Write() {
	fmt.Println("BallPen Writing")
}

// 具体产品工厂
type BallPenFactory struct {}
func (p *BallPenFactory) New() Pen {
    fmt.Println("create BallPen")
	return new(BallPen)
}

// 铅笔，具体产品
type Pencil struct {}
func (*Pencil) Write() {
	fmt.Println("Pencil Writing")
}
// 具体产品工厂
type PencilFactory struct {}
func (p *PencilFactory) New() Pen {
	fmt.Println("create Pencil")
	return new(Pencil)
}

func main() {
	var factory PenFactory
	factory = &PencilFactory{} // 生成一个具体产品
	var pen Pen
	pen = factory.New()
	pen.Write()
}
```

##### 3、抽象工厂

###### 1）结构

![img](https://img-blog.csdn.net/20130713163800203?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTG92ZUxpb24=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

[本图选取于引用中的文章](https://blog.csdn.net/lovelion/article/details/9319423)，由于画的十分清晰，我们用这个图进行说明

   ● 抽象工厂（AbstractFactory）：在抽象工厂中声明了多个具体工厂方法，用于创建不同类型的产品。可以看到，与第二节的工厂方法有所不同，该抽象工厂可以不止可以New一个具体产品，还可以生产一系列的产品族(A,B,C)，也就是抽象工厂实例具体工厂实例有创建这一系列产品的接口

- 具体工厂（ConcreteFactory）：具体工厂实现了抽象工厂，每一个具体的工厂方法可以返回一个特定的产品对象。它实现了在抽象工厂中声明的创建产品的方法，生成一组具体产品(A,B,C)，这些产品构成了一个产品族

   ● 抽象产品（AbstractProduct）：它为每种产品声明接口，在抽象产品中声明了产品所具有的业务方法。

   ● 具体产品（ConcreteProduct）：它定义具体工厂生产的具体产品对象，实现抽象产品接口中声明的业务方法。

###### 2）使用场景

 系统中有多于一个的产品族，而每次只使用其中某一产品族。

###### 3）优点

- 抽象工厂模式隔离了具体类的生成，使得客户并不需要知道哪些具体产品被创建

- 当一个产品族中的多个对象被设计成一起工作时，它能够保证客户端始终只使用同一个产品族中的对象

- 增加新的产品族很方便，无须修改已有系统，符合“开闭原则”

###### 4）缺点

- 增加新的产品等级结构麻烦，需要对原有系统进行较大的修改，甚至需要修改抽象层代码

###### 5）代码

皮肤库有Spring、Summer风格。Spring风格有浅绿色按钮、绿色文本框。Summer风格有浅蓝色按钮、浅蓝色文本框。

```go
//界面皮肤工厂接口：抽象工厂
type SkinFactory interface  {
	createButton() Button
	createTextField() TextField
}
// 抽象产品1
type Button interface{
	display();
}
// 抽象产品2
type TextField interface{
	write();
}

// 具体产品1
type SpringButton struct {}
func (s *SpringButton)display(){
	fmt.Println("SpringButton display")
}
// 具体产品2
type SpringTextField struct {}
func (s *SpringTextField)write(){
	fmt.Println("SpringTextField write")
}
// 具体产品3
type SummerTextField struct {}
func (s *SummerTextField)write(){
	fmt.Println("SummerTextField write")
}
// 具体产品4
type SummerButton struct {}
func (s *SummerButton)display(){
	fmt.Println("SummerButton display")
}


//Spring皮肤工厂：具体工厂
type SpringSkinFactory struct {}
// 创建具体产品1
func (s *SpringSkinFactory) createButton() Button {
	return new(SpringButton)
}
// 创建具体产品2
func (s *SpringSkinFactory) createTextField() TextField {
	return new(SpringTextField)
}

//Summer皮肤工厂：具体工厂
type SummerSkinFactory struct {}
// 创建具体产品3
func (s *SummerSkinFactory) createButton() Button {
	return new(SummerButton)
}
// 创建具体产品4
func (s *SummerSkinFactory) createTextField() TextField {
	return new(SummerTextField)
}


func main() {
	var skinFactory SkinFactory
	skinFactory = &SummerSkinFactory{} // 创建一个产品族（具体产品3，具体产品4）
    but := skinFactory.createButton() // 创建某一个产品（具体产品3）
	but.display()
}

/*
C:\Users\Administrator\AppData\Local\Temp\___go_build_xctest2_main.exe #gosetup

SummerButton display
*/
```

##### 4、单例

###### 1）结构

单例模式(Singleton Pattern)：确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例，这个类称为单例类，它提供全局访问的方法。比如数据库连接、资源配置。

常见有2种模式：饿汉模式、懒汉模式

- 饿汉模式：在程序初始化的时候实例化对象，无需考虑多线程问题。golang里面的`init`接口
- 懒汉模式：单例在第一次使用时创建，无须一直占用系统资源，实现了延迟加载，但是必须处理好多个线程同时访问的问题。需要考虑的是多线程并发调用而导致的会创建多个实例的问题，需要用到互斥锁。golang语言常用`sync.once`实现懒汉模式，`sync.once`自带互斥锁

###### 2）使用场景

- 系统只需要一个实例对象，如系统要求提供一个唯一的序列号生成器或资源管理器，或者需要考虑资源消耗太大而只允许创建一个对象

###### 3）优点

-  由于在系统内存中只存在一个对象，因此可以节约系统资源，对于一些需要频繁创建和销毁的对象单例模式无疑可以提高系统的性能。


###### 4）缺点

- 单例类的职责过重，在一定程度上违背了“单一职责原则”。因为单例类既充当了工厂角色，提供了工厂方法，同时又充当了产品角色，包含一些业务方法，将产品的创建和产品的本身的功能融合到一起

- golang运行环境都提供了自动垃圾回收的技术，因此，如果实例化的共享对象长时间不被利用，系统会认为它是垃圾，会自动销毁并回收资源，下次利用时又将重新实例化，这将导致共享的单例对象状态的丢失

###### 5）代码

```go
func main() {
	var once sync.Once
	onceBody := func() {
		fmt.Println("Only once") // 这里可以是数据库连接、初始化等功能
	}
	for i := 0; i < 10; i++ {
		once.Do(onceBody)
	}
}

/*
Only once
*/
```

#### 引用

- 墙裂推荐！[设计模式全套](https://blog.csdn.net/lovelion/article/details/17517213/)

