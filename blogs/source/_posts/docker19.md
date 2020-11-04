---
title: deployment介绍和使用
date: 2020-11-02 09:16:51
categories: [docker]
tags: [docker]
---
## 什么是deployment
deployment是对pods和ReplicaSet的定义，定义了pods和ReplicaSet的定义和实现方式等。
如下为deployment的定义
<!--more-->
``` yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.12.2
        ports:
        - containerPort: 80
```
metadata 指明了服务名为nginx-deployment, 标签为nginx, 
spec指定了pod的副本为3个，每个pod容器镜像为ngix:1.12.2, 容器端暴漏的端口为80
接下来我们启动deployment
``` cmd
kubectl create -f deployment_nginx.yml 
```
会显示："nginx-deployment deployment has been created"
我们执行
``` cmd
kubectl get deployment 
```
查看deployment状态
``` cmd
NAME               DESIRED   CURRENT   UP-TO-DATE   AVAILABLE  AGE  
nginx-deployment      3          3        3            3        9s
```
可以看到deployment启动了三个pod，并且三个pod都是可用的。
``` cmd
kubectl get rs
```
可以看到ReplicaSet的状态为启动了3个pod，都是就绪状态
接下来可以查看下pod
``` cmd
kubectl get pods
```
显示deployment详细信息
``` cmd
kubectl get deployment -o wide
```
我们也可以更新deployment的image
``` cmd
kubectl set image deployment nginx-deployment nginx=nginx:1.1.13
```
我们可以回滚deployment版本
``` cmd
kubectl rollout undo deployment nginx-deployment 
```
查看deployment的历史信息
``` cmd
kubectl rollout history deployment nginx-deployment 
```
将deployment服务暴露出去
``` cmd
kubectl expose deployment nginx-deployment --type=NodePort
```
终端会提示服务已经暴露出去
``` cmd
service nginx-deployment  exposed
```
我们接下来查看下service信息
``` cmd
kubectl get svc
```
会显示服务映射的端口和地址
## 安装kubeadm
基于ubuntu配置k8s环境
``` cmd
hostnamectl set-hostname k8s-master
```
设置好后可以查看下我们的配置
``` cmd
tail /etc/hosts
```
查看防火墙状态
``` cmd
sudo apt-get install ufw
```
关闭临时分区
``` cmd
swapoff -a
```
更新https
``` cmd
apt-get update && apt-get install -y apt-transport-https
```
获取gpg
``` cmd
curl -fsSL https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | apt-key add - 
```
新增源
``` cmd
add-apt-repository "deb [arch=amd64] https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main"
```
更新apt
``` cmd
apt-get update
```
查看1.15最新版本
``` cmd
apt-cache madison kubelet kubectl kubeadm |grep '1.15.4-00'         //查看1.15的最新版本
```
安装指定版本的工具
``` cmd
apt install -y kubelet=1.15.4-00 kubectl=1.15.4-00 kubeadm=1.15.4-00        //安装指定的版本
```
kubelet禁用swap

``` cmd
tee /etc/default/kubelet <<-'EOF'
KUBELET_EXTRA_ARGS="--fail-swap-on=false"
EOF

systemctl daemon-reload && systemctl restart kubelet
```
初始化k8s
``` cmd
kubeadm init \
  --kubernetes-version=v1.15.4 \
  --image-repository registry.aliyuncs.com/google_containers \
  --pod-network-cidr=10.24.0.0/16 \
  --ignore-preflight-errors=Swap
```
在当前账户下执行，kubectl配置调用
``` cmd
mkdir -p $HOME/.kube
cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
chown $(id -u):$(id -g) $HOME/.kube/config
```
使用fannel的overlay网络实现多节点pod通信
``` cmd
kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
```
查看pods信息
``` cmd
kubectl get pods -A
```
配置dashboard
``` cmd
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.0.0-beta4/aio/deploy/recommended.yaml
```
配置后查看pod信息
``` cmd
get pods -A
```
查看namespaces信息
``` cmd
kubectl get namespaces
```
可以查看所有的namespaces信息
设置好网络模式后，接下来查看下apiserver暴露的地址
``` cmd
kubectl cluster-info
```
显示如下
``` cmd
Kubernetes master is running at https://172.17.0.9:6443
Heapster is running at https://172.17.0.9:6443/api/v1/namespaces/kube-system/services/heapster/proxy
KubeDNS is running at https://172.17.0.9:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
monitoring-grafana is running at https://172.17.0.9:6443/api/v1/namespaces/kube-system/services/monitoring-grafana/proxy
monitoring-influxdb is running at https://172.17.0.9:6443/api/v1/namespaces/kube-system/services/monitoring-influxdb/proxy
```
如果外网访问，换成外网地址就行了。
我自己dashboard的访问地址: 
``` cmd
https://81.68.86.146:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
因为访问dashboard需要权限
1.创建服务账号
首先创建一个叫admin-user的服务账号，并放在kube-system名称空间下：
``` yaml
# admin-user.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
```
执行kubectl create命令：
``` cmd
kubectl create -f admin-user.yaml
```
2.绑定角色
默认情况下，kubeadm创建集群时已经创建了admin角色，我们直接绑定即可：
``` yaml
# admin-user-role-binding.yaml
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
```
执行kubectl create命令：
``` cmd
kubectl create -f  admin-user-role-binding.yaml
```
3.获取Token
现在我们需要找到新创建的用户的Token，以便用来登录dashboard：
``` cmd
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $
1}')
```
4 制作证书
k8s默认启动了证书验证，我们创建证书
``` cmd
# 生成client-certificate-data
grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
# 生成client-key-data
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
# 生成p12
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```
然后我们将kubecfg.p12 copy到windows双击安装证书即可。
然后chrome 打开地址：

``` cmd
https://81.68.86.146:6443/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```
单节点k8s,默认pod不被调度在master节点,需要设置去污点
``` cmd
kubectl taint nodes --all node-role.kubernetes.io/master-     //去污点，master节点可以被调度
```
输出如下
``` cmd
node/k8s-master untainted
```
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)







