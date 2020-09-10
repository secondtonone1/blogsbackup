---
title: docker网络(一)
date: 2020-09-10 10:53:48
categories: [docker]
tags: [docker]
---
## 构建两个busybox容器
构建两个busybox容器
``` cmd
docker run -d --name test1 busybox /bin/sh -c "while true; do sleep 3000; done"
docker run -d --name test2 busybox /bin/sh -c "while true; do sleep 3000; done"
```
然后我们分别执行ip a命令，看看各个容器的网络地址
<!--more-->
``` cmd
docker exec -it test1  ip a
```
可以看到test1的网络地址
``` cmd
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
806: eth0@if807: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:12:00:05 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.5/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
看看test2
``` cmd
docker exec -it test2  ip a
```
可以看到test2的网络地址
``` cmd
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
808: eth0@if809: <BROADCAST,MULTICAST,UP,LOWER_UP,M-DOWN> mtu 1500 qdisc noqueue 
    link/ether 02:42:ac:12:00:06 brd ff:ff:ff:ff:ff:ff
    inet 172.18.0.6/16 brd 172.18.255.255 scope global eth0
       valid_lft forever preferred_lft forever
```
我们通过test1 ping test2
``` cmd
docker exec -it test2  ping 172.18.0.5
```
可以看到ping成功了
``` cmd
64 bytes from 172.18.0.5: seq=0 ttl=64 time=0.092 ms
64 bytes from 172.18.0.5: seq=1 ttl=64 time=0.074 ms
64 bytes from 172.18.0.5: seq=2 ttl=64 time=0.073 ms
```
## linux 构建network namespace联通
本节在linux系统设置两个namespace连接，两个network namespace就好比是docker
这样方便我们了解网络连接的原理
查看本机net namespace
``` cmd
ip netns list 
```
添加network namespace 
``` cmd
sudo ip netns add network1
sudo ip netns add network2
```
查看network1 ip link 信息
``` cmd
sudo ip netns exec network1 ip link 
```
可以看到network1的link信息
``` cmd
1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
设置network1的link信息
``` cmd
sudo ip netns exec network1 ip link set dev lo up
```
可以看到lo信息不再时DOWN，而是UNKNOWN模式了
``` cmd
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
```
用同样的方法把network2的命名空间的模式也打开
``` cmd
sudo ip netns exec network1 ip link set dev lo up
```
通过veth技术将两个network连接起来
``` cmd
sudo ip link add veth-network1 type veth peer name veth-network2
```
此时执行
``` cmd
sudo ip link
```
可以看到link信息新增了两个veth，接下来将veth-network1接口添加到network1里
将veth-network2接口添加到network2里
``` cmd
sudo ip link set veth-network1 netns network1
sudo ip link set veth-network2 netns network2
```
接下来我们查看network1的link信息
``` cmd
sudo ip netns exec network1 ip link
```
可以看到network1的ip link信息
``` cmd
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
811: veth-network1@if810: <BROADCAST,MULTICAST> mtu 1500 qdisc noop state DOWN mode DEFAULT group default qlen 1000
    link/ether ca:dd:69:8a:59:18 brd ff:ff:ff:ff:ff:ff link-netnsid 1
```
接下来为两个namespace设置地址
``` cmd
sudo ip netns exec network1  ip addr add 192.168.1.1/24 dev veth-network1
sudo ip netns exec network2  ip addr add 192.168.1.2/24 dev veth-network2
```
然后将两个namespace的veth设置启动
``` cmd
sudo ip netns exec network1 ip link set dev veth-network1 up
sudo ip netns exec network2 ip link set dev veth-network2 up
```
这时候再查看ip信息
``` cmd
sudo ip netns exec network1 ip link
```
可以看到network1的veth端口up
``` cmd
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
811: veth-network1@if810: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP mode DEFAULT group default qlen 1000
    link/ether ca:dd:69:8a:59:18 brd ff:ff:ff:ff:ff:ff link-netnsid
```
然后查看两个网络的ip信息
``` cmd
sudo ip netns exec network1 ip a
```
可以看到network1的ip信息
``` cmd
811: veth-network1@if810: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc noqueue state UP group default qlen 1000
    link/ether ca:dd:69:8a:59:18 brd ff:ff:ff:ff:ff:ff link-netnsid 1
    inet 192.168.1.1/24 scope global veth-network1
       valid_lft forever preferred_lft forever
    inet6 fe80::c8dd:69ff:fe8a:5918/64 scope link 
       valid_lft forever preferred_lft forever
```
然后通过network2去ping包给network1
``` cmd
sudo ip netns exec network2 ping 192.168.1.1
```
可以看到这两个网络现在互通了。
``` cmd
64 bytes from 192.168.1.1: icmp_seq=1 ttl=64 time=0.036 ms
64 bytes from 192.168.1.1: icmp_seq=2 ttl=64 time=0.024 ms
64 bytes from 192.168.1.1: icmp_seq=3 ttl=64 time=0.046 ms
```
以上就是linux环境下通过network的namespace方式达到网络互联的。
## 个人公众号
![wxgzh.jpg](wxgzh.jpg)

