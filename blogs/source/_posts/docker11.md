---
title: docker网络(二)
date: 2020-09-15 15:28:09
categories: [docker]
tags: [docker]
---
## docker网络是如何和宿主机相通的
先用docker命令查看下我们的docker网络
``` cmd
docker network list
```
<!--more-->
可以看到网络列表
``` cmd
NETWORK ID          NAME                DRIVER              SCOPE
bd45b573efca        bridge              bridge              local
dffe767ef55b        com-sig             bridge              local
b58d4f1da54c        host                host                local
77d57a8a8f61        none                null                local
```
然后我们查看指定网络信息
``` cmd
docker inspect network bd45b573efca
```
可以看到test1容器的网络信息
``` cmd
"Containers": {
            "07f5643a681c19fb9d8907c62631478123e8e1bb520d7c579a8c231500147bbd": {
                "Name": "test1",
                "EndpointID": "e28c35eb738ed69124d26ab141e220a18016eaf11a6c703b932e012c8b09b20e",
                "MacAddress": "02:42:ac:12:00:04",
                "IPv4Address": "172.18.0.4/16",
                "IPv6Address": ""
            },
}
```
test1这个容器连接的是docker0这个桥。我们可以通过ip a查看网桥和接口
可以看到docker0信息
``` cmd
docker0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default 
    link/ether 02:42:54:68:91:f7 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.1/16 brd 172.18.255.255 scope global docker0
       valid_lft forever preferred_lft forever
    inet6 fe80::42:54ff:fe68:91f7/64 scope link 
       valid_lft forever preferred_lft forever
```
也可以看到veth的接口信息，test容器的网络其实是通过vetch的接口连接到docker0上的
``` cmd
sudo docker exec test1 ip a
```
可以看到test容器的veth接口，该接口连接的就是docker0网桥
接下来我们安装个工具
``` cmd
sudo apt-get install bridge-utils
brctl show 
```
这样看到docker0连接的接口veth077a502,而该接口就是和容器连接的接口
``` cmd
bridge name	  bridge id		    STP        enabled	interfaces
docker0		  8000.          0242546891f7	no		veth077a502
```
这样，容器和宿主机就连接了。
## 多个容器如何实现网络互联
每个容器通过自己的veth连接docker0网桥，从而达到网络互联的目的
![1.png](1.png)
而bridge0可以通过nat映射访问外网
![2.png](2.png)
## 多个容器之间互联
可以通过link进行连接，但是link是单向的，比如
``` cmd
docker run -d --name test2 --link test1 busybox /bin/sh -c "while true; do sleep 3000; done"
```
test2是单方向连接test1，也就是从test2中可以访问test1，而test1是无法访问test2的
可以通过构建network实现两个容器互相访问
``` cmd
docker network create -d bridge com-sig
```
输入
``` cmd
docker network ls
```
可以看到docker网络
``` cmd
NETWORK ID          NAME                DRIVER              SCOPE
bd45b573efca        bridge              bridge              local
```
启动新的容器指定network为com-sig
``` cmd
docker run -d --name test2 --network com-sig busybox /bin/sh -c "while true; do sleep 3000; done"
```
对于已经存在的容器，可以通过connect命令将两个容器连接到一个网络
``` cmd
docker network connect com-sig test1
```
上述命令将test1连接到com-sig网络里，这样test1和test2就可以通信了。
## 感谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)




