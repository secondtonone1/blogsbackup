---
title: docker命令(二)
date: 2020-07-02 20:45:53
categories: [docker]
tags: [docker]
---
## 删除docker
sudo docker rm 容器id
如果容器正在运行，可以执行强制删除命令
sudo docker rm -f 容器id
<!--more-->
## 启动端口映射
可以将容器内的端口映射到宿主机上的某个端口，从而达到通过访问宿主机端口访问容器的目的
比如我们启动一个tomcat容器
docker run -it --name mytomcat -p 8888:8080 tomcat
然后可以看到tomcat镜像启动日志，我们exit退出，通过docker ps 查看tomcat镜像还在
![1.png](1.png)
此时我打开浏览器访问8888端口，可以看到端口和地址是可以访问的，但是资源文件不存在
![2.png](2.png)
我们用exec命令进入容器
docker exec -it ce0da6732832 /bin/bash
进入容器后我们将webapps删除，然后将webapps.dist内容复制为webapps
rm -rf ./webapps
cp -r ./webapps.dist ./webapps
这时我们重启容器
docker restart ce0da6732832
这时我们登录浏览器访问8888端口，可以看到tomcat信息了
![3.png](3.png)
## 将容器副本打包为新的镜像
可以将容器副本打包为一个新的镜像，
docker commit -a "作者名" -m "说明" 容器id   镜像名:版本号
我们将我们改造好的tomcat容器打造为新的镜像
docker commit -a "secondtonone1" -m "new mytomcat" ce0da6732832 secondtonone1/mytomcat:v1.0
屏幕会输出镜像id
此时我们查看镜像
docker images -a
![4.png](4.png)
可以看到有两个tomcat镜像，secondtonone1/mytomcat:v1.0的是我们新生成的
## 根据镜像启动新的docker
我们通过secondtonone1/mytomcat:v1.0
docker run -id --name mytomcat1.0 -P  secondtonone1/mytomcat:v1.0
通过查看容器运行，看到如下
![5.png](5.png)
运行了两个tomcat，其中tomcat1.0是新生成的，他被分配了随机端口32768
在浏览器输入地址和32768端口，可以看到新生成的tomcat docker已经运行了。
![6.png](6.png)
## 感谢关注公众号
今天的笔记就这些吧，感谢关注公众号
![wxgzh.jpg](wxgzh.jpg) 
