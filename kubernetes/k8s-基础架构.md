从一个案例学习k8s 基本架构

前提：你已经对docker、容器、数据存储有了基本的认识，若还没有，请看我相关文章

> [零基础了解docker架构]https://juejin.cn/post/6915296756150304782
> [浅析docker网络原理]https://juejin.cn/post/6904201044390051848
> [docker数据卷的2种方式]https://juejin.cn/post/6916773929512075272

#### 一、基础组件

在服务器我已经使用kubadm搭建的k8s集群(若想深入学习，一定先搭建一套k8s集群哦)，一主(master)一从(slave)。系统创建的Pod都在namespace为kube-system中，我们可以看到k8s集群的都有以下的主要组件：

![1612423669415](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612423669415.png)

master和slave的组件分配如下图

![1612429786814](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612429786814.png)

##### 1、master的k8s主要组件

**kube-apiserver**
客户端组件通过kube-apiserver管理cluster各种资源，kube-apiserver提供了HTTP/HTTPS RESTful API，例如kubectl就算是一个客户端

**kube-controller-manager**
管理cluster的资源，kube-controller-manager由多种controller组成，包括replication controller、namespacecontroller等。
不同的controller管理不同的资源，replication controller管理Deployment、StatefulSet、DaemonSet的生命周期；namespacecontroller管理Namespace资源

**kube-schedule**
负责决定哪个Pod在哪个机器上运行。Scheduler在调度时会充分考虑Cluster的拓扑结构，当前各个节点的负载，以及应用对高可用、性能、数据亲和性的需求

**etcd**
是一个数据库，保存集群的配置信息和各种资源的状态信息。例如，kubectl get pod信息就是从etcd数据库获取的

**weave-net** 
Pod间总是要通信的，weave是Pod的网络的其中一个方案

##### 2、slave的k8s主要组件

**kube-proxy** 

service在逻辑上代表了后端的多个Pod，外界通过service访问Pod。service接收到的请求是如何转发到Pod的呢？这就是kube-proxy要完成的工作。每个Node都会运行kube-proxy服务，它负责将访问service的TCP/UPD数据流转发到后端的容器。如果有多个副本，kube-proxy会实现负载均衡。从图中可以看到master也有kube-proxy，这是因为master也可以作为一个slave来使用。

**kubelet**
是Node的agent，当Scheduler确定在某个Node上运行Pod后，会将Pod的具体配置信息（image、volume等）发送给该节点的kubelet，kubelet根据这些信息创建和运行容器，并向Master报告运行状态。

#### 二、从案例学习k8s整个框架

我们先部署一个案例来看一下k8s组件具体是怎么交互的。

kubernetes是通过各种controller来管理pod的生命周期。为了满足不同业务场景，Kubernetes开发了Deployment、ReplicaSet、DaemonSet、StatefuleSet、Job等多种Controller。我们首先学习最常用的Deployment。当然本文不是来学习controller的种类以及区别，所以我们现在选最常用的deployment控制器来进行案例学习

##### 1、生成一个案例镜像

###### 1）编写代码

写一个简单功能，服务端监听8080端口，访问UIL打印当前时间和一个字符串“This is server”

```go
// server.go
func server(rep http.ResponseWriter,req *http.Request) {
	time := time2.Now()
	rep.Write([]byte(time.String()))
	rep.Write([]byte("This is server"))
}

func main() {
	http.HandleFunc("/util",server)
	http.ListenAndServe(":8080",nil)
}
```

###### 2）生成镜像

本地生成镜像，这里是用Dockerfile文件生成镜像的。Dockerfile字段的说明在此文已经详细描述[零基础了解Docker的基础架构]https://juejin.cn/post/6915296756150304782

```bash
# Dockerfile
FROM golang:latest AS build
WORKDIR /go/src/service
ENV GOPROXY https://goproxy.cn
ENV GO111MODULE off
COPY . .
RUN CGO_ENABLED=1 GOOS=linux go build -ldflags="-s -w" -a -installsuffix cgo -tags=jsoniter -o main server.go

#
# Production stage
#
FROM ubuntu:latest
WORKDIR /go/src/service
COPY --from=build /go/src/service/main .
# 与代码的端口保持一致
EXPOSE 8080
ENTRYPOINT ["./main"]

```

生成镜像，build当前路径下的Dockerfile文件

```bash
docker build -t service_test:latest .
```

查看生成的镜像，这个镜像会在后面的deployment.yaml使用

![1612430670215](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612430670215.png)

##### 2、使用deployment控制器创建Pod

###### 1）编写namespce.yaml 

创建一个新的namespace,名为k8s-test

```bash
apiVersion: v1
kind: Namespace
metadata:
    name: k8s-test
```

创建ns

``` 
kubevtl create -f  namespce.yaml
```

###### 2）编写service.yaml

映射对外端口(8080->30000)

```bash
apiVersion: v1
kind: Service
metadata:
  # 与创建的ns保持一致
  namespace: k8s-test
  name: service
spec:
  ports:
    - name: "service-port"
      targetPort: 8080
      port: 8080
      nodePort: 30000
  # 这里selector的app与下面的deployment要一致
  selector:
    app: service-test
  type: NodePort
```

创建service

```bash
kubevtl create -f  service.yaml
```

###### 3）编写deployment.yaml

```bash
apiVersion: apps/v1  # 当前格式的版本
#  创建的控制器资源类型，这里是Deployment，其他资源是其他的字段StatefuleSet、DaemonSet等
kind: Deployment 
metadata:
  name: service-test
  namespace: k8s-test
# 规格说明,Pod描述
spec:
  # 副本数量
  replicas: 2
  # Pod的更新策略：1、Recreate(杀掉正在运行的Pod，然后创建新的) 
  # 2、RollingUpdate，滚动更新，即逐渐减少旧Pod的同时逐渐增加新Pod
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: service-test
  # Pod模板
  template:
    metadata:
      labels:
        app: service-test
    # pod的规格
    spec:
      hostname: service-test
      # 容器镜像，是我们本地创建好的容器
      containers:
        - image: service_test:latest
          name: service-test
# imagePullPolicy 可选字段：Never(只从本地拉镜像)/IfNotPresent(如果本地不存在则拉取仓库中) /Always(希望每次都拉取最新的镜像) 
          imagePullPolicy: Never
          # 端口与代码一致
          ports:
            - containerPort: 8080
      # restartPolicy重启策略；Always(只要退出就重启) OnFailure(失败退出重启) Never(不重启)
      restartPolicy: Always
```

创建pod

```bash
kubectl create -f  deployment.yaml
```

查看创建的pod

![1612430952109](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612430952109.png)

在机器上访问对外端口30000，是功能正常的。使用deployment部署pod完成

![1612431366759](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612431366759.png)



#### 三、分析流程

##### 1、创建Pod的主要流程

我们结合上面的例子回看文章开始的图，通过此图和案例学习各个组件的主要职能

![1612434052778](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612434052778.png)

① kubectl 向k8s-master的apiserver发出部署请求
② apiserver通知controller manager创建一个deployment资源
③ controller创建好之后通知schedule调度
④ schedule执行调度任务，决定调度在哪个节点上后会将pod的配置信息包括镜像、volume、副本集通知到k8s-slave的kubelet
⑤ kubectl在自己的node上创建并运行pod副本、容器相关

注意这里没有涉及到kube-proxy，是没有创建service，没有对外的请求访问pod
应用配置以及状态信息保存在etcd中，执行kubectl get命令的时候，会向etcd读取信息

##### 2、分析deployment各种资源

先看以下三条命令

```bash
# 获取deployment资源,可以看到2个副本正常的运行(名字为service-test)
[centos@wunaichi ~]$ kubectl get deployment -n k8s-test  
NAME           READY   UP-TO-DATE   AVAILABLE   AGE
service-test   2/2     2            2           18h

# kubectl describe deployment了解更详细的信息
[centos@wunaichi ~]$ kubectl describe deployment -nk8s-test
Name:               service-test
Namespace:          k8s-test
CreationTimestamp:  Thu, 04 Feb 2021 07:05:43 +0000
Labels:             <none>
Annotations:        deployment.kubernetes.io/revision: 1
Selector:           app=service-test
Replicas:           2 desired | 2 updated | 2 total | 2 available | 0 unavailable
StrategyType:       Recreate
MinReadySeconds:    0
Pod Template:
  Labels:  app=service-test
  Containers:
   service-test:
    Image:        service_test:latest
    Port:         8080/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   service-test-958ccb545 (2/2 replicas created)
Events:          <none>

 # 我们创建了一个replicaset(service-test-958ccb545)
 # replicaset是由deployment创建的，deployment是通过replicaset管理pod
[centos@wunaichi ~]$ kubectl get replicaset -n k8s-test 
NAME                     DESIRED   CURRENT   READY   AGE
service-test-958ccb545   2         2         2       18h

# 获取创建的pod
[centos@wunaichi ~]$ kubectl get pod -n  k8s-test  
NAME                           READY   STATUS    RESTARTS   AGE
service-test-958ccb545-b78xl   1/1     Running   0          18h
service-test-958ccb545-scz2c   1/1     Running   0          18h

# 查看pod信息的详情
[centos@wunaichi ~]$ kubectl describe pod service-test-958ccb545-b78xl -n k8s-test
Name:               service-test-958ccb545-b78xl
Namespace:          k8s-test
Priority:           0
PriorityClassName:  <none>
Node:               wunaichi.novalocal/10.0.0.173
Start Time:         Thu, 04 Feb 2021 07:05:43 +0000
Labels:             app=service-test
                    pod-template-hash=958ccb545
Annotations:        <none>
Status:             Running
IP:                 10.44.0.4
Controlled By:      ReplicaSet/service-test-958ccb545  # 通过ReplicaSet管理
Containers:
  service-test:
    Container ID:   docker://b7f0319de468d1886616faf6896b01c9abbb593a6d383343514cc65e8d07c99b
    Image:          service_test:latest
    Image ID:       docker://sha256:8c6afd1695977500af6c5efebb4b4176f4bf80bf9b0f850ef374d70455daaff2
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Thu, 04 Feb 2021 07:05:45 +0000
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-9ps55 (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  default-token-9ps55:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-9ps55
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:          <none>
```

总结一下整个流程：

![1612490825170](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1612490825170.png)

（1）用户通过kubectl创建Deployment
（2）Deployment创建ReplicaSet
（3）ReplicaSet创建Pod

可以看出Deployment是通过ReplicaSet来管理Pod的多个副本的

学习到这里你一定对k8s组件的各个功能，并且会使用depolyment创建pod资源~



#### 引用

> 《每天5分钟完成kubernetes》