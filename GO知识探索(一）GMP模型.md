### GO知识探索(一）GMP模型

------

一、GMP说明

二、源码分析

```

```

三、GMP的状态码











































1、 其实在M对应的P中以及全局的sched中均维护了2个队列，一个是待运行goroutine队列runq，一个是空闲goroutine队列gFree。

![1605689957533](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1605689957533.png)





