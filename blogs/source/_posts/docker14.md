---
title: DockerStack 实战
date: 2020-10-16 15:54:04
categories: [docker]
tags: [docker]
---
## Docker Stack简介
docker stack是基于cluster集群模式，发布服务的一个功能。
docker stack 有如下几个命令
docker stack deploy  发布或者更新一个stack
docker stack list 获取所有stack
docker stack ps 列出stack中运行的task
docker stack services  列出stack中的服务
docker stack rm 移除stack
<!--more-->
## wordpress实战
基于docker stack 实现之前的wordpress功能

``` docker-compose
version: '3'

services:

  web:
    image: wordpress
    ports:
      - 8080:80
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD: root
    networks:
      - my-network
    depends_on:
      - mysql
    deploy:
      mode: replicated
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      update_config:
        parallelism: 1
        delay: 10s

  mysql:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: wordpress
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - my-network
    deploy:
      mode: global
      placement:
        constraints:
          - node.role == manager

volumes:
  mysql-data:

networks:
  my-network:
    driver: overlay
```
简单解释下docker-compose的各个参数
services : 要运行的服务，这里填写了两个服务，web服务和mysql服务
web服务的构建方式为镜像构建，端口为80端口映射为8080端口
environment : 表示环境变量，通过environment传递给容器
networks : 两个服务都通过my-network 网络通信
deploy : 这里定义了发布规则,mode 为replicated表示这个服务可以复制很多个实例
模式为global表示之启动一个实例，不允许复制
replicas : 表示副本数量
restart_policy : 重启策略，condition为失败时，重启
delay : 多个实例重启延迟为5s
max_attempts : 表示最大尝试次数为3
update_config ： 表示更新配置，parallelism表示并行更新数量
placement ：设置服务运行的位置，constraints表示约束，
node.role == manager只允许该服务运行在manager节点
volumes ：表示挂载的卷
networks: 设置网络driver为overlay，这样可以允许多主机互通
## 通过docker stack发布服务
执行如下命令
``` cmd
docker stack deploy  wordpress --compose-file  docker-compose.yml
```
可以看到创建了如下服务
``` cmd
Creating network wordpress_my-network
Creating service wordpress_web
Creating service wordpress_mysql
```
接下来我们看看运行了哪些stack
``` cmd
docker stack list
```
会展示cluster运行的stack
``` cmd
NAME                SERVICES            ORCHESTRATOR
wordpress           2                   Swarm
```
可以看到名字为wordpress的stack正在运行，其上运行了两个服务
查看wordpress上具体的服务
``` cmd
docker stack services wordpress
```
可以看到服务
``` cmd
ID                  NAME                MODE                REPLICAS            IMAGE               PORTS
2soubn9aey1y        wordpress_mysql     global              1/1                 mysql:5.7           
p3h9g6tigocx        wordpress_web       replicated          3/3                 wordpress:latest    *:8080->80/tcp
```
有两个服务，分别是wordpress_mysql和wordpress_web。在web和mysql之前增加了wordpress这个stack的名字
接下来列出wordpress中运行的task
``` cmd
docker stack ps wordpress
```
可以看到stack上跑了三个服务
``` cmd
ID                  NAME                                        IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
gvtnr0dy5sjs        wordpress_mysql.axapk9ke3o47r3er3jeqilpgg   mysql:5.7           VM-0-9-ubuntu       Running             Running 24 minutes ago                       
q15ncnt0cx73        wordpress_web.1                             wordpress:latest    instance-6nsdfhv9   Running             Running 24 minutes ago                       
z3ivbcopqguv        wordpress_web.2                             wordpress:latest    VM-0-9-ubuntu       Running             Running 24 minutes ago                       
zmlyu8lnw78x        wordpress_web.3                             wordpress:latest    instance-6nsdfhv9   Running             Running 24 minutes ago    
```
如果要更新服务，可以通过修改docker-compose修改配置，然后重新deploy指定修改后的docker-compose即可。

最后可以通过docker stack rm 删除stack
``` cmd
docker stack rm wordpress
```
可以看到服务被移除
``` cmd
Removing service wordpress_mysql
Removing service wordpress_web
Removing network wordpress_my-network
```
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)

