从一个Go案例来了解僵尸进程

------

本文从一个Golang案例学习僵尸进程:skull:的主要内容

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
	argv := []string{name,"-c",`sleep 10 && echo "child: exit"`}
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
	fmt.Printf("parent pid %d,ppid%d,egid%d,euid%d,uid%d",os.Getpid(),os.Getppid(),os.Getegid(),os.Geteuid(),os.Getuid())
	fmt.Printf("parent pid is: %d,chile pid is: %d",os.Getpid(),pro.Pid)
    // 主进程睡20s
	time.Sleep(20*time.Second)
	fmt.Println("parent out!!")
	// 清理子进程
	processState,err := pro.Wait()
	if err!=nil {
		panic(err)
	}
	fmt.Println(processState.Pid(),processState.String())
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

![1613976313237](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1613976313237.png)

从上面的打印看出：主进程PID：23380，子进程PID：23385；子进程10s退出，但此时主进程是存活状态，所以该子进程是僵尸进程状态；主进程睡眠20s后清扫僵尸子进程(子进程彻底退出)

同时我们查看进程的相关信息，打印进程的状态、父进程、子进程以及运行命令。分析如下：

1、主进程与子进程同时存活

23380[主进程]状态是S(TASK_INTERRUPTIBLE) 可中断的睡眠状态，因为在睡眠，等到时钟信号量
23385[子进程]状态是S(TASK_INTERRUPTIBLE) 可中断的睡眠状态，因为在睡眠，等到时钟信号量

```bash
[centos@wunaichi ~]$ ps -A ostat,ppid,pid,cmd|grep 23380
Sl+  23325 23380 /tmp/go-build733946445/b001/exe/test
S+   23380 23385 /bin/bash -c sleep 60 && echo "child: exit"
S+   16325 23412 grep --color=auto 23380
```

2、子进程退出

10s后子进程退出，再查看进程相关信息

23380[主进程]状态是S(TASK_INTERRUPTIBLE) 可中断的睡眠状态，因为在睡眠，等到时钟信号量
23385[子进程]状态是Z(TASK_DEAD - EXIT_ZOMBIE)退出状态，进程成为僵尸进程，描述符保存

```bash
[centos@wunaichi ~]$ ps -A ostat,ppid,pid,cmd|grep 23380
Sl+  23325 23380 /tmp/go-build733946445/b001/exe/test
Z+   23380 23385 [bash] <defunct>
S+   16325 23547 grep --color=auto 23380
```

查看机器所以的僵尸进程(查看机器所有的僵尸进程)

```bash
[centos@wunaichi ~]$ ps aux |grep Z
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
centos   23385  0.0  0.0      0     0 pts/3    Z+   06:39   0:00 [bash] <defunct>
centos   23572  0.0  0.0 112712   964 pts/2    S+   06:41   0:00 grep --color=auto Z
```

3、清扫僵尸进程

主进程23380和子进程23385全部退出

```bash
[centos@wunaichi ~]$ ps -A ostat,ppid,pid,cmd|grep 23380
S+   16325 23879 grep --color=auto 23380
```

主进程清扫僵尸子进程，查看僵尸子进程已经不存在

```bash
[centos@wunaichi ~]$ ps aux |grep Z
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
centos   23572  0.0  0.0 112712   964 pts/2    S+   06:41   0:00 grep --color=auto Z
```

##### 三、僵尸进程

到这里，我想大家对僵尸进程已经有一定的概念。**一个进程创建子进程，如果子进程退出，而父进程并没有调用wait或waitpid获取子进程的状态信息，那么子进程的进程描述符仍然保存在系统中。这种进程称之为僵死进程**，**也就是僵尸进程**。

在这个退出过程中，僵尸进程占有的所有资源将被回收，除了进程描述符、进程的退出码、以及一些统计信息以外[这些信息将存储在一个结构体中task_struct]。于是该进程几乎剩下这么个空壳，故称为僵尸。之所以还保留一些僵尸进程的基础信息是因为父进程很可能会关心这些信息。当然，内核也可以将这些信息保存在别的地方，而将task_struct结构释放掉，以节省一些空间。但是使用task_struct结构更为方便，因为在内核中已经建立了从pid到task_struct查找关系，还有进程间的父子关系。

另外一种情况是如果父进程在子进程结束之前退出，则子进程将由init接管。init将会以父进程的身份对僵尸状态的子进程进行处理。所以任何进程都会经过僵尸的这个状态

但是也会产生一些问题：如果进程不调用wait / waitpid的话， 那么保留的那段信息就不会释放，其进程号就会一直被占用，但是系统所能使用的进程号是有限的，如果大量的产生僵死进程，将因为没有可用的进程号而导致系统不能产生新的进程。

所以如果产生了大量的僵尸进程，我们可以将其父进程kill，这样的话僵尸进程都会交给init进程处理。当然，不是所有的进程都必须kill，这要根据你的业务而定了。

下一节我们来了解孤儿进程:stuck_out_tongue_closed_eyes:

##### 引用

[僵尸进程和孤儿进程的Golang实现](https://blog.csdn.net/qq_27068845/article/details/78816995)

[Linux进程状态解析](https://www.cnblogs.com/YDDMAX/p/4979878.html)

