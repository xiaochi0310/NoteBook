零基础了解Docker架构

本文会讲到docker的整体框架和基本原理。以及最主要的镜像(是什么，怎么构建)，容器(是什么，怎么运行)，在最后说一下容器都有哪些优点。容器基础网络已经在之前的文章分享过，有兴趣的可以查看。https://juejin.cn/post/6904201044390051848



#### 一、docker 框架与底层技术

##### 1、docker 框架

Docker采用的是C/S架构，客户端向服务端发送请求，服务器负责构建、运行和分发容器。这里最常用的Docker客户端就是docker命令。服务端即是Docker daemon，其运行在宿主机上，负责创建、运行、监控容器和构建存储镜像。我们经常改完Docker的配置之后，会执行systemctl daemon-reload，这就是重启服务的作用。下图是查看docker服务。

![1610089694391](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610089694391.png)

##### 2、容器的底层技术

1> cgroup 实现资源限额

cgroup 即是control group。linux 通过cgroup 限制进程的资源，我们对容器CPU/IO/MEM限额实际就是在配置cgroup（容器一节会详细介绍怎么限制）。

容器的资源配置都在/sys/fs/cgroup/路径下可以找到。如下图标注的即是对IO\CPU\MEM限额的配置

![1610074565477](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610074565477.png)

现在以cpu为例查看某一容器的配置在哪？

在cpu目录下包含docker独立的文件夹，进入docker文件夹就会有各个容器的ID

![1610074671832](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610074671832.png)

接下来进入某个容器，查看cpu配额，是默认值1024

![1610074793493](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610074793493.png)

2> namespace 实现资源隔离

linux使用namespace让每一个容器看起来像一个独立的计算机，也就是ns实现了容器间的资源隔离。

linux主要包含6种ns，分别是6种资源：Mount、UTS、IPC、PID、Network和User

1、Mount让容器自己觉得有整个文件系统。可以在容器内部mount、unmout

2、UTS让容器有自己的hostname。默认是短ID，或者通过-h设置的。

![1610075870751](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610075870751.png)

3、IPC 容器有自己的共享内存和信号量来实现进程间的通信。不会和host和其他容器混在一起

4、独立的PID

所有容器挂在docker进程下

> ps  axf 查看本机所有的进程

![1610076095223](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610076095223.png)

可以看到容器的进程和容器的子进程。

进入容器内部再查看进程ID和host的进程ID又有不同，容器有自己独立的PID，这就是ns提供的功能

![1610076278192](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610076278192.png)

5、Network

让容器有自己独立的网卡和IP等资源，这些在docker网络中介绍过

6、User

在容器可以创建新的用户，但是在宿主机是看不到该用户的。

![1610076487140](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610076487140.png)

#### 二、docker 镜像

##### **1、是什么**

镜像实际是一个文件系统。可以看成是一个只读模板，通过镜像可以创建容器。

最大优点：共享资源。生成一个镜像都是基于一个基础镜像进行构建的，这个基础镜像可以被多个镜像使用。所以一个镜像是一层一层叠加而成的。

对于镜像层只是只读的，对于容器的改动(增、删、改)只会发生在容器层，所以只有容器层才是可写的。容器还有一个特性就是Copy-on-Write，就是在修改容器的时候会拷贝镜像层到容器层进行修改，所有的修改对于镜像层也是不感知的。这样大大增加了容器操作的效率，也维护了镜像层可以作为其他容器的base的特性

##### 2、如何构建镜像

通常都是使用Dockerfile构建镜像。

以下是Dockerfile关键字的说明，现在以我写的测试程序编写的Dockerfile文件为例进行说明。

```dockerfile
#
# Build stage
#
# FROM 指定base镜像
FROM golang:latest AS build
# WORKDIR 为后面的RUN\CMD\ENTRYPOINT\ADD\COPY 指令设置镜像中的当前工作目录
WORKDIR /home/centos/gopath/src/blockfilecheck
# ENV 设置环境变量
ENV GO111MODULE off
ENV GOPATH /home/centos/gopath
# COPY src dst 将文件从build context 复制文件到镜像
COPY . .
#RUN 在容器中运行指定的命令
# CGO_ENABLED=1 GOOS=linux 交叉编译，linux系统编译
# go build -ldflags="-s -w"[压缩编译后的体积] -a [完全编译,不产生中间文件]
RUN CGO_ENABLED=1 GOOS=linux go build -ldflags="-s -w" -a -installsuffix cgo -tags=jsoniter -o main /home/centos/gopath/src/blockfilecheck/main.go

#
# Production stage
#
# FROM 指定base镜像
FROM ubuntu:latest
WORKDIR /home/centos/gopath/src/blockfilecheck
# 将默认配置文件添加到对应目录
#ADD ./Shanghai /etc/localtime
COPY --from=build /home/centos/gopath/src/blockfilecheck/ .
# 指定容器中的进程会监听某个端口
EXPOSE 8090
# ENTRYPOINT 设置容器启动时运行的命令
ENTRYPOINT ["./main"]
```

运行写好的Dockerfile文件

```shell
# -t 镜像名字 
# . Dockerfile所在目录，此时是当前目录
docker build -t fileblocktool:latest . 
```

以下是构建镜像的流程

```shell
ending build context to Docker daemon  1.281MB
Step 1/11 : FROM golang:latest AS build
 ---> 05c8f6d2538a
Step 2/11 : WORKDIR /home/centos/gopath/src/blockfilecheck
 ---> 4bb708fbbea5
Removing intermediate container 3319005c0ee2
Step 3/11 : ENV GO111MODULE off
 ---> Running in 73977df3eaf3
 ---> 11fdb6b2f433
Removing intermediate container 73977df3eaf3
Step 4/11 : ENV GOPATH /home/centos/gopath
 ---> Running in e12147ed9e8c
 ---> ff9311d66923
Removing intermediate container e12147ed9e8c
Step 5/11 : COPY . .
 ---> 71534a4e7846
Removing intermediate container 42952bdc6130
Step 6/11 : RUN CGO_ENABLED=1 GOOS=linux go build -ldflags="-s -w" -a -installsuffix cgo -tags=jsoniter -o main /home/centos/gopath/src/blockfilecheck/main.go
 ---> Running in a39a9ea5d13e
 ---> 2dd48f617cb7
Removing intermediate container a39a9ea5d13e
Step 7/11 : FROM ubuntu:latest
 ---> bb0eaf4eee00
Step 8/11 : WORKDIR /home/centos/gopath/src/blockfilecheck
 ---> b9bd32e694b5
Removing intermediate container 1cf1f6d21b4f
Step 9/11 : COPY --from=build /home/centos/gopath/src/blockfilecheck/ .
 ---> c1dbb8e28d10
Removing intermediate container c047a0c978ee
Step 10/11 : EXPOSE 8090
 ---> Running in e38285ae72c7
 ---> d7b4045bb8ac
Removing intermediate container e38285ae72c7
Step 11/11 : ENTRYPOINT ./main
 ---> Running in 4344bd27d796
 ---> 599396d0bcc1
Removing intermediate container 4344bd27d796
Successfully built 599396d0bcc1
Successfully tagged fileblocktool:latest

```

通过构建流程可以看到step都是按照Dockerfile的每一条命令执行的，在golang基础镜像构建我们自己的镜像

通过docker images 查看生成成功镜像

![1610013261381](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610013261381.png)

在Dockerfile有如下三个关键字功能比较相似：

RUN\CMD\ENTEYPOINT的区别

RUN 在当前镜像的顶部执行命令，并创建新的镜像层。

CMD 允许用户指定容器的默认执行的命令，但当docker run 有其他命令时，CMD命令被忽略

ENTEYPOINT 指定要执行的命令和参数，与CMD类似，但是ENTEYPOINT命令不会被忽略（常用）

#### 三、docker 容器

##### **1、是什么**

容器在宿主机实际上是一个进程。Docker容器就是镜像的一个运行实例。

如果镜像是软件生命周期的构建和打包阶段，容器则是启动和运行阶段。

##### **2、如何运行容器**

使用docker run命令

```shell
#  -d 后台执行  --name=容器名字 -v 挂载 宿主机:容器 镜像名字：版本
docker run -d --name=tool -v /etc/www:/var/www tool:latest
```

docker ps 查看宿主机所有的容器

进入某个容器，可以看到就是一个文件系统。

```shell
# -it 在容器启动后直接进入，使用bash进程
docker exec -it <container_ID> bash
# ps -elf 显示容器启动的进程
```

前面说容器是一个进程，那么docker stop  实际上就是向该进程发送一个SIGNTREM信号

docker kill 实际上就是向进程发送SIGNKILL信号

##### 3、资源限制

同样的，容器像虚拟机一样可以对某些资源进行限制。

1、内存限额

容器也可以使用物理内存和swap

```shell
# 内存200m swap 100m
# 若指出-m 没有指出--memory-swap 则默认swap分配与m相同的内存空间
docker run -m 200m --memory-swap=300m tool
```

2、CPU限额

使用-c 限制cpu使用的比例。是一个相对权重值，默认1024

```shell
docker run -c 1024 tool
```

3、磁盘读写限额

可以限制bps和iops

bps byte per second 每秒读写的数据量

iops io per second 没秒IO读写的次

```shell
--device-read-bps:限制读某个设备的bps
--device-write-bps:限制写某个设备的bps
--device-read-iops:限制读某个设备的iops
--device-write-iops:限制写某个设备的iops
```

在运行容器的时候执行，限制写磁盘/dev/sda最大数据量为30MB

```shell
docker run -it --device-write-bps /dev/sda:30MB tool
```

通过dd可以测试容器写磁盘的速度

#### 四、容器有哪些优点

从项目经验分析：（官方还有很多高大上的优点）

1、部署方便

我们在以往部署一个工具可能要下载源码，然后进程运行，才使得这个工具生效，甚至还有各种环境兼容问题。但使用容器，只要拉取镜像，运行容器即可。且容器还提供在运行中出现退出的情况下，自动重启容器。

2、隔离性很好

我们部署多个容器都是一个个独立的个体，各个容器是解耦的。资源也是容器自己独立拥有的，即使一个容器挂了，也不会影响其他的

3、提高开发效率

docker在机器可以轻易让几十个容器起来。不用将过多的时间放在搭建环境上。也有很好的移植性

缺点：之前由于部署的容器一直有问题，就打算删掉重启起一个容器，最后导致数据全部丢失。这也是惨痛的代价。所以要保证删除容器之前需要把使用的数据要迁移好。