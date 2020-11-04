---
title: ReplicaSet和ReplicationController
date: 2020-10-27 14:48:40
categories: [docker]
tags: [docker]
---
前文提及kubectl根据pod yml启动pod，接下来我们删除pod
``` cmd
kubectl delete -f pod_nginx.yml
```
查看运行pod
``` cmd
kubectl get pods
```
<!--more-->
编写yml，实例类型为ReplicationController
``` cmd
apiVersion: v1
kind: ReplicationController 
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    app: nginx
  template:
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
selector是选择器，告诉Service之间如何发现pod
teplate定义了pod格式
metadata是pod的元信息
spec指定pod内容，containers定义了容器
接下来我们启动ReplicationController
``` cmd
kubectl create -f rs_nginx.yml
```
因为我们定义了副本数量为3，所以启动三个pod
``` cmd
kubectl get rc
```
查看replicationcontroller，可以看到启动了nginx的controller
同时，我们查看下pods
``` cmd
kubectl get pods
```
当我们删除其中一个pod时，controller会重新启动一个pod，保证副本数为三个
``` cmd
kubectl delete pods nginx-6r92b
```
再次查看kubectl get pods 查看pod列表，发现pod数量仍为三个
我们也可以扩充副本数量
``` cmd
kubectl scale rc nginx --replicas=2 
```
同样我们可以通过replicaset 设置启动pod
``` yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    tier: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      name: nginx
      labels:
        tier: frontend
    spec:
      containers:
      - name: nginx
        image: nginx
        ports:
        - containerPort: 80
```
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)