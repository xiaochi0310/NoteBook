浅析kubernetes的网络原理(CNI-weave)

------

##### 一、kubernetes网络基础

​		Kubernetes网络模型设计的一个基础原则是：单Pod单IP模型。该模型的目标是为了每个Pod分配一个Kubernetes集群私有网络地址段的IP地址。通过该IP，Pod能够跨网络与其他Pod，物理机，容器等进行通信。一个Pod内部的所有容器共享一个网络堆栈（相当于一个网络命名空间，它们的IP地址、网络设备、配置等都是共享的），彼此之间通过localhost通信，就像在一个机器上一样。所以可以将Pod简单看成一个独立的虚拟机。   		        

​        Kubernetes为每个pod分配IP地址都在一个非NAT(网络地址转换)的扁平网络地址空间中，这一点非常重要，因为NAT将网络地址空间分段的做法，会引入了额外的复杂性。当然Pod的IP是不固定的，通常添加service资源进行对Pod的访问([在这篇已经解释过](https://juejin.cn/post/6932365595821801480))

Kubernetes为每个pod分配IP地址都在一个非NAT(网络地址转换)的扁平网络地址空间中，这一点非常重要，因为NAT将网络地址空间分段的做法，会引入了额外的复杂性。当然Pod的IP是不固定的，通常添加service资源进行对Pod的访问([在这篇已经解释过](https://juejin.cn/post/6932365595821801480))

​		现在网络模型有了，怎么实现呢?
​        为了保证网络方案的标准化、扩展性和灵活性，k8s采用CNI(Container Networking Interface)规范。目前已经有多种网络方案，比如Flannel、Calico、Canal、Weave Net等。本文是使用Weave为例。

##### 二、Weave Net 原理

weave创建的虚拟网络可以将部署在多个机器的容器连接起来。对容器来说，weave就像一个巨大的以太网交换机，所有容器都会接入这个交换机，容器可以直接通信，无需NAT和端口映射。

安装weave

```bash
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

环境介绍：2台slave。每台均有一个Pod资源，一个Pod里面创建2个容器。如下图，我们稍后会对本图进行更详尽的解释。本节只学习weave网络的基本原理。

![1614740538836](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614740538836.png)



安装好的weave Pod资源会在kube-system的ns中，每一个Pod运行2个容器

![1614741940946](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614741940946.png)

- weave：主程序，负责建立weave网络，收发数据，提供DNS服务。

- weave-npc：Network policy controller。weave-npc使用iptables来生效network policy，控制接入输出。

安装weave网络插件之后，会自动生成这些网卡：weave、vethwe-datapath@vethwe-bridge、vethwe-bridge@vethwe-datapath，vxlan-6784。

weave 网络包含两个虚拟交换机(见第一张图)： weave 和 datapath， vethwe-bridge 和 vethwe-datapath 将二者连接在一起。weave 和 datapath 分工不同，weave 负责将容器接入 weave 网络，datapath 负责在主机间VxLAN 隧道中并收发数据，datapath 使Weave Net路由器能够告知内核如何处理数据包



回看第一张图，会有很多网卡，我们一一来介绍。

- Container内部eth0：eth0是容器主机的默认网络，主要提供容器访问外网所提供的服务，走的默认docker网络架构，只不过他创建了docker_gwbridge这个网桥。
- docker_gwbridge是容器所创建的网桥它替代了docker0的服务。
- Container ethwe：它是veth pair虚拟设备对，与其他容器通信的网络虚拟网卡。
- vethwe-bridge：是ethwe设备对另外一段在创建的weave网桥上。网桥内分配的具体的IP与网关。
- weave：weave网桥，通过route路由表找到目标，通过端口将数据包转发到对端端口节点。
- eth0：机器网卡与外界网卡连接得真机网卡，它用来转发，容器VXLAN与NAT两种网卡类型的数据包到指定的对端节点。
- weave会将相邻的节点互相学习，通过route路由表进行相互通信，并通过单独的端口发送数据。类似于静态路由。

在node1节点，有一个pod（service-test-958ccb545-kg5wn），里面运行2个容器，这里说明的是每一个Pod都会运行额外的pause容器。在k8s中，用pause容器来作为一个pod中所有容器的父容器。这个pause容器有两个核心的功能，第一，它提供整个pod命名空间的基础，例如网络。第二，启用PID命名空间，它在每个pod中都作为PID为1进程，回收僵尸进程。所以Pod的网络依靠对应的pause容器的配置

1、查看Pod的IP(10.44.0.5)

![1614743446911](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614743446911.png)

2、查看Pod的网卡，也即pause容器的网卡

进入pause容器(21276为容器的PID)，查看网卡对。eth0是pause容器内部

![1614677757352](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614677757352.png)

全局网卡 ip a

![1614750753167](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614750753167.png)

也就是eth0-vethwepl6811277是网卡对。这里vethwepl6811277是Pod另一端连接weave网桥的一个网卡。

3、weave网桥还挂着vethwe-bridge，这是什么

weave网桥ip范围

![1614736076923](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614736076923.png)

ip -d link打印详细网卡

![1614753669631](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614753669631.png)

可以看到vethwe-bridge与vethwe-datapath是一对网卡对

vethwe-datapath挂在master为datapath上(datapath是一个交换机)

vxlan-6784是vxlan的网卡，weave主机间是通过VxLAN进行通信的。如下图，即是weva简单的架构图

![1614754189081](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614754189081.png)

这样容器的网络为：

- 所有容器都连接到weave网桥
- weave网桥通过veth pair连到内核的openvswitch模块
- 跨主机容器通过openvswitch vxlan通信

##### 三、kubernetes网络通信

###### 1、容器到容器之间的直接通信

同一个Pod内的容器（Pod内的容器是不会跨宿主机的）共享同一个网络命名空间，共享同一个Linux协议栈。所以对于网络的各类操作，就和它们在同一台机器上一样，它们甚至可以用localhost地址访问彼此的端口。例如，如果容器2运行的是MySQL，那么容器1使用localhost:3306就能直接访问这个运行在容器2上的MySQL了

###### 2、抽象的Pod到Pod直接的通信

1）同一台机器上Pod通信（下图来源权威指南）

![1614755858389](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614755858389.png)

​		Pod1和Pod2都是通过Veth连接到同一个docker0网桥上的，它们的IP地址IP1、IP2都是从docker0的网段上动态获取的，它们和网桥本身的IP3是同一个网段的。另外，在Pod1、Pod2的Linux协议栈上，默认路由都是docker0的地址，也就是说所有非本地地址的网络数据，都会被默认发送到docker0网桥上，由docker0网桥直接中转。综上所述，由于它们都关联在同一个docker0网桥上，地址段相同，所以它们之间是能直接通信的

2）不同机器上Pod通信

红线表示数据流向

node1的pod与node2的pod进行通信

![1614757273549](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614757273549.png)

- eth0 会将数据包发送给vethwe-bridge网桥。
- vethwe-bridge接收到数据包后由weave去处理这个数据，通过UDP6784数据端口依照weave的路由表转发到下一路由节点。
- 如果该节点就是目的地，本地weave会把信息转发到内核的TCP协议站，再转发到目的节点。

###### 3、Pod到service之间通信

使用iptables。详见[在这篇已经解释过](https://juejin.cn/post/6932365595821801480)

###### 4、集群外部与内部组件之间的通信

[在这篇已经解释过](https://juejin.cn/post/6932365595821801480)



引用

- 《容器与容器云》

- 《kubernetes权威指南》

- 《每天5分钟玩转kubernetes》

- [weave介绍](https://kubernetes.feisky.xyz/extension/network/weave)

