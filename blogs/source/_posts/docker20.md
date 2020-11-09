---
title: k8s service和网络
date: 2020-11-04 11:55:12
categories: [docker]
tags: [docker]
---
## k8s的service
k8s创建service，然后外部可以访问service从而实现访问pod和容器。
service 主要有三种类型，一种叫ClusterIP， 一种叫NodePort类型，一种叫外部的LoadBalancer
ClusterIP只能是cluster集群内部访问，NodePort可以支持外部访问。
kubectl expose 可以导出一个service
我们查看下pod信息
<!--more-->
``` cmd
kubectl  get pods -o wide -n default
```
显示如下
``` cmd
NAME                                READY   STATUS    RESTARTS   AGE    IP           NODE       NOMINATED NODE   READINESS GATES
busybox                             1/1     Running   0          14m    10.24.0.27   zackhost   <none>           <none>
nginx                               1/1     Running   0          109m   10.24.0.20   zackhost   <none>           <none>
```
然后我们导出busybox和nginx服务
``` cmd
kubectl expose pods nginx
```
然后我们可以通过命令查看service
``` cmd
kubectl get svc
```
会显示如下
``` cmd
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP   10.96.0.1       <none>        443/TCP   25h
nginx        ClusterIP   10.99.158.252   <none>        80/TCP    4m56s
```
可以看到service的clusterIP
我们在k8s任何一个节点都可以访问这个地址
``` cmd
curl 10.99.158.252:80
```
会输出nginx的页面信息
另外，当我们使用kubectl命令时可以设置自动补全功能
``` cmd
source <(kubectl completion bash)
```
设置后kubectl get 然后按下tab就可以自动补全了。

## 利用deployment启动python网站服务
先实现该pod文件
``` yml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: service-test
spec:
  replicas: 2
  selector:
    matchLabels:
      app: service_test_pod
  template:
    metadata:
      labels:
        app: service_test_pod
    spec:
      containers:
      - name: simple-http
        image: python:2.7
        imagePullPolicy: IfNotPresent
        command: ["/bin/bash"]
        args: ["-c", "echo \"<p>Hello from $(hostname)</p>\" > index.html; python -m SimpleHTTPServer 8080"]
        ports:
        - name: http
          containerPort: 8080
```
用命令启动deployment
``` cmd
kubectl create -f deployment_python_http.yml
```
将deployment导出为service
``` cmd
kubectl expose deployment service-test
```
然后我们查看service信息
``` cmd
kubectl get svc
```
显示结果
``` cmd
NAME           TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)    AGE
service-test   ClusterIP   10.108.183.252   <none>        8080/TCP   8s
```
我的k8s是单节点的，如果有其他节点，可以在任意节点执行curl命令指定地址和端口
``` cmd
curl 10.108.183.252:8080
```
显示
``` cmd
<p>Hello from service-test-85b6644b4d-6khz8</p>
```
deployment支持热更新
``` cmd
kubectl edit deployment service-test
```
之后会显示deployment的yml信息，修改后保存，之后service-test就自动热更新了。无需重启服务。
## NodePort 网络
我们先实现一个nginx_pod.yml文件，定义Pod
``` yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx
    ports:
    - name: nginx-port
      containerPort: 80
```
定义了一个名字为nginx-pod的pod，labels的key为app，value为nginx。然后定义了容器的名字为nginx-container，镜像为nginx， 端口为80
我们启动这个pod
``` cmd
kubectl create -f nginx_pod.yml
```
然后查看pod
``` cmd
kubectl get pods -o wide
```
接下来将pod导出service,选择类型NodePort
``` cmd
kubectl expose pods nginx-pod --type=NodePort
```
查看service信息
```cmd
kubectl get svc
```
可以看到nginx-pod这个服务的类型为NodePort
``` cmd
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.96.0.1        <none>        443/TCP        2d2h
nginx-pod    NodePort    10.104.141.197   <none>        80:31427/TCP   4m27s
```
我们通过任意一个节点的ip加上端口号31427即可访问nginx-pod的服务。查看节点信息
``` cmd
kubectl get node -o wide
```
可以看到节点信息
``` cmd
NAME       STATUS   ROLES    AGE    VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION      CONTAINER-RUNTIME
zackhost   Ready    master   2d3h   v1.15.4   172.17.0.9    <none>        Ubuntu 18.04.5 LTS   4.15.0-88-generic   docker://19.3.6
```
172.17.0.9为内网IP，我通过外网ip和端口31427就可以访问。
![1.png](1.png)
## 通过yml文件创建svc
我们先将之前创建的nginx的service删除
``` cmd
kubectl delete svc 
```
然后我们实现一个nginx的service的yml文件
``` yml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  ports:
  - port: 32333
    nodePort: 32334
    targetPort: nginx-port
    protocol: TCP
  selector:
    app: nginx
  type: NodePort
```
targetPort指定的端口名字为nginx-port， selector指定选择哪个pod，type指定service的类型
nodePort和port分别是service的端口和映射的端口
启动service
``` cmd
kubectl create -f service_nginx.yml
```
启动服务后，可以通过节点地址和端口访问该nginx服务。
``` cmd
kubectl get svc
```
显示服务启动成功
``` cmd
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE
kubernetes      ClusterIP   10.96.0.1        <none>        443/TCP           2d3h
nginx-service   NodePort    10.100.241.247   <none>        32333:32334/TCP   65s
```
接着访问指定网址和端口就能查看nginx服务了。
![2.png](2.png)

## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)