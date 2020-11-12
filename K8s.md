### K8s基础知识总结

------

一、StatefulSet和Deployment部署的区别

1、StatefulSet

删除Pod之后，只有StatefulSet和Pod之外的任何资源都没有被删除，之前创建的，pv以及pvc对象没有发生任何变化，这也是 StatefulSet 的行为，它会在服务被删除之后仍然保留其中的状态，也就是数据，这些数据就都存储在 `PersistentVolume` 中

如果我们重新创建相同的 StatefulSet，它还会使用之前的 PV 和 PVC 对象，不过也可以选择手动删除所有的 PV 和 PVC 来生成新的存储，这两个对象都属于 Kubernetes 的存储系统

StatefulSet 是 Kubernetes 为了处理有状态服务引入的概念，在有状态服务中，它为无序和短暂的 Pod 引入了顺序性和唯一性，使得 Pod 的创建和删除更容易被掌控和预测，同时加入 PV 和 PVC 对象来存储这些 Pod 的状态，我们可以使用 StatefulSet 实现一些偏存储的有状态系统，例如 Zookeeper、Kafka、MongoDB 等，这些系统大多数都需要持久化的存储数据，防止在服务宕机时发生数据丢失。