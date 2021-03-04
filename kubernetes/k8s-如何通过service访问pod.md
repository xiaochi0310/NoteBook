如何通过service访问pod



​		我们知道对于Pod而言其IP是不固定的，因为若该Pod发生故障时会立即被新的Pod代替，此时会分配新的IP，注意这些IP是虚拟IP，但是思考一下Pod对外提供服务，是怎么在IP变化的情况下，还能提供完善的功能呢?

​		答案是通过Service

##### 一、创建Service

我们案例使用之前介绍[K8s框架](https://juejin.cn/post/6925602768635822088)使用deployment控制器创建对外服务

在之前案例的基础上，我们增加service资源(service.yaml)

```yaml
# service的版本
apiVersion: v1
# 资源类型
kind: Service
# service的命名空间以及名字
metadata:
  namespace: k8s-test
  name: service
spec:
  # 将service8080端口映射到30000端口
  ports:
    - name: "service-port"
      targetPort: 8080
      port: 8080
      nodePort: 30000
  # 指明label为service-test的pod为service的后端
  selector:
    app: service-test
  # 暴露给外网
  type: NodePort

```

执行

```bash
kubectl apply -f service.yaml
```

查看service服务

```bash
[centos@wunaichi k8stest]$ kubectl get service -n k8s-test
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)          AGE
service   NodePort   10.111.35.139   <none>        8080:30000/TCP   18d
```

service分配到CLUSTER-IP为10.111.35.139，端口映射为8080:30000

查看service与pod的关系

```bash
[centos@wunaichi k8stest]$ kubectl describe service service -n k8s-test
Name:                     service
Namespace:                k8s-test
Labels:                   <none>
Annotations:              <none>
Selector:                 app=service-test
Type:                     NodePort
IP:                       10.111.35.139
Port:                     service-port  8080/TCP
TargetPort:               8080/TCP
NodePort:                 service-port  30000/TCP
Endpoints:                10.44.0.3:8080,10.44.0.4:8080
Session Affinity:         None
External Traffic Policy:  Cluster
Events:                   <none>
```

其中Endpoints罗列了2个Pod的IP和端口，我们知道Pod的IP是在容器中配置的，那么Service的ClusterIP又是配置在哪里的呢?CLUSTER-IP又是如何映射到Pod IP的呢?

答案是kube-proxy+iptables

##### 二、Service的工作原理

Service是由kube-proxy组件和iptables组成。iptables规则将service的ip映射到pod的ip.

查看机器的iptables

```
sudo iptables-save
```

我们截取关键部分进行解读

![1614046493300](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614046493300.png)

第一条：宿主机访问service，对于原地址非10.24.0.0/16，目的地址为10.111.35.139，则跳转到KUBE-MARK-MASQ( 这里跳转到KUBE-MARK-MASQ是为了包出宿主机时，其ip是宿主机ip)，不怎么重要，重点看第二条

第二条：凡是目的地是10.111.35.139(service的ip)端口是8080都要跳转到KUBE-SVC-IUAUJYFCIUUMOL2O这条规则

我们继续跳转。 KUBE-SVC-IUAUJYFCIUUMOL2O以50%的概率跳转到KUBE-SEP-3C4G3TP5TJKPOH3O，以50%的概率跳转到KUBE-SEP-2ATDMDQBZCO4RJUM

![1614047734468](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614047734468.png)

以跳转到KUBE-SEP-3C4G3TP5TJKPOH3O为例

![1614047871913](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614047871913.png)

可以看到KUBE-SEP-3C4G3TP5TJKPOH3O将目的地址通过DNAT转到10.44.0.3:8080， 而这个ip正是其中一个pod的ip

这里DNAT的作用就是把流入的IP和端口改成新的端口和地址，也就是被代理的Pod的IP和端口

到此，我们把service的请求已经交给对应的pod来进行处理

所以iptables规则将访问Service的流量会以相同概率转发到后端Pod，而且使用类似轮询的负载均衡策略。而这些规则正是kube-proxy监听Pod的变化事件，在宿主机上生成并维护的。

`kube-proxy` 会监视 Kubernetes 控制节点对 Service 对象和 Pod 对象的添加和移除。 对每个 Service，它会配置 iptables 规则，从而捕获到达该 Service 的 clusterIP`和端口的请求，进而将请求重定向到 Service 的一组后端中的某个 Pod 上面。 对于每个 Endpoints 对象，它也会配置 iptables 规则，这个规则会选择一个后端组合。默认的策略是，kube-proxy 在 iptables 模式下随机选择一个后端。

可以看到，kube-proxy 通过 iptables 处理 Service 的过程，其实需要在宿主机上设置相当多的 iptables 规则。而且，kube-proxy 还需要在控制循环里不断地刷新这些规则来确保它们始终是正确的

##### 三、外网如何访问service

除了cluster内部访问service，很多情况下我们也希望可以将service暴露到外部

service通过cluster内部的IP对外提供服务，但是只有cluster节点以及Pod才可以访问。那外网怎么访问service?那就是NodePort。我们再回看一下第一节的yaml文件

```yaml
# service的版本
apiVersion: v1
# 资源类型
kind: Service
# service的命名空间以及名字
metadata:
  namespace: k8s-test
  name: service
spec:
  # 将service8080端口映射到30000端口
  ports:
    - name: "service-port"
      # pod监听的端口
      targetPort: 8080
      # cluster监听的端口
      port: 8080
      # 节点上监听的端口
      nodePort: 30000
  # 指明label为service-test的pod为service的后端
  selector:
    app: service-test
  # 暴露给外网
  type: NodePort
```

可以看到 type: NodePort ；且含有节点监听的端口30000

最终，Node和ClusterIP在各自端口上接收到的请求都会通过iptables转发到Pod的targetPort

![1614060276686](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614060276686.png)

可以看到对外端口已经生效

我们再看看一下iptables规则

![1614061041873](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614061041873.png)

可以看到监听30000端口，该端口的请求都会跳转到KUBE-SVC-IUAUJYFCIUUMOL2O，而KUBE-SVC-IUAUJYFCIUUMOL2O同cluster一样均会按照相同比例跳转到某个Pod上面。



至此关于外部如何访问Pod的问题应该很清楚了吧

![1614066812071](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1614066812071.png)

##### 引用

[kubernetes官方文档-服务](https://kubernetes.io/zh/docs/concepts/services-networking/service/#proxy-mode-iptables)

《每天5分钟玩转kubernetes》