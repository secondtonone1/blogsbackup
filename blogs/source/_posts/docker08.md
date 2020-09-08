---
title: images的发布和私有仓库
date: 2020-09-07 18:50:10
categories: [docker]
tags: [docker]
---
## images发布到docker hub
首先登录[https://hub.docker.com](https://hub.docker.com)注册自己的账号，然后创建仓库
接着将我们之前的一个镜像打tag，tag的形式为id/镜像名:版本, id就是dockerhub的id。
<!--more-->
``` cmd
#登录
docker login
#给镜像打标签
docker tag status  secondtonone1/status
#查看打标签后的镜像
docker images
docker push secondtonone1/status
```
之后登录docker hub网址就能看到我们的仓库了
如果将本地的镜像删除，再次pull，就可以从docker-hub中拉取刚才提交的镜像
``` cmd
#删除本地镜像
rmi secondtonone1/status
#拉取远程镜像
docker pull secondtonone1/status
```
## 搭建私有仓库
可以拉取registry镜像搭建私有仓库
``` cmd
docker pull registry:2
```
然后启动镜像
``` cmd
docker run -d -v /home/zack/dockerwork/registry -p 5000:5000 --restart always --name myregistry registry:2
```
可以从其他机器telnet这个台机器的5000端口，保证端口畅通
生成一个新的镜像，镜像名字要用这台机器的  地址:端口/镜像名
``` cmd
docker build -t 81.68.86.146:5000/comsig .
```
此时可以尝试提交镜像，由于镜像仓库未配置权限信息，所以会提交失败。
此时进入/etc/docker下，修改daemon.json文件
``` cmd
{
  "registry-mirrors": ["https://hya0o79u.mirror.aliyuncs.com"],
  "insecure-registries": ["81.68.86.146:5000"]
}
```
registry-mirrors是阿里云加速地址，insecure-registries是我们配置的私有仓库的地址。
接着修改dockerservice配置,  sudo vim /lib/systemd/system/docker.service
添加
``` cmd
EnvironmentFile=-/etc/docker/aemon.json
```
接下来 push我们的镜像到私有仓库
``` cmd
docker push 81.68.86.146:5000/comsig
```
可以去docker官网
[https://docs.docker.com/registry/spec/api/#catalog](https://docs.docker.com/registry/spec/api/#catalog)
查看相应的registry的api, 进而获取私有仓库信息。
在浏览器输入81.68.86.146:5000/v2/_catalog
可以看到结果如下
{"repositories":["comsig"]}
## 个人公众号
![wxgzh.jpg](wxgzh.jpg)
