#### weave网段被占用的问题解决方案

------

此时主机已经完成基础的搭建，安装CNI插件(weave)的时候，查看master状态还是NotReady状态

查看weave-net不可用

![1606962757801](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606962757801.png)

查看对应的pod的错误是容器不断重启

```
kubectl describe  pod <pod-name> -n <ns>
```

![1606962866588](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606962866588.png)

查看容器是非正常状态

![1606962816336](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606962816336.png)

查看容器对应的日志是网段被占用

![1606962982348](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606962982348.png)

查看本机的route，果然10.42.0.0网段被占用，这个排查是之前rancher安装k8s残留下来的

![1606963048536](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606963048536.png)

接下来删除即可啦

![1606963134113](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606963134113.png)我们再看一下route，以及没有42网段了

![1606963168986](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606963168986.png)

接下来删除pod，自动重启之后，weave相关pod已经健康

查看node的状态是Ready状态

![1606963307651](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1606963307651.png)