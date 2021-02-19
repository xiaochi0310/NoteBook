### Docker-overlay2

------

#### 一、/var/lib/docker/overlay2里面装的是什么

可以看到每个容器里面

![1604134977384](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604134977384.png)

merged存放的是容器的可读可写文件，diff存放的是容器可读文件。可以看到diff文件是merged的子文件

![1604135951304](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604135951304.png)

可以看到docker初始内存占用：**overlay2占用24G**

```shell
[root@wunaichi docker]# du -sh *
16K	builder
56K	buildkit
1.4M	containerd
462M	containers
36M	image
172K	network
24G	overlay2
0	plugins
0	runtimes
0	swarm
0	tmp
0	trust
213M	volumes
```

在网上查了各种资料来降低内存占用：

1、docker system prune // 清理磁盘，删除关闭的容器，无用的数据卷、以及挂起的镜像(无TAG的镜像)。**overylay2占用22G,少了2G**

```shell
[root@wunaichi docker]# du -sh *          
16K	builder
56K	buildkit
1.4M	containerd
471M	containers
31M	image
172K	network
22G	overlay2
0	plugins
0	runtimes
0	swarm
0	tmp
0	trust
37M	volumes
```

2、docker system prune -a  // 清理磁盘，删除关闭的容器，无用的数据卷、挂起的镜像、以及没有使用的镜像

**overylay2占用14G,少了8G**  这个操作需要慎重，比上一个操作多删除了没有使用的镜像

```shell
[root@wunaichi docker]# du -sh *       
16K	builder
56K	buildkit
1.4M	containerd
462M	containers
15M	image
172K	network
14G	overlay2
0	plugins
0	runtimes
0	swarm
0	tmp
0	trust
37M	volumes
```

3、cat /dev/null > *-json.log   // 删除容器打印的所有日志，这个日志文件占比很大

overylay2占用14G,减少的微乎其微

```shell
[root@wunaichi docker]# du -sh * 
16K	builder
56K	buildkit
1.4M	containerd
26M	containers
15M	image
172K	network
14G	overlay2
0	plugins
0	runtimes
0	swarm
0	tmp
0	trust
37M	volumes
```

删除的操作：

![1604137016193](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604137016193.png)

在后期可以约束日志文件的大小：

```shell
#  新建或修改/etc/docker/daemon.json，添加log-dirver和log-opts参数

{
   "log-driver":"json-file",
   "log-opts": {"max-size":"10m", "max-file":"1"}
}
```

4、systemctl restart docker  // 删除完重启docker

overylay2占用12G,减少2G

```shell
[root@wunaichi docker]# du -sh * 
16K	builder
56K	buildkit
1.4M	containerd
27M	containers
16M	image
172K	network
12G	overlay2
0	plugins
0	runtimes
0	swarm
0	tmp
0	trust
37M	volumes
```

#### 二、Docker-copmose安装了怎么还是不能执行

安装流程：

```shell
sudo curl -L "https://github.com/docker/compose/releases/download/1.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

但是docker-compose version还是出现命令找不到

![1604287118310](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1604287118310.png)

执行以下命令即可解决

```shell
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```

