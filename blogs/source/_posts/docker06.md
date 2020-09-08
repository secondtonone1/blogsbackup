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
## docker 设置ssh
1 拉取ubuntu镜像
``` cmd
docker pull ubuntu:18.04
```
2 启动ubuntu容器,将22端口映射为1022端口
``` cmd
docker run -it  -p 1022:22  ubuntu:18.04
```
3 在容器中安装如下应用
``` cmd
apt-get update
apt-get upgrade
apt-get install vim
apt-get install openssh-server
apt-get install net-tools
```
然后vim /etc/ssh/sshd_config
将PermitRootLogin设置为yes
创建文件夹
``` cmd
mkdir -p /var/run/sshd
```
然后启动服务
``` cmd
/usr/sbin/sshd -D &
```
这时我们查看网路端口
``` cmd
netstat -tunpl
```
可以看到22端口启动了
为了让容器启动时可以自启动ssh服务,我们实现一个脚本
vim /run.sh添加如下
``` sh
#!/bin/bash
/usr/sbin/sshd -D
```
然后赋予这个脚本执行权限
``` cmd
chmod +x /run.sh
```
然后exit退出，基于改造的docker提交新的镜像
``` cmd
docker commit cafd85cb0645 ubuntu:ssh
```
然后我们基于这个镜像启动新的容器
``` cmd
docker run  -d  --name ubuntu-ssh -p 1022:22 ca1a463f5c99 /run.sh
```
因为ssh登录需要账户名和密码，账户名为root，密码我们进入容器设置下
``` cmd 
docker exec -it 28afa8e39353 /bin/bash
passwd
```
安装后输入passwd,设置密码.
之后通过ssh连接就可以了
``` cmd
ssh root@172.98.23.45 -p 1022
```
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)













