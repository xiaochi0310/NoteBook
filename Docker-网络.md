Docker-网络

------

#### 一、Docker网络有哪些

学习Docker，会学习到docker网络，一个网络是一组可以相互联通的端点。Docker在安装时就会创建3个网络

```shell
# 查看网络
docker network ls
```

其余2个是自己创建的网络，可以暂时不关注

![1607496812014](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607496812014.png)

**host网络**：与宿主机在同一个网络中，与宿主机共用一个Network Namespace。容器将不会虚拟自己的网卡和IP，而是使用宿主机的IP和端口。若端口冲突就不能使用该模式了

**none网络**：关闭容器的网络，放在上面的容器都是不需要网络的。例如生成随机密码的功能

**bridge网络**：容器使用独立的Network Namespace，并连接到docker0虚拟网桥。如果不指明网络，默认创建的容器都会挂在docker0中。容器通过docker0网桥以及Iptables net表配置与宿主机通信。

#### 二、Bridge网络是什么

Docker derver启动的时候，主机会自动创建docker0的虚拟网桥，所有新创建的容器都会连接到这个网桥之上，这样主机的容器相当于在一个二层网络中。Bridge网络会为每个容器分配veth pair对，网卡的一头在容器中，一头在挂在网桥docker0上。

```shell
# 查看主机的网桥
brctl show
```

可以看到docker0网桥上已经挂载了3个网卡了。这里说明已经创建了三个容器，这三个就是容器的虚拟网卡。

![1607498082077](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607498082077.png)

```shell
# 查看bridge网络的相关信息
docker network inspect bridge
```

可以看到bridge网段是172.17.0.0/16，网关占用第一个ip（172.17.0.1），剩下的ip随机分配给创建的容器

![1607498883104](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607498883104.png)

接下来看一下mysqltest容器的网络配置：

> apt-get install net-tools  // 安装ifconfig
>
> apt-get install iputils-ping  // 安装ping
>
> apt-get install iproute2  // 安装ip

![1607566578209](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607566578209.png)

可以看到为该容器分配的ip是172.17.0.3，容器端网卡是eth0@if1232。

通过ip addr 命令找到另一端网卡vethc991ce1

![1607566894359](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607566894359.png)

通过brctl show 命令看到vethc991ce1挂在docker0上

![1607567057574](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607567057574.png)

我们可以看到mysqltest容器2端有一对veth pair(vethc991ce1和eth0@if1232)，这2端一端(eth0@if1232)在容器中，一端(vethc991ce1)挂在docker0，这相当于eth0@if1232也挂在docker0上，从而将容器连接到bridge网络中

当前的容器网络拓扑图如下：

![1607567689524](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607567689524.png)

#### 三、容器之间是怎么相互通信的

##### 1、主机内部，相同网络

Docker基于veth pair网卡对，实现相同网络之间的通信，不是直接端对端，是通过bridge间接实现通信。

![1607567734591](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607567734591.png)

在mysql2容器内部，ping mysql1的ip，可以看到是可以互通的

![1607565505058](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607565505058.png)

在mysql2容器内部可以看到路由表：接口是eth0，通过网卡将信息传输到docker0，docker0广播地址，通过网卡找到mysql1

![1607501886026](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607501886026.png)

##### 2、主机内部，不同网络

不同网络之间是不能直接进行通信的，也就是在172.17.0.2容器内部不能ping通172.18.0.4。如下图是连接不通的网络拓扑图：

![1607566103434](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607566103434.png)

可以通过以下2种方法实现不同网络直接的通信：

###### 方法一：把容器加入到另一个网络中，把想要通信的容器放在同一容器

也就是将peer0.org2容器加入到bridge

通过ifconfig可以看到peer0.org2容器在之前的网卡基础上(lo,eth0)增加了eth1网卡，且分配了一个bridge网络网段的ip。

```shell
# 将04aec1064812(peer0.org2)容器加入到bridge网络中
[centos@wunaichi-fabric ~]$ docker network connect bridge 04aec1064812
# 进入04aec1064812容器
[centos@wunaichi-fabric ~]$ docker exec -it 04aec1064812 bash
# 查看ip 可以看到在以前的基础上增加了eth1网卡，且分配了一个bridg网段的ip
root@04aec1064812:/opt/gopath/src/github.com/hyperledger/fabric/peer# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.18.0.4  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:ac:12:00:04  txqueuelen 0  (Ethernet)
        RX packets 2487840  bytes 433949085 (413.8 MiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 2455380  bytes 426722690 (406.9 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 172.17.0.5  netmask 255.255.0.0  broadcast 0.0.0.0
        ether 02:42:ac:11:00:05  txqueuelen 0  (Ethernet)
        RX packets 8  bytes 656 (656.0 B)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 0  bytes 0 (0.0 B)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 38  bytes 3151 (3.0 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 38  bytes 3151 (3.0 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0


```

查看bridge网络也可以看到peer0.org2容器

![1607504956208](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607504956208.png)

进入peer0.org2容器可以看到该容器同时属于2个网络中

![1607505281057](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607505281057.png)

这样我们的网络拓扑图会变成这样：

![1607566129875](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607566129875.png)

peer0.org2加入到新网络之后，会产生一对veth pair在bridge网络上，这样就和在同一网络通信是一致的了。

###### 方法二：修改底层iptables实现网络的通信

```shell
# iptables-save 查看iptables规则
[centos@wunaichi-fabric ~]$ sudo iptables-save
...
-A DOCKER-ISOLATION -i br-b86310df8e66 -o br-7a0183b7dfed -j DROP
-A DOCKER-ISOLATION -i br-7a0183b7dfed -o br-b86310df8e66 -j DROP
-A DOCKER-ISOLATION -i docker0 -o br-7a0183b7dfed -j DROP
-A DOCKER-ISOLATION -i br-7a0183b7dfed -o docker0 -j DROP
-A DOCKER-ISOLATION -i docker0 -o br-b86310df8e66 -j DROP
-A DOCKER-ISOLATION -i br-b86310df8e66 -o docker0 -j DROP
...
```

**为什么2个网络不通其实原理就在这里**：在iptables规则就drop掉了网桥docker0和br-b86310df8e66之间的双向流量。规则的命名DOCKER-ISOLATION也可知道docker 在设计上就是隔离了不同的network。

所以这里最直接的办法就是修改iptables规则：增加双向关系

```shell
sudo iptables -I DOCKER-USER -i docker0 -o br-b86310df8e66 -j ACCEPT
sudo iptables -I DOCKER-USER -i br-b86310df8e66 -o docker0 -j ACCEPT
```

验证一下：完美~

![1607507040948](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607507040948.png)

##### 3、外部世界与容器的通信

在容器内ping www.baidu.com 是可以通的

![1607507076378](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607507076378.png)

可见，容器默认就可以访问外网。但是原理是什么呢？

先查一下iptables规则，在NAT表有这么一条规则：

```
-A POSTROUTING -s 172.17.0.0/16 ! -o docker0 -j MASQUERADE
```

这条语句表明：网桥docker0收到来自172.17.0.0/16网段的外出包，把它交给MASQUERADE处理。MASQUERADE的处理方式将包的源地址替换成host的地址发送出去，也就是做了一次网络地址转换(NAT)

查看路由表，可以看到默认路由是通过eth0发送出去

![1607507576839](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607507576839.png)

我们通过tcpdump抓包来看下是怎么转换的

第一步：先让docker0网络的容器ping 百度的网址

![1607508246082](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607508246082.png)

第二步：看到docker0抓包情况

![1607508502472](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607508502472.png)

docker0收到mysqltest的ping包，源地址为容器的ip 172.17.0.3，然后交给MASQUERADE处理。

第三步：看到eth0抓包情况

![1607508511425](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607508511425.png)

在eth0看到把ping包源地址转成10.0.0.234，这就是NAT规则处理的结果，保证数据包到达外网。下图就是以上的流程：

![1607509012229](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1607509012229.png)



看到这里你可能还有疑问，那不同主机之间的容器是怎么通信的，这个留到下次学习~







