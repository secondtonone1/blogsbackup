---
title: k8s最小调度单位pod
date: 2020-10-26 20:46:13
categories: [docker]
tags: [docker]
---
pod是k8s调度最小单位，一个pod可以包含多个容器，各容器之间共享同一个网络。
可以通过yml文件创建一个pod
<!--more-->
``` yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
yml文件中各个属性
kind: 指定创建资源的角色/类型
metadata: 资源的元数据/属性
metadata下name和labels分别为
name: web04-pod #资源的名字，在同一个namespace中必须唯一  
labels: #设定资源的标签
spec: 指定该资源的内容
``` cmd
spec:#specification of the resource content 指定该资源的内容  
    restartPolicy: Always #表明该容器一直运行，默认k8s的策略，在此容器退出后，会立即创建一个相同的容器  
    nodeSelector:     #节点选择，先给主机打标签kubectl label nodes kube-node1 zone=node1  
        zone: node1
    containers:  #容器
    - name: nginx  #容器名字
      image: nginx  #镜像名字
      ports:
      - containerPort: 80  #容器对外开放的端口
```
## 启动pod
通过Kubectl version可以查看看k8s版本
根据pod的yml文件启动一个pod
``` cmd
kubectl create -f pod_nginx.yml
```
根据当前的pod_yaml文件创建一个pod，会显示pod "nginx" created
我们通过
``` cmd
kubectl get pods
```
可以看到nginx 的pod已经运行
也可以通过
``` cmd
kubectl get pods -o wide
```
查看详细信息
``` cmd
NAME    READY    STATUS    RESTARTS    AGE      IP             NODE
nginx    1/1     RUNNING    0           1m    172.17.0.4      minikube
```
可以看到容器 运行在minikube这个节点上， 容器的ip为172.17.0.4
我们通过minikube ssh 进入virtualbox里， 这样通过docker ps可以查看虚拟机中运行的容器
我们通过docker network list 查看 虚拟机内部的所有网络
docker network inspect bridge 查看bridge网络
可以看到bridge网络里有容器为nginx的容器网络地址为172.17.0.4，正好就是我们通过kubectl get pods -o wide获取的
除此之外，我们可以通过
``` cmd
kubectl exec -it nginx  sh 
```
这种方式直接进入nginx pod 里的容器里了。
如果pod中有多个容器，可以选择进入某个容器
我们可以先查看nginx pod中的容器
``` cmd
kubectl describe pods nginx 
```
接下来我们可以通过-c 选项进入指定容器
``` cmd
kubectl exec -c 容器id
```
我们启动pod后，容器运行在pod中，外界无法访问，需要将pod的端口暴漏出去
``` cmd
kubectl  port-forward nginx 8080:80
```
这样就将nginx容器内部的80端口映射为本机的8080端口，可以通过127.0.0.1：8080访问nginx服务了。

## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)



