---
title: Dockerfile入门
date: 2020-08-03 10:08:29
categories: [docker]
tags: [docker]
---
今天介绍下Dockerfile的基本命令和使用案例
## Dockerfile基本命令

``` cmd
FROM ：基础镜像，该镜像基于哪个镜像生成
MAINTAINER ：镜像维护者的姓名和邮箱
RUN ：构建容器时需要运行的命令
EXPOSE ：容器对外暴露的端口
WORKDIR ： 指定在创建容器后，终端默认登录进来的工作目录
ENV ：用来在构建镜像过程中设置环境变量
ADD : 将宿主机目录下的文件拷贝进镜像且ADD命令会自动处理URL和解压tar压缩包
COPY : 类似ADD，拷贝文件和目录到镜像中。将从构建上下文目录中
CMD : 指定容器启动时要运行的命令，如果有多个CMD命令，只有最后一个生效，CMD会被docker run 之后的参数替换。
ENTRYPOINT ：指定一个容器启动时要运行的命令，ENTRYPOINT的目的和CMD一样，都是指定容器启动程序及参数
ONBUILD ：当构建一个被继承的Dockerfile时运行命令，父镜像在被子继承后父镜像的onbuild被触发。
```
<!--more-->
## Dockerfile使用示例
``` Dockerfile
FROM centos
MAINTAINER zack<zack@163.com>

ENV MYPATH /usr/local
WORKDIR $MYPATH
RUN yum -y install vim
RUN yum -y install net-tools
EXPOSE 80

CMD echo $MYPATH
CMD echo "success---------ok"
CMD /bin/bash
```
## 根据Dockerfile生辰镜像
docker build -t mycentos:v1.0 .
## 启动容器
docker run -it --name mycentosdk  2081f3e41884


