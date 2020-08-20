---
title: docker实战
date: 2020-08-10 09:31:06
categories: [docker]
tags: [docker]
---
## 安装mysql
今天试试mysql实战安装myql
先pull镜像
``` cmd
docker pull mysql:5.6
```
接着启动mysql镜像
``` cmd
docker run -p 12345:3306 --name mysql56 \
-v  /home/zack/dockerwork/mysql/conf:/etc/mysql/conf.d \
-v  /home/zack/dockerwork/mysql/logs:/logs \
-v  /home/zack/dockerwork/mysql/data:/var/lib/mysql \
-e  MYSQL_ROOT_PASSWORD=123456  \
-d  mysql:5.6
```
<!--more-->
我们查看下docker ps 列出所有docker进程
然后我们进入docker里
``` cmd
docker exec -it 615164d51197 /bin/bash
```
进入后我们使用myql登录
``` cmd
mysql -uroot -p
```
输入密码后进入mysql。之后就可以建立数据库和表了。

## 启动mongo容器
首先先下载mongo的安装包，然后解压放在自己设定的目录下
``` Dockerfile
FROM ansible/centos7-ansible:latest
RUN mkdir -p /data/mongodb/log/
RUN mkdir -p /data/mongodb/bin
RUN mkdir -p /data/mongodb/data
#RUN yum install libssl1.0.0 libssl-dev
ENV PATH /data/mongodb/bin:$PATH
ADD mongodb-linux-x86_64-rhel70-4.2.8 /data/mongodb
WORKDIR /data/mongodb/
EXPOSE 60000
#VOLUME ["/data/env/mongo/data/:/data/mongodb/data","/data/env/mongo/log/:/data/mongodb/log/"]
CMD ["/data/mongodb/bin/mongod","-f", "/data/mongodb/mongodb.conf"]
```
根据Dockerfile生成镜像
``` cmd
docker build -f ./Dockerfile -t mymongo .
```
启动docker
``` cmd
docker run  -p 54321:60000 --name submitmg -v /home/zack/dockerwork/mongodb_/data:/data/mongodb/data  -v /home/zack/dockerwork/mongodb_/log:/data/mongodb/log/  --privileged=true  -d  mymongo
```
进入docker容器
``` cmd
docker exec -it cdebf8e13939 /bin/bash
```
登录数据库
``` cmd
./mongo --port 60000
```
创建数据库 submit
``` cmd
use submit
```
创建数据库表
``` cmd
db.createCollection("log_info")
```
文档结构如下
``` cmd
{
    "compareid":"12345",
    "ic":"23333HC",
    "phone":"18301152001",
    "url":"www.singlepo.com",
    "dir":0,
    "imageurl":"www.singlepo.com/wangqiang.jpg"
}
```
创建索引
``` cmd
db.log_info.createIndex({"ic":1,"compareid":-1})
```
插入数据测试
``` cmd
db.log_info.insert({
    "compareid":"12345",
    "ic":"23333HC",
    "phone":"18301152001",
    "url":"www.singlepo.com",
    "dir":0,
    "imageurl":"www.singlepo.com/wangqiang.jpg"
})
```
查询刚才插入的数据
``` cmd
db.log_info.find()
```
## 启动redis容器

1 拉取镜像
``` cmd
docker pull redis:3.2
```
2 用镜像启动容器
``` cmd
docker run -p 6679:6379 -v /home/zack/dockerwork/redis/data:/data 
-v /home/zack/dockerwork/redis/conf/redis.conf:/usr/local/etc/redis/redis.conf  
-d redis:3.2 redis-server  /usr/local/etc/redis/redis.conf --appendonly yes
```
3 配置redis
在/home/zack/dockerwork/redis/conf/redis.conf下创建redis.conf文件
``` cmd

port 6379

tcp-backlog 511

timeout 0

tcp-keepalive 0

loglevel notice

logfile ""

databases 16

save 900 1
save 300 10
save 60 10000

stop-writes-on-bgsave-error yes

rdbchecksum yes

dbfilename dump.rdb

dir ./

slave-serve-stale-data yes

slave-read-only yes

repl-diskless-sync no

repl-diskless-sync-delay 5

repl-disable-tcp-nodelay no

slave-priority 100

maxheap 51200000

heapdir ./

appendonly no

appendfilename "appendonly.aof"

appendfsync everysec

no-appendfsync-on-rewrite no

auto-aof-rewrite-percentage 100
auto-aof-rewrite-min-size 64mb

aof-load-truncated yes

lua-time-limit 5000

slowlog-log-slower-than 10000

slowlog-max-len 128

latency-monitor-threshold 0

notify-keyspace-events ""

hash-max-ziplist-entries 512
hash-max-ziplist-value 64

list-max-ziplist-entries 512
list-max-ziplist-value 64

set-max-intset-entries 512

zset-max-ziplist-entries 128
zset-max-ziplist-value 64

hll-sparse-max-bytes 3000

activerehashing yes

client-output-buffer-limit normal 0 0 0
client-output-buffer-limit slave 256mb 64mb 60
client-output-buffer-limit pubsub 32mb 8mb 60
hz 10
aof-rewrite-incremental-fsync yes
```
4 启动客户端
docker exec -it 139abd0bd512 redis-cli
进入命令模式后就可以set key value测试了。退出容器可以在redis/data路径里看到appendonly.aof文件里有命令
## 感谢关注公众号
今天的笔记就这些吧，感谢关注公众号
![wxgzh.jpg](wxgzh.jpg)







