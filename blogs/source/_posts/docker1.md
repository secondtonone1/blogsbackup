---
title: docker 基本命令
date: 2020-07-02 09:15:38
categories: [docker]
tags: [docker]
---

## docker 基本命令
今天介绍一些docker基本命令，自己最近在学习。docker安装就不介绍了，接下来介绍一些docker常用命令
<!--more-->
### 查看镜像
查看本地所有镜像
sudo docker images -a 
如果查看镜像id
sudo docker images -aq
查看摘要信息
sudo docker images --digests
查看摘要和imageid 的全部信息
sudo docker images --digests --no-trunc
在docker仓库搜索tomcat镜像
sudo docker search tomcat
搜索星数大于30的tomcat镜像
sudo docker search -s 30 tomcat
### 拉取镜像
将tomcat镜像拉取到本地
sudo docker pull tomcat
上面的命令相当于
sudo docker pull tomcat:latest
### 删除镜像
sudo docker rmi tomcat
删除本地所有镜像
sudo docker rmi -f $(docker images -aq)
### 创建容器
启动centos容器，本地没有centos镜像会优先拉取，然后启动
sudo docker run -it  centos
上面的命令it表示交互方式，并启动新的终端，所以会进入新的终端，终端环境是centos镜像
docker run -d --name mycentos centos 启动后退出，ps看不到
-d 表示以守护进程方式启动，但是由于容器没有任务，所以自动关闭了，所以ps命令看不到。
-- name 指定新启动的容器的名字
### 查看本地正在运行的docker
sudo docker ps
### 查看上一次运行的docker
sudo docker ps -l
### 查看本地所有docker
sudo docker ps -a
### 退出docker
当我们在docker的终端中，想要退出终端，可以使用exit命令
exit命令会导致容器关闭
我们也可以使用Ctrl p q的方式退出，按住ctr键，然后先按p,再按q，
这种方式退出容器，不会导致容器关闭。
### 关闭容器
我们可以根据容器id关闭指定容器
sudo docker stop  0ebc1216db4b
强制关闭容器
sudo docker kill  0ebc1216db4b
### 重启和启动容器
根据容器id启动容器
sudo docker start 0ebc1216db4b
根据容器id重启容器
sudo docker restart 0ebc1216db4b
### 查看容器日志
docker logs -f -t --tail 行数 容器id
-f 跟随最新的文件
-t 打印时间
--tail 显示最近多少条
我们创建一个mycentos的容器，让其以守护方式运行，并且执行轮询输出hello world
docker run --name mycentos -d centos /bin/sh -c "while true ; do echo hello world ; sleep 2; done "
我们查看docker日志
docker logs -f -t --tail 3 380975cf268a
### 查看容器内部进程
我们之前创建的mycentos容器以守护进程的方式轮询输出hello world
可以根据容器id查看其内部进程
docker top 380975cf268a
同样可以查看容器内部细节
docker inspect 380975cf268a
### 进入容器
如果容器在后台运行，可以通过attach和exec命令向容器发送指令，也可以实现进入容器的效果
docker attach  本机的输入直接输到容器中， 不启动新的终端，执行命令
docker attach 56d3b13fdc70   //进入docker中
docker exec  在docker 里面新开了一个bash 进程，在该终端可以通过命令和容器交互，执行命令
//exec也可以实现进入容器的目的
docker exec -it 56d3b13fdc70   /bin/bash  
上述命令通过exec 向容器发送 /bin/bash命令，这样会产生新的终端，进入docker
docker exec -it 56d3b13fdc70 ls  
上述命令没有进入容器，但是通过容器启动的新终端向容器发送了ls命令
### 将容器内容copy至宿主机
docker cp 容器id:文件路径  宿主机路径
例如
docker cp 56d3b13fdc70:/tmp/tmp.log    ~/download
将容器中的文件copy至宿主机~/download文件夹下






