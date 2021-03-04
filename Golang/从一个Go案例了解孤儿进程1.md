从一个Go案例了解孤儿进程

------

本文从一个Golang案例学习孤儿进程:skull:的主要内容

[僵尸进程在这](https://juejin.cn/post/6931990188029804552)

##### 一、案例

main进程创建一个bash子进程，子进程存活10s退出，main进程睡20s退出

```go
package main

import (
	"fmt"
	"os"
	"time"
)

func main() {
	name := "/bin/bash" // 要运行的进程
    // 子进程的主要动作  -c 表示后续所有字符串当做完成指令来执行
	argv := []string{name,"-c",`sleep 100 && echo "child: exit"`}
	attr := &os.ProcAttr{
		Dir:   "/tmp",  // 新进程的工作目录
		Env:os.Environ(), // 新进程的环境变量列表
		Files: []*os.File{os.Stdin, os.Stdout, os.Stderr}, // 对应标准输入，标准输出和标准错误输出,若为nil,表示该进程启动时file是关闭的

	}
    // 创建一个子进程
	pro,err := os.StartProcess(name,argv,attr)
	if err!=nil {
		panic(err)
	}
    // 打印相关信息
	fmt.Printf("parent pid is: %d,chile pid is: %d",os.Getpid(),pro.Pid)
    // 主进程睡30s 后退出
	time.Sleep(30*time.Second)
	fmt.Println("parent out!!")
}
```

启动一个子进程，os包自带StartProcess创建子进程 API

```go
func StartProcess(name string, argv []string, attr *ProcAttr) (*Process, error) {
	testlog.Open(name)
	return startProcess(name, argv, attr)
}
// 该函数可以调用或启动外部系统命令和二进制可执行文件；它的第一个参数是要运行的进程，第二个参数用来传递选项或参数，第三个参数是含有系统环境基本信息的结构体
// 函数返回被启动进程的 id（pid），或者启动失败返回错误
```

我们在linux环境下运行该代码

##### 二、结果分析

![1614762356822](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614762356822.png)

从上面的打印看出：主进程PID：13642，子进程PID：13647；主进程30s退出，但此时子进程是存活状态(sleep)，所以该子进程是孤儿进程状态，会由init进程(PID=1)来处理

同时我们查看进程的相关信息，打印进程的状态、父进程、子进程以及运行命令。分析如下：

1、主进程与子进程同时存活

查看子进程13647的状态，可以看代父进程是13642

```bash
[centos@wunaichi ~]$ cd /proc/13647
[centos@xxxx 13647]$ cat status 
Name:	bash
Umask:	0002
State:	S (sleeping)
Tgid:	13647
Ngid:	0
Pid:	13647
PPid:	13642
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
```

2、主进程退出

30s后主进程退出，再查看子进程相关信息。子进程的父进程变成init进程(PID=1)

```bash
[centos@xxxx 13647]$ cat status 
Name:	bash
Umask:	0002
State:	S (sleeping)
Tgid:	13647
Ngid:	0
Pid:	13647
PPid:	1
TracerPid:	0
Uid:	1000	1000	1000	1000
Gid:	1000	1000	1000	1000
```

##### 三、孤儿进程

孤儿进程：一个父进程退出，而它的一个或多个子进程还在运行，那么那些子进程将成为孤儿进程。孤儿进程将被init进程(进程号为1)所收养，并由init进程对它们完成状态收集工作，init 进程会循环地 wait() 它的已经退出的子进程。

孤儿进程结束后会被 init 进程善后，并没有危害。