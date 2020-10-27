---
title: docker之CMD, ENTRYPOINT, RUN使用和对比
date: 2020-09-07 14:28:43
categories: [docker]
tags: [docker]
---
## CMD
CMD命令是在容器启动后执行的命令，一个Dockerfile可以有多个CMD，但是只有最后一个CMD生效。当容器启动时如果指定了命令，那么CMD的命令将被忽略。
<!--more-->
写一个Dockerfile
``` Dockerfile
FROM alpine:latest
WORKDIR /workdir
ENV  name "Docker"
CMD  echo $name
```
生成新的镜像 secondtonone1/alpine-cmd
docker build -t secondtonone1/alpine-cmd .
生成后生成容器
``` cmd
docker run --rm --name cmd secondtonone1/alpine-cmd
```
可以看到输出docker了
接下来我们在容器启动时后边增加一个命令
``` cmd
docker run --rm -it --name cmd secondtonone1/alpine-cmd sh
```
此时进入了容器内部，执行了sh命令。Dockerfile中的cmd被忽略了。
## RUN
run命令是在构建镜像时执行的命令，我们可以安装一些应用。
``` Dockerfile
FROM ubuntu:18.04
WORKDIR /workdir
RUN  apt-get update
RUN  apt-get install -y net-tools
CMD  netstat
```
生成镜像
``` cmd
docker build -f Dockerfile -t cmd2 .
```
生成容器并启动
``` cmd
docker run -it --rm  cmd2
```
可以看到容器启动后调用了cmd命令netstat
``` cmd
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path
```
## ENTRYPOINT
ENTRYPOINT 和CMD不同，他不会被docker启动后执行的命令覆盖 
``` cmd
FROM ubuntu:18.04
WORKDIR /workdir
RUN  apt-get update
RUN  apt-get install -y net-tools
ENTRYPOINT  netstat
```
生成镜像
``` cmd
docker build -f Dockerfile -t cmd3 .
```
生成容器并启动
``` cmd
docker run -it --rm  cmd3 /bin/bash
```
可以看到容器启动后并没有执行/bin/bash命令，而是调用了ENTRYPOINT命令netstat
``` cmd
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path
```
## RUN和CMD支持参数形式命令
``` Dockerfile
FROM ubuntu:18.04
WORKDIR /workdir
ENV  name "Docker"
RUN  ["/bin/bash", "-c", "apt-get update"] 
RUN  ["/bin/bash", "-c", "apt-get install -y net-tools"] 
CMD  ["/bin/bash","-c","echo Hello $name !"]
```
生成镜像
``` cmd
docker build -f ./Dockerfile -t cmd4 .
```
运行容器
``` cmd
docker run -it --rm cmd4
```
可以看到输出了Hello, Docker!
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)