---
title: golang 使用etcd
date: 2019-08-29 16:40:47
categories: 技术开发
tags: [golang]
---
## etcd 安装和配置
下载etcd release版本：[https://github.com/coreos/etcd/releases/](https://github.com/coreos/etcd/releases/)
我的是windows版本，下载win64的就行了，下载后进入文件夹，建立几个配置文件启动就行了，可以写yml文件，我是windows的就做成了bat命令。
一般是一个节点对应一个配置，一个节点的配置如下
yml文件版本
``` yml
name: etcd-1
data-dir: .\data\etcd01 
advertise-client-urls: http://127.0.0.1:2379 
listen-client-urls: http://127.0.0.1:2379 
listen-peer-urls: http://127.0.0.1:2380 
initial-advertise-peer-urls: http://127.0.0.1:2380 
initial-cluster-token: etcd-cluster-1 
initial-cluster: etcd01=http://127.0.0.1:2380,etcd02=http://127.0.0.1:2381,etcd03=http://127.0.0.1:2382 ^
initial-cluster-state: new
```
<!--more-->
各参数含义:
``` cmd
etcd  命令含义
`--name` 
etcd集群中的节点名，这里可以随意，可区分且不重复就行
`--listen-peer-urls`
监听的用于节点之间通信的url，可监听多个，集群内部将通过这些url进行数据交互(如选举，数据同步等) 
 `--initial-advertise-peer-urls`
建议用于节点之间通信的url，节点间将以该值进行通信。
`--listen-client-urls`
监听的用于客户端通信的url,同样可以监听多个。
`--advertise-client-urls`
建议使用的客户端通信url,该值用于etcd代理或etcd成员与etcd节点通信。
`--initial-cluster-token etcd-cluster-1`
节点的token值，设置该值后集群将生成唯一id,并为每个节点也生成唯一id,当使用相同配置文件再启动一个集群时，只要该token值不一样，etcd集群就不会相互影响。
-`-initial-cluster`
也就是集群中所有的initial-advertise-peer-urls 的合集
`--initial-cluster-state new`
新建集群的标志，初始化状态使用 new，建立之后改此值为 existing
```
当然我的是windows版本，那么我就写bat文件就行了，我建立了三个bat文件对应三个节点。
start1.bat
``` bat
.\etcd.exe --name etcd01 ^
--data-dir .\data\etcd01 ^
--advertise-client-urls http://127.0.0.1:2379 ^
--listen-client-urls http://127.0.0.1:2379 ^
--listen-peer-urls http://127.0.0.1:2380 ^
--initial-advertise-peer-urls http://127.0.0.1:2380 ^
--initial-cluster-token etcd-cluster-1 ^
--initial-cluster etcd01=http://127.0.0.1:2380,etcd02=http://127.0.0.1:2381,etcd03=http://127.0.0.1:2382 ^
--initial-cluster-state new

pause
```

start2.bat
``` bat
.\etcd.exe --name etcd02 ^
--data-dir .\data\etcd02 ^
--advertise-client-urls http://127.0.0.1:3379 ^
--listen-client-urls http://127.0.0.1:3379 ^
--listen-peer-urls http://127.0.0.1:2381 ^
--initial-advertise-peer-urls http://127.0.0.1:2381 ^
--initial-cluster-token etcd-cluster-1 ^
--initial-cluster etcd01=http://127.0.0.1:2380,etcd02=http://127.0.0.1:2381,etcd03=http://127.0.0.1:2382 ^
--initial-cluster-state new
pause
```

start3.bat
``` bat
.\etcd.exe --name etcd03 ^
--data-dir .\data\etcd03 ^
--advertise-client-urls http://127.0.0.1:4379 ^
--listen-client-urls http://127.0.0.1:4379 ^
--listen-peer-urls http://127.0.0.1:2382 ^
--initial-advertise-peer-urls http://127.0.0.1:2382 ^
--initial-cluster-token etcd-cluster-1 ^
--initial-cluster etcd01=http://127.0.0.1:2380,etcd02=http://127.0.0.1:2381,etcd03=http://127.0.0.1:2382 ^
--initial-cluster-state new
pause
```
这三个bat文件放在之前etcd解压的文件夹里，我的目录结构如下
![1.jpg](1.jpg)
接下来分别启动弄三个bat文件，然后在终端输入
./etcdctl member list 
查看节点和集群信息
![2.jpg](2.jpg)
如果是yml文件写的配置，可以通过
./etcd --config-file=/etc/etcd/conf.yml启动
/etc/etcd/conf.yml是yml文件所在路径。

目前节点和集群启动完毕。
## etcd 常用命令
查询集群信息
![2.jpg](2.jpg)
查看集群状态
![3.jpg](3.jpg)
存储数据
![4.jpg](4.jpg)
读取数据
![5.jpg](5.jpg)
删除数据
![6.jpg](6.jpg)
etcd api分为2.0版本和3.0版本
使用3.0版本api需要在命令前加ETCDCTL_API=3
![7.jpg](7.jpg)
## golang 访问etcd集群
golang 安装etcd包
``` cmd
go get go.etcd.io/etcd/clientv3
```
大部分会失败，去github克隆吧
``` cmd
git clone https://github.com/etcd-io/etcd
```
接下来编码
``` golang
package main

import (
	"context"
	"fmt"
	"time"

	"go.etcd.io/etcd/clientv3"
)

const (
	EtcdKey string = "name"
	EtcdVal string = "zack"
)

func main() {
    //建立连接
	cli, err := clientv3.New(clientv3.Config{
		Endpoints:   []string{"localhost:2379", "localhost:3379", "localhost:4379"},
		DialTimeout: 5 * time.Second,
	})
	if err != nil {
		fmt.Println("connect failed, err:", err)
		return
	}
	fmt.Println("connect succ")
	defer cli.Close()
    //存储数据
	ctx, cancel := context.WithTimeout(context.Background(), time.Second)
	_, err = cli.Put(ctx, EtcdKey, string(EtcdVal))
	cancel()
	if err != nil {
		fmt.Println("put failed, err:", err)
		return
	}
    //读取数据
	ctx, cancel = context.WithTimeout(context.Background(), time.Second)
	resp, err := cli.Get(ctx, EtcdKey)
	cancel()
	if err != nil {
		fmt.Println("get failed, err:", err)
		return
	}
	for _, ev := range resp.Kvs {
		fmt.Printf("%s : %s\n", ev.Key, ev.Value)
	}

}
```
谢谢关注我的公众号
![wxgzh.jpg](wxgzh.jpg)




