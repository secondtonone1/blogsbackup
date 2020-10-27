---
title: k8s基本概念和单节点服务搭建
date: 2020-10-20 11:11:51
categories: [docker]
tags: [docker]
---
## K8s基本概念
Kubernetes（k8s）是自动化容器操作的开源平台，这些操作包括部署，调度和节点集群间扩展。
k8s有两种节点，master节点和node节点
master节点：是集群的大脑，master节点包括，Api Server，Scheduler，Controller 。
Api Server组件，该组件主要是为了响应UI或者CLI的请求
Scheduler组件： 用来调度容器运行和停止，以及运行在哪些节点上
Contoller组件：维持服务可扩展，保证稳定运行数量
Etcd组件：主要是分布式存储k8s的服务状态等。
Node节点：包括Pod，Docker，kubelet，kube-proxy，fluentd

Pod ：运行在节点上，包含一组容器和卷。同一个Pod里的容器共享同一个网络命名空间，可以使用localhost互相通信。
Docker： 容器技术
kubelet：负责在创建容器，分配volume和network等
kube-proxy：主要负责网络端口的代理和转发
fluentd：负责日志的采集和存储。

## minikube 搭建单节点集群
安装minikube前提需要安装kubectl和VM
1 安装kubectl
``` cmd
curl -LO https://storage.googleapis.com/kubernetes-release/release/`curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt`/bin/linux/amd64/kubectl
```
然后提取权限
``` cmd
chmod +x ./kubectl
```
将二进制文件添加至路径中
``` cmd
cp kubectl /usr/local/bin
```
此时执行kubectl可以看到帮助信息
2 安装virtual-box
先升级apt
``` cmd
sudo apt update && sudo apt upgrade
```
然后安装依赖包
``` cmd
sudo apt install gdebi build-essential
```
去virtual-box下载源码
``` cmd
wget http://download.virtualbox.org/virtualbox/5.2.8/virtualbox-5.2_5.2.8-121009~Ubuntu~xenial_amd64.deb
```
安装virtual box
``` cmd
gdebi  virtualbox-5.2_5.2.8-121009~Ubuntu~xenial_amd64.deb
```
继续安装其他依赖包
``` cmd
apt-get install libqt5x11extras5 libsdl1.2debian
```
3 关闭swap交换分区
关闭swap
``` cmd
swapoff -a
```
然后打开swap配置
``` cmd
vi /etc/fstab
```
注释掉swap分区的那一行
4 安装minikube
从阿里云地址下载
``` cmd
curl -Lo minikube http://kubernetes.oss-cn-hangzhou.aliyuncs.com/minikube/releases/v0.26.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```
接下来我们查看下minikube 版本，显示版本证明安装成功
``` cmd
minikube version
```
创建minikube集群
``` cmd
minikube start
```
可以看到日志后minikube启动成功了
5 kubectl命令

kubectl view config 查看配置信息
kubectl config get-contexts 查看context信息
假如我们有两个集群，可以使用两个context，利用不同的context链接不同集群
kubectl cluster-info 查看集群信息

minikube ssh 进入虚拟主机环境，之后可以执行docker命令
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)
