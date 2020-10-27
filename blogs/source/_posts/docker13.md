---
title: docker-swarm实战
date: 2020-10-09 11:13:33
categories: [docker]
tags: [docker]
---
## docker-swarm 简介
docker-swarm是一个集群管理工具，主要有以下几个组件
1 Swarm 主要负责集群的管理和编排工作
2 Node节点，分为manager节点和worker节点
3 Service是任务的定义，管理机或工作节点上执行
4 Task是Service的实例，是容器运行的一组命令
<!--more-->
## docker-swarm搭建
准备两台机器，在一台机器上执行swarm初始化
``` cmd
docker swarm init --advertise-addr 81.68.86.146
```
会显示加入的token和地址端口
``` cmd
docker swarm join --token SWMTKN-1-2ifek5rwyq0k1d4rhhcyxjrmijlbuxm27rfcvqtkoxdgqdgn4j-0epqxkqlixe1r8uvr47g69iox 81.68.86.146:2377
```
在另一台机器上执行上述加入操作,之后执行查看命令，可以看到集群节点状态
``` cmd
docker node list
```
显示一下
``` cmd
ID                            HOSTNAME            STATUS              AVAILABILITY        MANAGER STATUS      ENGINE VERSION
axapk9ke3o47r3er3jeqilpgg *   VM-0-9-ubuntu       Ready               Active              Leader              19.03.6
7ynq5qr8s8tn3j5vkr5ke19k0     instance-6nsdfhv9   Ready               Active                                  19.03.12
```
## docker-swarm service
我们可以创建service服务，然后指定具体的副本数，docker会根据副本数将service生成指定数量的容器执行task，这些容器负载均衡分配到不同的节点上。我们先创建一个名为demo的service
``` cmd
docker service create --name demo busybox sh -c "while true; do sleep 3600; done"
```
上述命令创建了一个名为demo的service, 我们可以通过docker service list 查看服务列表
``` cmd
docker service list
```
可以看到服务列表， 存在名字为demo的服务
``` cmd
ID                  NAME                MODE                REPLICAS            IMAGE               
xr5psjjt0nvx        demo                replicated          1/1                 busybox:latest      
```
我们也可以通过docker ps 查看启动的容器
通过如下命令可以查看service的详细信息
``` cmd
docker service ps demo
```
可以看到服务运行在哪个节点上
接下来我们扩充下服务的数量
``` cmd
docker service scale demo=5
```
然后我们查看demo服务
``` cmd
docker service ps demo
```
会看到如下效果,启动了五个任务分别跑在不同的节点上
``` cmd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
j78ygjp9r42k        demo.1              busybox:latest      VM-0-9-ubuntu       Running             Running 47 minutes ago                       
9w7xifrhztki        demo.2              busybox:latest      instance-6nsdfhv9   Running             Running 19 seconds ago                       
s15r73i10m43        demo.3              busybox:latest      instance-6nsdfhv9   Running             Running 19 seconds ago                       
3ep3sjw50fro        demo.4              busybox:latest      instance-6nsdfhv9   Running             Running 19 seconds ago                       
x80vmybr1dak        demo.5              busybox:latest      VM-0-9-ubuntu       Running             Running 22 seconds ago       
```
通过
``` cmd
docker service list
```
看到启动了五个容器，如果关闭其中一个，swarm会自动帮我们启动一个新的，保持数量为5个
通过
``` cmd
docker service rm demo
```
删除demo服务，这样所有运行的容器都会被删除，执行docker service ps demo会看到demo服务被删除了。
执行docker ps 看到容器也被删掉了
## service搭建wordpress
利用service搭建wordpress，首先要创建一个共用网络，常见两个service，分别是mysql和wordpress
``` cmd
docker network create -d overlay sw-net 
```
创建一个overlay类型的网络sw-net，这个网络可以保证多个宿主机之间通信
之后查看网络列表
``` cmd
docker network list
```
可以看到我们新创建的网络。
接下我们用service启动mysql服务
``` cmd
docker service create --name mysql --env MYSQL_ROOT_PASSWORD=root --env MYSQL_DATABASE=wordpress --network sw-net  --mount type=volume,source=mysql-data,destination=/var/lib/mysql mysql:5.6
```
注意--mount后的key和value对之间用逗号隔开，不能有空格
命令运行后等待服务启动成功，根据docker service ps mysql 可以看到这个服务具体运行在哪个节点。
接下来创建wordpress的service，和之前启动wordpress的容器类似
``` cmd
docker service create --name wordpress -p 7080:80 --env WORDPRESS_DB_USESR=root --env WORDPRESS_DB_PASSWORD=root \
--env WORDPRESS_DB_HOST=mysql:3306 --network sw-net  wordpress
```

## 服务更新
先将服务扩充为多个，这样保证服务稳定性
``` cmd
docker service scale wordpress=5
```
然后执行update
``` cmd
docker service update --image wordpress:2.0  wordpress
```
可以看到wordpress会在后台更新

如果要更新端口则执行如下命令即可
``` cmd
docker service update --publish-rm 8080:5000  --publish-add 8088:5000 wordpress
```
上述命令将8080端口移除，换为8088端口

## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)




