docker 数据存储的2种方式



docker提供2种方法对数据卷进行存储。 1、bind mount  2、docker管理数据卷

##### 1、bind mount

使用-v 关键字标识容器数据挂载路径。这里一定要指定宿主机地址和容器地址。

```shell
# -v <host path>:<container path>
docker run -d -p 80:80 -v /var/run:/var/run test
```

bind还有其他特性

可以添加单个文件

```shell
# docker-compose.yaml文件 创建容器
	prometheus:
        image: prom/prometheus
        container_name: prometheus
        hostname: prometheus
        restart: always
        # 挂载单个文件
        volumes:
            - /home/centos/config/prometheus.yml:/etc/prometheus/prometheus.yml
        ports:
            - "9090:9090"
        networks:
            - monitor
```

可以限制挂载文件夹的权限

```shell
# docker-compose.yaml文件 添加容器
	cadvisor:
        image: google/cadvisor:latest
        command: "--enable_load_reader=true"
        container_name: cadvisor
        hostname: cadvisor
        restart: always
        # 对文件夹的权限进行限制
        volumes:
            - /:/rootfs:ro
            - /var/run:/var/run:rw
            - /sys:/sys:ro
            - /var/lib/docker/:/var/lib/docker:ro
        ports:
            - "8080:8080"
        networks:
            - monitor
```

可以看到，bind mount的使用十分的简单和高效。但是也可以看到有很大的不灵活性，因为挂载的时候需要指定宿主机文件系统的具体路径，对后续的数据迁移很不友好。



##### 2、docker管理的volume

也是使用关键字-v  ，与第一种方法不同的是不用指出挂载源地址，而要指出一个挂载点。挂载点常用的是local和nfs类型

local默认挂载点是/var/lib/docker/volumes/<volumeName>/_data

这样docker可以管理数据卷。

```shell
# 创建数据卷
docker volume create <volume_name>
# 列出所有的数据卷
docker volume ls
# 删除数据卷。删除容器后，对应的数据卷不会被删除，需要执行rm volume删除数据卷
docker volume rm <volume_name>
# 查看某个数据卷的详情
docker volume inspect <volume_name>
```

docker管理的数据卷，更加安全和高效。但是也有其局限性，不支持文件只支持目录；不能控制文件夹的读写权限，都拥有读写权限；且可移植性强，不需要指定host目录。

现在使用docker-compose创建一个cadvisor容器

```dockerfile
# docker-compose.yaml
version: '2'

services:
    cadvisor:
        image: google/cadvisor:latest
        command: "--enable_load_reader=true"
        container_name: cadvisor
        hostname: cadvisor
        restart: always
        volumes:
            - /:/rootfs:ro
            - /var/run:/var/run:rw
            - /sys:/sys:ro
            - /var/lib/docker/:/var/lib/docker:ro
            - test:/var/lib/docker
        ports:
            - "9100:9100"
volumes:
  test:
```

可以看到挂载卷使用了前面2种存储方式。

执行docker-compose创建容器，如下图。可以看到会creating volume，这也就是docker volume create

![1610354559178](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1610354559178.png)



