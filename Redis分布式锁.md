### Redis分布式锁

------

#### 一、分布式锁与其他锁的区别

**锁**：我理解锁相当于对某个资源设定标记(可唯一标识)，且这个标记多个抢占方都可以看到。如果资源有这个标记(加锁)，说明该资源暂时被抢占，需要等待没有标记时(释放锁)再使用该资源

**线程锁**：主要给方法和代码块加锁。在多个线程都要访问共享资源的时候，会只允许一个线程进程访问，待资源空闲后，会再被其他线程占用。例如go语言sync包里提供了互斥锁Mutex和读写锁RWMutex用于处理并发过程多个goroutine或者线程读写同一个变量的时候。对于多个线程是在进程中共享内存的，所以设定的标记实在内存中的

**进程锁**：在多个进程访问临界资源的时候，就要想哪写位置是多个进程都能访问到的，在这里我们才能加锁操作。不如回想一下操作系统中进程间的通信来解决临界资源的抢占问题，最常见的就是使用信号量。

> P-V操作 
>
> P 检查信号量的大小，若小于0则阻塞，返回到等到队列；否则申请资源，信号量-1
>
> V 释放资源，信号量+1，若有等到进程，则该进程会被唤醒

在go语言我们使用文件锁(fileLock)解决多个进程读写一个文件的情况。当一个进行抢占到文件，会给这个文件锁定，除了当前进程访问外，不能被其他进程访问。这里可以看到使用的是syscall系统信号量

```go
//加锁
func (l *FileLock) Lock() error {
	f, err := os.OpenFile(l.filePath, os.O_CREATE|os.O_RDONLY, 0666)
	if err != nil {
		return err
	}
	l.f = f
	err = syscall.Flock(int(f.Fd()), syscall.LOCK_EX)
	if err != nil {
		return err
	}
	return nil
}

//释放锁
func (l *FileLock) Unlock() error {
	defer l.f.Close()
	return syscall.Flock(int(l.f.Fd()), syscall.LOCK_UN)
}
```

**分布式锁**：多个机器要访问某一个临界资源，还要对这一资源进行做标记(加锁)还得让每个机器都能访问到。在go语言中常使用的是基于Redis缓存来实现，基于Zookeeper协调系统来实现，以及基于etcd等等。目前应该redis的相比之下是实现起来比较简单，本文主要介绍redis分布式锁实现的原理

#### 二、Redis分布式锁实现的原理

##### 1、测试场景以及结果

首先可以看到我做的测试的redis锁的情况，2台机器(188,173)对NFS中的csv文件进行读写。

场景一：使用文件锁以及线程锁会出现乱码(如图)的情况，说明进程锁/线程锁不能满足我们需要的功能。

![1604025379183](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604025379183.png)

**![1604025547980](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604025547980.png)**

场景二：使用redis分布式锁。每个节点开启10个协程，每个协程读写5K数据，不会出现乱码。具体的执行情况可以看到下面的图，每个节点的goroutine等待这锁资源，有条不紊的读写文件。

![1604025746653](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604025746653.png)



##### 2、redis锁代码实现(golang)

1>  部署redis

​	这里要墙裂推荐Docker部署，源码部署遇到了很多坑。

```shell
# 拉取镜像
docker pull redis 
# 查看镜像
docker images 
# 挂载运行
docker run -p 6379:6379 --name myredis -v /usr/local/docker/redis.conf:/etc/redis/redis.conf -v /usr/local/docker/data:/data -d redis redis-server /etc/redis/redis.conf --appendonly yes
```

> 这里注意如果使得多个节点都能访问redis资源，就必须修改redis的配置文件

redis.conf 修改几个变量

```shell
bind 127.0.0.1 #注释掉这部分，这是限制redis只能本地访问
protected-mode no #默认yes，开启保护模式，限制为本地访问
daemonize no #默认no，改为yes意为以守护进程方式启动，可后台运行，除非kill进程，改为yes会使配置文件方式启动redis失败
```

2> 连接redis服务器

```go
// 使用的redis包
"github.com/go-redis/redis"

// 本地访问addr:127.0.0.1
// 远端访问addr:redis部署机器的ip
func connRedis(addr, password string) *redis.Client {
	conf := redis.Options{
		Addr:     addr,
		Password: password,
	}
	return redis.NewClient(&conf) // 调用库
}
```

3> 加锁

```go
func (r *redisClient) lock(value string) (error, bool) {
    // 关键点SetNX(原子操作): key,V,过期时间 
	ret := r.SetNX("hello", value, time.Minute*10)
	if err := ret.Err(); err != nil {
		fmt.Printf("set value %s error: %v\n", value, err)
		return err, false
	}
	return nil, ret.Val()
}
```

这里使用的是**set ex nx**，这里设置K-V值，以及过期时间都是原子性的，只在KEY不存在的情况下才会SET成功。

如下图，为加锁的流程

![1604032470317](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604032470317.png)

4> 解锁

```go
func (r *redisClient) unlock() bool {
    // 关键点，删除K值对应的信息
	ret := r.Del("hello")
	if err := ret.Err(); err != nil {
		fmt.Println("unlock error: ", err)
		return false
	}
	return true
}
```

5> 休息一会再抢占

开始没抢到会休息1s(自己配置)再继续抢占

```go
func (r *redisClient) retryLock(threadId int) bool {
	ok := false
	for !ok {
		err, t := r.getTTL() // 获取过期时间
		if err != nil {
			return false
		}
		if t > 0 {
			fmt.Print(time.Now().Format("2006-01-02 15:04:05"))
			fmt.Printf("线程 %d 锁被抢占, %f 秒后重试...\n", threadId, (t / 600).Seconds())
			time.Sleep(t / 600) // 睡1s，这里是因为过期时间设定的是10min
		}
		err, ok = r.lock("Jan") // 再次获取锁
		if err != nil {
			return false
		}
	}
	return ok
}
```

#### 三、遇到的坑

1、2个机器，其中一个机器没抢过另外一个机器，只能傻等。然后好不容抢到了，在释放锁之前，锁资源过期了，导致死锁的情况，系统强制结束进程。这很明显不符合我们的预期

![1604027266426](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604027266426.png)

解决办法：在上述场景中我设置锁的资源是10s中，读写数据量较大，很容易锁时间到期。所以后面改成1分钟解决该问题。**所以，锁资源的过期时间要符合业务要求**。但是设置过大时，当某一个机器死掉的时候，使得释放锁的时间过长，其他节点等待时间过长，资源浪费。所以我们要谨慎设定锁资源的过期时间