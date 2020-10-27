---
title: Docker Secret加密
date: 2020-10-19 10:34:47
categories: [docker]
tags: [docker]
---
## Docker Secret
在我们启动docker或者service需要指定密码，这种密码我们有时不想被别人知道，所以可以采用docker secret方式管理。
创建secret可以有两种方式，一种通过文件创建，一种通过命令行创建
我们在本地创建一个文件passwd
``` cmd
zack1024
```
<!--more-->
接下我们可以通过如下命令创建secret
``` cmd
docker secret create my-pw  passwd
```
运行后可以看到屏幕输出如下
``` cmd
qq463dkw4da9s05rwinntmo09
```
这是一段加密后的数字，接下来我们查看下所有secret
``` cmd
docker secret list
```
可以看到secret列表
``` cmd
ID                          NAME                DRIVER              CREATED              UPDATED
qq463dkw4da9s05rwinntmo09   my-pw                                   About a minute ago   About a minute ago
```
为了保护隐私，我们可以将passwd这个文件删除。
当然也可以通过命令行方式传入，从而创建secret
``` cmd
echo "zack1024" | docker secret create mysql-root-pw -
```
接下来我们利用secret创建一些service
``` cmd
docker service create --name se-busybox --secret my-pw busybox sh -c "while true; do sleep 3600; done"
```
接下来我们查看下跑起来的service 列表
``` cmd
docker service list
```
可以看到服务列表，docker service ps se-busy的box 查看具体服务，可以看到这个服务运行的节点书和状态
``` cmd
ID                  NAME                IMAGE               NODE                DESIRED STATE       CURRENT STATE            ERROR               PORTS
6xgkpk55tno7        se-busybox.1        busybox:latest      VM-0-9-ubuntu       Running             Running 19 seconds ago     
```
接下来我们在VM-0-9-ubuntu这台机器上执行docker ps 找到se-busybox服务运行的容器
``` cmd
CONTAINER ID        IMAGE                COMMAND                  CREATED             STATUS              PORTS                               NAMES
23d0dce1b7cf        busybox:latest       "sh -c 'while true; …"   9 minutes ago       Up 9 minutes                                            se-busybox.1.6xgkpk55tno7fi6uiim4am00n
```
我们进入这个容器，docker exec -it 23d0dce1b7cf sh 
在容器里，接下来我们进入这个目录 cd /run/secrets/
``` cmd
/run/secrets # ls
my-pw
/run/secrets # cat my-pw
zack1024
```
通过cat my-pw可以看到我们最初设定的密码。
接下来我们利用secret构建mysql服务
``` cmd
docker service create --name se-db --secret my-pw -e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/my-pw mysql
```
启动mysql后进入容器内部也可以看到/run/secrets/目录下有my-pw文件，其内部记录的就是我们设置的密码。
我们在容器内部执行
``` cmd
mysql -u root -p
```
录入之前设置的密码, 可以看到连接成功。
## 通过docker-compose使用secrete
可以通过secrets字段指定要使用的密钥，这个密钥必须提前通过docker secret create创建
也可以通过secrets字段单独指定密钥名称，在其子域下指定file关键字,就是密码保存的文件
``` docker-compose
version: '3.6'
services:

  web:
    image: wordpress
    ports:
      - 8080:80
    secrets:
      - my-pw
    environment:
      WORDPRESS_DB_HOST: mysql
      WORDPRESS_DB_PASSWORD_FILE: /run/secrets/my-pw
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
    image: mysql
    secrets:
      - my-pw
    environment:
      MYSQL_ROOT_PASSWORD_FILE: /run/secrets/my-pw
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

secrets:
  my-pw:
    file: ./password
```
然后我们通过docker stack 启动服务
``` cmd
docker stack deploy wordpress -c=docker-compose.yml
```

## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)