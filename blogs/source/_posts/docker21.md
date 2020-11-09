---
title: k8s知识补充和汇总
date: 2020-11-06 10:50:58
tags: [docker]
categories: [docker]
---
今天补充下k8s命令和基础，查缺补漏。
## kubectl命令
kubectl 是k8s操作的基本命令
``` cmd
kubectl get nodes 获取集群运行的节点信息
kubectl config get-contexts 获取集群上下文信息
kubectl get pods 获取集群的pod信息
kubectl get svc 获取服务信息
kubectl get namespace 获取namespace信息
可以通过-n 指定namespace
kubectl get svc -n kube-system
也可以查看当前的上下文信息
kubectl config current-context
可以设置上下文
kubectl config set-context kubeadm
使用某个上下文
kubectl config use-context kubeadm
查看节点信息
kubectl describe node 节点id
查看node扩充信息
kubectl get node -o wide
```
<!--more-->
## 节点和标签
可以通过Kubectl get 指定节点得名字获得指定节点得信息
``` cmd
kubectl get node master-node
```
我们查看节点详细信息可以指明类型
``` cmd
kubectl get node -o yaml
kubectl get node -o json
```
查看节点及其labels
``` cmd
kubectl get node --show-labels
```
会显示所有节点，及其所有labels
``` cmd
NAME       STATUS   ROLES    AGE     VERSION   LABELS
zackhost   Ready    master   5d18h   v1.15.4   beta.kubernetes.io/arch=amd64,
beta.kubernetes.io/os=linux,kubernetes.io/arch=amd64,
kubernetes.io/hostname=zackhbernetes.io/master=
```
我们可以给指定的节点打标签
``` cmd
kubectl label node zackhost env=test
```
查看label
``` cmd
kubectl get node --show-labels
```
可以看到env=test的label了
``` cmd
NAME       STATUS   ROLES    AGE     VERSION   LABELS
zackhost   Ready    master   5d19h   v1.15.4   beta.kubernetes.io/arch=amd64,beta.kubernetes.io/os=linux,env=test,  kubernetes.io/arch=amd64,kubernetes.io/hostname=zackhost,kubernetes.io/os=linux,node-role.kubernetes.io/master=
```
删除label
``` cmd
kubectl label node zackhost env-
```
删除时将对应的key值env后边写上-就是删除key值为env的label
给节点设置角色worker
``` cmd
kubectl label node k8s-node1 node-role.kubernetes.io/worker=
```
## pod总结
pod是一个或者一组应用容器，他们分享资源(volume等)
pod内容器分享相同的命名空间，如ip地址等
Pod是k8s最小的调度单位
Pod的定义
``` cmd
apiVersion: v1
kind: Pod
metadata:
  name: nginx-busybox
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
  - name: busybox
    image: busybox
    command: ["/bin/sh"]
    args: ["-c", "while true; do echo hello; sleep 10;done"]
```
然后我们基于上面的nginx_busybox.yml文件创建pod
``` cmd
kubectl create -f nginx_busybox.yml 
```
显示
``` cmd
pod "nginx-busybox" created
```
然后我们查看pod列表
``` cmd
kubectl get pods
```
会显示pod列表
``` cmd
NAME                                READY   STATUS    RESTARTS   AGE
nginx-busybox                       2/2     Running   0          76s
```
我们通过describe查看一个pod的详细信息
``` cmd
kubectl describe pod nginx-busybox
```
也可以
``` cmd
kubectl get pods nginx-busybox -o wide
```
我们可以通过pod进入到容器内部
``` cmd
kubectl exec nginx-busybox -it sh
```
会默认进入第一个容器中
``` cmd
Defaulting container name to nginx.
Use 'kubectl describe pod/nginx-busybox -n default' to see all of the containers in this pod.
```
我们可以根据yml文件中指定的pod，删除该pod
``` cmd
kubectl delete -f nginx_busybox.yml
```
如果我们想操作pod中某个容器，可以通过-C选项指定容器名字
``` cmd
kubectl exec nginx-busybox -c nginx date
kubectl exec nginx-busybox -c busybox date
kubectl exec nginx-busybox -c busybox hostname
kubectl exec nginx-busybox -c nginx hostname
```
## Namespace 隔离环境
多个团队使用k8s需要通过namespace进行环境隔离
查看机器上的namespace
``` cmd
kubectl get namespaces
```
显示如下
``` cmd
NAME                   STATUS   AGE
default                Active   5d21h
kube-node-lease        Active   5d21h
kube-public            Active   5d21h
kube-system            Active   5d21h
kubernetes-dashboard   Active   5d20h
```
查看kube-system命名空间下的pod
``` cmd
kubectl get pods --namespace kube-system
```
我们创建一个demo的namespace
``` cmd
kubectl create namespace demo
```
然后我们实现一个pod的yaml文件
``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: demo
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
```
我们指明了pod的namespace为demo，接下来创建这个pod
``` cmd
kubectl create -f nginx_demo.yml
```
我们可以通过指定demo命名空间查看pod
``` cmd
kubectl get pods --namespace demo
```
可以看到demo命名空间下存在我们刚创建的pod
``` cmd
NAME    READY   STATUS    RESTARTS   AGE
nginx   1/1     Running   0          14s
```
也可以列举出所有命名空间的pod
``` cmd
kubectl get pod --all-namespaces
```
可以看到列举出了所有命名空间下的pod
``` cmd
NAMESPACE              NAME                                        READY   STATUS             RESTARTS   
demo                   nginx                                       1/1     Running            0          
kube-system            coredns-bccdc95cf-mp6hs                     1/1     Running            0          
kubernetes-dashboard   kubernetes-dashboard-6bb65fcc49-dznpl       1/1     Running            0          
```
## context上下文
获取所有contexts
``` cmd
kubectl config get-contexts
```
显示context列表如下
``` cmd
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
```
我们可以设置一个context
``` cmd
kubectl config set-context demo --user=kubernetes-admin --cluster=kubernetes --namespace=demo
```
创建后我们查看下
``` cmd
kubectl config get-contexts
```
显示如下
``` cmd
CURRENT   NAME                          CLUSTER      AUTHINFO           NAMESPACE
          demo                          kubernetes   kubernetes-admin   demo
*         kubernetes-admin@kubernetes   kubernetes   kubernetes-admin   
```
user和cluster是怎么确定的呢？我们可以通过
``` cmd
kubectl config view 
```
可以看到user和cluster信息
删除context之前，最好先切换到其他context
``` cmd
kubectl config use-context kubernetes-admin@kubernetes
kubectl config delete-context demo
```
删除namespace
``` cmd
kubectl delete namespace demo
```
## deployment和controller
controller位于master节点，负责任务调度和节点启动停止等控制。
deployment是描述了一个期望的状态，而controller会根据实际状态和期望状态做对比，
会改变实际状态满足期望状态。
比如我们通过scheduler将pod放在第一个node上了，而第一个node宕机了。pod内的环境就会消失，任务也会终止。
所以我们尽可能的使用deployment，deployment会根据实际情况调整任务分配到其他节点。而controller就是会尝试
在其他节点重新创建pod。
我们先实现一个nginx_deployment.yml文件
``` cmd
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      # unlike pod-nginx.yaml, the name is not included in the meta data as a unique name is
      # generated from the deployment name
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
上述deployment文件定义了deployment名字为nginx-deployment，选择器选择匹配的pod的label为app:nginx，
replicas：2告诉deployment启动2个pods，template定义了pod的内容，pod的标签为app:nginx，
pod内部定义了容器的名字为nginx，容器的镜像选择nginx:1.7.9，暴露的端口号为80
我们启动这个deployment
``` cmd
kubectl create -f nginx_deployment.yml
```
查看deployment
``` cmd
kubectl get deployment
```
我们可以根据label查看具体的pod
``` cmd
kubectl get pods -l app=nginx
```
显示如下
``` cmd
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5754944d6c-br7nh   1/1     Running   0          19m
nginx-deployment-5754944d6c-jfhxz   1/1     Running   0          19m
```
接下来我们校验下deployment如何通过controller控制pod数量的，比如我们删除其中的一个pod
``` cmd
kubectl delete pod nginx-deployment-5754944d6c-br7nh
```
然后我们立即查看现在pod的信息
``` cmd
kubectl get pods -l app=nginx
```
输出如下
``` cmd
NAME                                READY   STATUS        RESTARTS   AGE
nginx-deployment-5754944d6c-br7nh   0/1     Terminating   0          60s
nginx-deployment-5754944d6c-cb79g   1/1     Running       0          3s
nginx-deployment-5754944d6c-jfhxz   1/1     Running       0          23m
```
看到nginx-deployment-5754944d6c-br7nh的pod正在终止
过了几秒我们再次查看
``` cmd
kubectl get pods -l app=nginx
```
可以看到目前稳定运行的两个pod
``` cmd
NAME                                READY   STATUS    RESTARTS   AGE
nginx-deployment-5754944d6c-cb79g   1/1     Running   0          2m29s
nginx-deployment-5754944d6c-jfhxz   1/1     Running   0          25m
```
我们可以更新deployment中容器的镜像版本，在更新前我们看下版本
``` cmd
kubectl get deployment nginx-deployment -o wide
```
显示版本为nginx:1.7.9
``` cmd
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES        SELECTOR
nginx-deployment   2/2     2            2           28m   nginx        nginx:1.7.9   app=nginx
```
我们再实现一个deployment,他的nginx版本为nginx:1.8
``` yml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8 
        ports:
        - containerPort: 80
```
我们更新deployment，需要使用apply命令
``` cmd
kubectl apply -f nginx_deployment_update.yml
```
更新后我们查看deployment
``` cmd
kubectl get deployment nginx-deployment -o wide
```
显示版本变为nginx:1.8
``` cmd
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES      SELECTOR
nginx-deployment   2/2     1            2           35m   nginx        nginx:1.8   app=nginx
```
我们也可以扩充pod数量，我们在实现一个nginx_deployment_scale.yml文件
``` yml
apiVersion: apps/v1 
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 4 # Update the replicas from 2 to 4
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.8
        ports:
        - containerPort: 80
```
同样，我们执行更新命令
``` cmd
kubectl apply -f nginx_deployment_scale.yml 
```
查看deployment信息
``` cmd
kubectl get deployment -o wide
```
显示版本
``` cmd
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES      SELECTOR
nginx-deployment   4/4     4            4           41m   nginx        nginx:1.8   app=nginx
```
获取pod信息
``` 
kubectl get pod -l app=nginx -o wide
```
显示四个pod正在运行
``` cmd
NAME                                READY   STATUS    RESTARTS   AGE     IP           NODE         
nginx-deployment-6f655f5d99-4ktlw   1/1     Running   0          8m53s   10.24.0.38   zackhost   
nginx-deployment-6f655f5d99-4mjqn   1/1     Running   0          3m39s   10.24.0.40   zackhost   
nginx-deployment-6f655f5d99-tcmb2   1/1     Running   0          9m19s   10.24.0.37   zackhost   
nginx-deployment-6f655f5d99-vvfrr   1/1     Running   0          3m39s   10.24.0.39   zackhost   
```
除了可以采用上述yml文件apply方式更新deployment，还可以在线编辑deployment
``` cmd
kubectl edit deployment nginx-deployment
```
会看到打开了一个vim，vim内容就是deployment的yml文件
![1.png](1.png)
也可以通过这个vim修改deployment信息，比如我们将replicas设置为3，然后ESC, :wq保存退出。
然后我们查看deployment信息
``` cmd
NAME               READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES      SELECTOR
nginx-deployment   3/3     3            3           62m   nginx        nginx:1.8   app=nginx
```
也可以通过scale命令扩充pod数量
``` cmd
kubectl scale --current-replicas=3 --replicas=4 deployment/nginx-deployment
```
更新image版本，也可以通过yml文件apply，也可以edit方式，也可以用如下命令
``` cmd
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1
```
## ReplicaSet应用
replicaset负责处理deployment的scale操作
我们将之前生成的deployment全部删除，然后实现一个新的nginx_deployment.yml
``` yml
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2 # tells deployment to run 2 pods matching the template
  template: # create pods using pod definition in this template
    metadata:
      # unlike pod-nginx.yaml, the name is not included in the meta data as a unique name is
      # generated from the deployment name
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
        ports:
        - containerPort: 80
```
生成deployment
``` cmd
kubectl apply -f nginx_deployment.yml 
```
查看生成得deployment
``` cmd
kubectl get deployment -o wide
```
显示如下
``` cmd
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE    CONTAINERS   IMAGES        SELECTOR
nginx-deployment-test   4/4     4            4           3m2s   nginx        nginx:1.7.9   app=nginx
```
我们可以查看replicaset信息
``` cmd
kubectl get replicaset
```
显示如下
``` cmd
NAME                               DESIRED   CURRENT   READY   AGE
nginx-deployment-test-5754944d6c   4         4         4       5m46s
```
我们用scale命令将nginx的pod从四个变为六个
``` cmd
kubectl scale --current-replicas=4 --replicas=6 deployment/nginx-deployment-test
```
接下来我们通过一下几个命令都能查看更新后的pod和deployment信息
``` cmd
kubectl get pods -l app=nginx
kubectl get deployment -o wide
```
我们可以通过describe获取deployment的描述信息
``` cmd
kubectl describe deployment nginx-deployment-test
```
会输出scale信息
``` cmd
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  16m    deployment-controller  Scaled up replica set nginx-deployment-test-5754944d6c to 4
  Normal  ScalingReplicaSet  6m45s  deployment-controller  Scaled up replica set nginx-deployment-test-5754944d6c to 6
```
如果我们更新容器镜像
``` cmd
kubectl set image deployment/nginx-deployment-test nginx=nginx:1.9.1
```
这时我们查看描述信息
``` cmd
kubectl describe deployment nginx-deployment-test
```
可以看到更新政策为25%用来更新，25%用来提供服务的模式,而且会开辟副本做更新，最后统一合并。
``` cmd
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Events:
  Type    Reason             Age                    From                   Message
  ----    ------             ----                   ----                   -------
  Normal  ScalingReplicaSet  36m                    deployment-controller  Scaled up replica set nginx-deployment-test-5754944d6c to 4
  Normal  ScalingReplicaSet  26m                    deployment-controller  Scaled up replica set nginx-deployment-test-5754944d6c to 6
  Normal  ScalingReplicaSet  2m55s                  deployment-controller  Scaled up replica set nginx-deployment-test-7448597cd5 to 2
  Normal  ScalingReplicaSet  2m55s                  deployment-controller  Scaled down replica set nginx-deployment-test-5754944d6c to 5
  Normal  ScalingReplicaSet  2m55s                  deployment-controller  Scaled up replica set nginx-deployment-test-7448597cd5 to 3
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled down replica set nginx-deployment-test-5754944d6c to 4
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled up replica set nginx-deployment-test-7448597cd5 to 4
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled down replica set nginx-deployment-test-5754944d6c to 3
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled up replica set nginx-deployment-test-7448597cd5 to 5
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled down replica set nginx-deployment-test-5754944d6c to 2
  Normal  ScalingReplicaSet  2m52s                  deployment-controller  Scaled up replica set nginx-deployment-test-7448597cd5 to 6
  Normal  ScalingReplicaSet  2m48s (x2 over 2m50s)  deployment-controller  (combined from similar events): Scaled down replica set nginx-deployment-test-5754944d6c to 0
```
我们可以通过如下命令查看更新版本的历史信息
``` cmd
kubectl rollout history deployment nginx-deployment-test 
```
显示历史信息
``` cmd
deployment.extensions/nginx-deployment-test 
REVISION  CHANGE-CAUSE
1         <none>
2         <none>
```
可以看到有两次版本信息,可以指明指定版本信息
``` cmd
kubectl rollout history deployment nginx-deployment-test --revision 1
```
显示的是版本1的信息
``` cmd
deployment.extensions/nginx-deployment-test with revision #1
Pod Template:
  Labels:	app=nginx
	pod-template-hash=5754944d6c
  Containers:
   nginx:
    Image:	nginx:1.7.9
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```
我们输入--revision 2可以看到版本2的信息
``` cmd
kubectl rollout history deployment nginx-deployment-test --revision 2
```
版本2采用的镜像是1.9.1
``` cmd
deployment.extensions/nginx-deployment-test with revision #2
Pod Template:
  Labels:	app=nginx
	pod-template-hash=7448597cd5
  Containers:
   nginx:
    Image:	nginx:1.9.1
    Port:	80/TCP
    Host Port:	0/TCP
    Environment:	<none>
    Mounts:	<none>
  Volumes:	<none>
```
我们可以回滚到指定版本
``` cmd
kubectl rollout undo deployment nginx-deployment-test 
```
我们查看下deployment的信息
``` cmd
kubectl get deployment nginx-deployment-test  -o wide
显示如下
NAME                    READY   UP-TO-DATE   AVAILABLE   AGE   CONTAINERS   IMAGES        SELECTOR
nginx-deployment-test   6/6     6            6           52m   nginx        nginx:1.7.9   app=nginx
```
这时我们看下回滚历史
``` cmd
kubectl rollout history deployment nginx-deployment-test
显示如下
deployment.extensions/nginx-deployment-test 
REVISION  CHANGE-CAUSE
2         <none>
3         <none>
```
默认只能保存两个版本。我们可以通过指定版本号回退到指定版本
``` cmd
kubectl rollout undo deployment nginx-deployment-test --to-revision 2
deployment.extensions/nginx-deployment-test rolled back
```
## 总结
目前k8s的补充就到此为止，包括了kubectl命令，deployment的使用，pod，namespace，label等概念和操作。
以后会不断完善。感谢关注公众号
![wxgzh.jpg](wxgzh.jpg)














