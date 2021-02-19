### docker存储之storage-driver

------

#### 一、docker存储

docker 为容器提供2种存放数据的资源：（1）由storage driver管理的镜像层和容器层（2）数据卷

容器由最上面的一个可写的容器层，以及若干只读的镜像层组成，容器的数据就存放在这些层中。容器分层结构最大的特性是**Copy-on-Write**：

(1) 新数据会直接存放在最上面的容器层
(2)修改现有数据先从镜像层将数据复制到容器层，修改后的数据直接保存在容器层，镜像层的数据不变
(3)如果多个层中有命名相同的文件，用户只看到最上面那层中的文件

分层结构使得镜像和容器的创建、共享以及分发变得十分的高效，这些都归功于Docker storage driver。storage driver实现了多层数据堆叠并未用户提供一个单一的合并之后的统一视图

Docker支持有多种storage driver，接下来我们来了解一下其中的overlay2。overlay2是在overlay的升级版

> docker info 查询本机的storage driver

![1607052732426](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607052732426.png)

可以看到我机器的storage driver是overlay，本机文件系统是xfs，overlay是在xfs上面创建的。各层的数据存放在/var/lib/docker/overlay2。

> blkid 查看本机磁盘类型以及磁盘个数 

![1607052637571](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607052637571.png)

可以看到本机磁盘分区有2种，类型分别是xfs和iso9660。docker的数据也都是存储在/dev/vda1磁盘中

> 查看docker所在磁盘的使用情况

![1607063451275](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607063451275.png)

查看磁盘的占用情况，可以看到是overlay在/dev/vda1磁盘空间下

![1607330098293](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607330098293.png)



#### 二、overlay2

如下图是overlay2的结构，主要分为2层。一个是upperdir文件系统和lowerdir文件系统，分别是docker的容器层和镜像层。镜像层属于只读层，容器层是可写层，展现给用户看到的是merged层

![1607327917443](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607327917443.png)

进入/var/lib/docker/overlay （如下图）查看某一个overlay层可以看到有lower-id，merged，upper，work

其中lower-id文件保存了当前容器依赖镜像的最上层的UUID。upper文件就是容器的读写层，对容器的修改都保存在这个文件夹里。merged文件夹就是容器文件系统的挂载点，容器通过它提供给客户统一视角，对容器的任何修改在这个文件夹会体现。work是工作目录，用来支持coW。

![1607328290817](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607328290817.png)

> docker inspect containersId 查看容器与这些层的关系。

![1607328497601](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607328497601.png)

overlay的读写操作

**读文件**：
(1) 要读的文件不在container layer中：那就从lowerdir中读，会耗费一点性能；
(2) 要读的文件之存在于container layer中：直接从upperdir中读；
(3) 要读的文件在container layer和image layer中都存在：从upperdir中读文件；

**修改文件**
(1) 第一次修改一个文件的内容：第一次修改时，文件不在container layer(upperdir)中，overlay driver调用`copy-up`操作将文件从lowerdir读到upperdir中，然后对文件的副本做出修改。
需要说明的是，overlay的copy-up操作工作在文件层面，不是块层面，这意味着对文件的修改需要将整个文件拷贝到upperdir中。

下面两个事实使这一操作的开销很小：
`copy-up`操作仅发生在文件第一次被修改时，此后对文件的读写都直接在`upperdir`中进行；
overlayfs中仅有两层，这使得文件的查找效率很高(相对于aufs)



#### 三、案例分析——overlay2所占磁盘空间大

##### 1、overlay2所占磁盘空间大小分析

（1）问题分析与描述

> 占用磁盘空间大是docker经常遇到的一个问题

了解到docker是以文件形式存储在磁盘中，所以在基础的镜像以及容器会占用一定的磁盘空间。overlay2所占的磁盘空间也就是docker所在的磁盘空间

```shell
# 查询docker所在的磁盘
df -h /var/lib/docker
```

![1607063451275](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607063451275.png)

```shell
# 查看docker占用的磁盘大小
du -sh /var/lib/docker
```

![1607061981626](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607061981626.png)

```shell
# docker的磁盘使用
docker system df 
```

![1607063960073](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607063960073.png)

（2）是什么导致overlay2所在磁盘一直增长

在docker中，容器、镜像、数据卷会占用磁盘空间，除此之外还有容器的日志文件。有些容器会一直打印日志，这会使得日志文件不断增加，使得磁盘空间一直增长。

本机使用的是json-file日志驱动。json-file 日志存储的日志路径为：
/var/lib/docker/containers/container_id/container_id-json.log

在docker中json-file的日志文件默认大小是无限制的，随着长时间业务的执行会使得日志文件很大，占满磁盘空间。会导致系统无法正常运行

##### 2、解决方案

（1）清理磁盘，删除关闭的容器，无用的数据卷、以及挂起的镜像(无TAG的镜像)。

```shell
docker system prune 
```

（2） 清理磁盘，删除关闭的容器，无用的数据卷、挂起的镜像、以及没有使用的镜像**(慎用)**

```
docker system prune -a 
```

（3）删除容器打印的所有日志，日志文件占比很大

```
cd /var/lib/docker/containerId
cat /dev/null > *-json.log 
```

（4）限制日志的大小

```powershell
# sudo 权限下
cd /etc/docker/
vi daemon.json
# 增加日志的大小的限制
# 这里的单位可以是k,m,g
{
    "log-driver":"json-file",
    "log-opts": {"max-size":"10m"}
}
```

此时，daemon文件的配置对**于新建的容器才可以生效**。需要执行以下操作：

```shell
# 加载daemon配置
systemctl daemon-reload 
# 重启docker，使得daemon配置生效
systemctl restart docker
```

**查看新容器的log配置是否生效**：

```shell
docker inspect -f '{{.HostConfig.LogConfig}}' containersId
```

例如：可以看到配置项里面已经有日志大小的限制

![1607071777467](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607071777467.png)
























