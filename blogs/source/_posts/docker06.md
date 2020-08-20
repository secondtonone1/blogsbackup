---
title: docker命令补充
date: 2020-08-18 11:42:08
categories: [docker]
tags: [docker]
---
## 基于本地模板导入镜像
假如本地有一个ubuntu系统模板压缩包, 可以通过import导入生成新的镜像
``` cmd
cat ubuntu-18.04.tar.gz | docker import - ubuntu:18.04
```
## 存出和导入镜像
存出镜像
``` cmd
docker save -o ubuntu-18.04.tar  ubuntu:18.04
```
导入镜像
``` cmd
docker load -i ubuntu-18.04.tar
```
<!--more-->
## 导出容器
``` cmd
docker export -o ce.tar ce5
```
## 导入容器
``` cmd
docker import ce.tar - ce:v1.0
```
## 查看容器内进程
``` cmd
docker top 容器id
```
## docker私有仓库
先拉取registry镜像
``` cmd
docker pull registry
```
根据registry启动镜像，构造仓库
``` cmd
docker run -d -p 5000:5000 -v /opt/data/registry:/var/lib/registry  registry
```
然后我们查看本地有哪些镜像，随便选择一个推送上去
``` cmd
docker images
```
选择一个mongo,我们打个tag，tag前面要写上我们服务器地址和仓库端口号
``` cmd
docker tag mymongo:latest 81.68.86.146:5000/mymongo
```
推送到私有仓库
``` cmd
docker push 81.68.86.146:5000/mymongo
```
如果出现了http错误，请修改/etc/docker/daemon.json文件
``` cmd
"insecure-registries": ["81.68.86.146:5000"]
```
然后重启docker服务
``` cmd
sudo systemctl daemon-reload
sudo systemctl restart docker.service
sudo systemctl enable docker.service
```
重启docker
``` cmd
docker restart $(docker ps -aq)
```
这样再次push就可以将镜像push到docker私有仓库了。

## 利用容器卷备份和迁移数据
1 备份，可以将数据备份至挂在目录，这样外界就可以访问并获取了。
用ubuntu镜像启动一个新的容器worker，该容器和dbdata容器共享卷, worker启动后将/dbdata下的数据打包放在/backup下
``` cmd
docker run --volumes-from dbdata -v $(pwd):/backup --name worker ubuntu tar cvf /backup/backup.tar /dbdata
```
2 还原
启动一个容器,挂在/dbdata目录
``` cmd
docker run -v /dbdata  --name  dbdata2 ubuntu /bin/bash 
```
再用ubuntu 启动一个新的镜像，共享dbdata2容器的卷
``` cmd
docker run --volume-from dbdata2 -v $(pwd):/backup busybox tar xvf /backup/backup.tar
```









